# Compartments and Identity

## Why Compartments Matter

Compartments are OCI's primary isolation boundary. They are not AWS accounts, not Azure resource groups, and not GCP projects — they are a hierarchical namespace inside a single tenancy that carries authorization, quotas, and budgets simultaneously.

Consequences for Terraform:

- Nearly every resource takes a `compartment_id`, and it is not inferred from context
- Moving a resource between compartments is a real API operation, not a tag change
- IAM policies are attached to a compartment and cascade downward
- Quotas and budgets are set per compartment
- Deleting a compartment requires it to be empty, and deletion is asynchronous and slow

## Compartment Hierarchy Design

A workable default separates environment first, then function:

```
tenancy (root)
├── shared
│   ├── network          # hub VCN, DRG, on-prem connectivity
│   ├── security         # vaults, keys, logging, Cloud Guard
│   └── tooling          # CI runners, artifact storage
├── prod
│   ├── prod-app
│   ├── prod-data
│   └── prod-network
├── stage
│   ├── stage-app
│   └── stage-data
└── dev
    └── dev-sandbox
```

Rules that hold up in practice:

- Never create workload resources in the root compartment
- Keep the tree shallow; depth beyond four levels makes policies hard to reason about
- One compartment per environment per workload is the smallest unit worth isolating
- Put shared networking in its own compartment so network admins get scoped access
- Sandbox compartments get quotas and budgets, not trust

## Creating Compartments

Compartments are identity resources and must target the home region.

```hcl
resource "oci_identity_compartment" "env" {
  provider = oci.home

  compartment_id = var.parent_compartment_ocid
  name           = var.environment
  description    = "Top-level compartment for ${var.environment}"

  # Terraform can delete compartments, but only when empty.
  enable_delete = false

  freeform_tags = {
    managed_by  = "terraform"
    environment = var.environment
  }
}

resource "oci_identity_compartment" "workload" {
  provider = oci.home
  for_each = toset(var.workloads)

  compartment_id = oci_identity_compartment.env.id
  name           = "${var.environment}-${each.key}"
  description    = "${each.key} workload in ${var.environment}"
  enable_delete  = false
}
```

`enable_delete = false` is the safe default. With it set to `true`, a `terraform destroy` will attempt compartment deletion, which can take hours and leaves the compartment in a `DELETING` state that blocks name reuse.

## Referencing Compartments

Look compartments up rather than hardcoding OCIDs across environments:

```hcl
data "oci_identity_compartments" "app" {
  compartment_id            = var.tenancy_ocid
  name                      = "prod-app"
  access_level              = "ACCESSIBLE"
  compartment_id_in_subtree = true
  state                     = "ACTIVE"
}

locals {
  app_compartment_id = one(data.oci_identity_compartments.app.compartments[*].id)
}
```

`compartment_id_in_subtree = true` searches the whole hierarchy; without it the lookup only sees direct children.

## OCIDs

An OCID is an opaque identifier with a documented shape:

```
ocid1.<resource-type>.<realm>.<region>.<unique-id>

ocid1.compartment.oc1..aaaaaaaa...        # no region — global
ocid1.instance.oc1.phx.anyhq...           # region-scoped
```

Practical guidance:

- Treat OCIDs as opaque. Do not parse them to extract a region or type.
- Never hardcode an OCID that differs per tenancy — resolve it via a data source or pass it as a variable.
- Tenancy and compartment OCIDs are legitimate root-module variables; nested resource OCIDs usually are not.
- OCIDs are safe to log and appear in plan output. They are identifiers, not secrets.

Validate the shape at the module boundary so a mistyped value fails fast:

```hcl
variable "compartment_ocid" {
  description = "OCID of the compartment to deploy into"
  type        = string

  validation {
    condition     = can(regex("^ocid1\\.(compartment|tenancy)\\.", var.compartment_ocid))
    error_message = "compartment_ocid must be a compartment or tenancy OCID."
  }
}
```

## IAM Policy Language

OCI policies are human-readable statements, not JSON documents. The grammar is:

```
Allow <subject> to <verb> <resource-type> in <location> [where <condition>]
```

Verbs are cumulative tiers: `inspect` < `read` < `use` < `manage`.

```hcl
resource "oci_identity_policy" "network_admins" {
  provider = oci.home

  compartment_id = var.tenancy_ocid
  name           = "network-admins"
  description    = "Network team manages shared networking"

  statements = [
    "Allow group NetworkAdmins to manage virtual-network-family in compartment shared:network",
    "Allow group NetworkAdmins to manage load-balancers in compartment shared:network",
    "Allow group NetworkAdmins to read all-resources in tenancy",
  ]
}
```

Compartment paths use `:` as the separator for nested compartments (`shared:network` means the `network` compartment inside `shared`).

### Conditions

```hcl
statements = [
  # Restrict by tag
  "Allow group Developers to manage instance-family in compartment dev where target.resource.tag.Operations.Environment = 'dev'",

  # Restrict by request source
  "Allow group Contractors to read object-family in compartment data where request.networkSource.name = 'corp-network'",

  # Restrict to specific operations
  "Allow group Auditors to inspect all-resources in tenancy where request.operation = 'ListBuckets'",
]
```

### Least Privilege Patterns

```hcl
resource "oci_identity_policy" "app_team" {
  provider = oci.home

  compartment_id = oci_identity_compartment.env.id
  name           = "${var.environment}-app-team"
  description    = "Application team scoped to its own compartment"

  statements = [
    # Scoped to one compartment, specific families only
    "Allow group ${var.environment}-AppTeam to manage instance-family in compartment ${oci_identity_compartment.workload["app"].name}",
    "Allow group ${var.environment}-AppTeam to use subnets in compartment shared:network",
    "Allow group ${var.environment}-AppTeam to use vnics in compartment shared:network",
    "Allow group ${var.environment}-AppTeam to read repos in tenancy",
  ]
}
```

Anti-patterns to avoid:

- `Allow group X to manage all-resources in tenancy` outside a break-glass administrator policy
- Attaching policies at the tenancy level when a compartment would do — they cascade
- Granting `manage` where `use` suffices; `use` cannot create or delete

Note that instances need `use subnets` and `use vnics` in whichever compartment holds the network, which is frequently a different compartment from the workload.

## Dynamic Groups

Dynamic groups let non-human principals (instances, functions, OKE clusters) act without keys. Membership is defined by a matching rule over resource attributes.

```hcl
resource "oci_identity_dynamic_group" "terraform_runners" {
  provider = oci.home

  compartment_id = var.tenancy_ocid
  name           = "terraform-runners"
  description    = "Compute instances permitted to run Terraform"

  matching_rule = join("", [
    "All {",
    "instance.compartment.id = '${oci_identity_compartment.workload["tooling"].id}',",
    "tag.Operations.Role.value = 'terraform-runner'",
    "}",
  ])
}

resource "oci_identity_policy" "terraform_runners" {
  provider = oci.home

  compartment_id = var.tenancy_ocid
  name           = "terraform-runners"
  description    = "Permissions for instance-principal Terraform runs"

  statements = [
    "Allow dynamic-group terraform-runners to manage all-resources in compartment ${var.environment}",
    "Allow dynamic-group terraform-runners to read objectstorage-namespaces in tenancy",
  ]
}
```

Matching rules support `All` and `Any`, and can match on `instance.id`, `instance.compartment.id`, `tag.<namespace>.<key>.value`, `resource.type`, and `resource.compartment.id`.

For OKE workload identity the rule targets the cluster and service account:

```hcl
matching_rule = "ALL {resource.type = 'workload', resource.compartment.id = '${var.compartment_ocid}'}"
```

## Identity Domains

Newer tenancies use identity domains, which nest users and groups inside a domain rather than directly in the tenancy. Resource types differ:

| Legacy IAM | Identity Domains |
|------------|------------------|
| `oci_identity_user` | `oci_identity_domains_user` |
| `oci_identity_group` | `oci_identity_domains_group` |
| `oci_identity_dynamic_group` | `oci_identity_domains_dynamic_resource_group` |

Identity-domain resources require an `idcs_endpoint` argument:

```hcl
data "oci_identity_domains" "default" {
  compartment_id = var.tenancy_ocid
  display_name   = "Default"
}

locals {
  idcs_endpoint = one(data.oci_identity_domains.default.domains[*].url)
}

resource "oci_identity_domains_group" "app_team" {
  provider = oci.home

  idcs_endpoint = local.idcs_endpoint
  display_name  = "AppTeam"
  schemas       = ["urn:ietf:params:scim:schemas:core:2.0:Group"]
}
```

Policies still use `oci_identity_policy`, but the group reference becomes `<domain-name>/<group-name>`:

```
Allow group 'Default'/'AppTeam' to manage instance-family in compartment dev
```

Check which model a tenancy uses before writing identity code; mixing them produces confusing failures.

## Quotas

Quotas are compartment-scoped policy statements that cap service consumption. They are the practical defense for sandbox compartments.

```hcl
resource "oci_limits_quota" "dev_caps" {
  compartment_id = var.tenancy_ocid
  name           = "dev-sandbox-caps"
  description    = "Consumption caps for the developer sandbox"

  statements = [
    "Set compute quota vm-standard-e4-flex-core-count to 32 in compartment dev:dev-sandbox",
    "Set compute quota vm-standard-e4-flex-memory-count to 256 in compartment dev:dev-sandbox",
    "Set block-storage quota total-storage-gb to 2000 in compartment dev:dev-sandbox",
    "Zero database quota adb-ocpu-count in compartment dev:dev-sandbox",
  ]
}
```

Quota verbs are `Set`, `Unset`, and `Zero`. `Zero` blocks the service outright, which is the cleanest way to keep expensive services out of a sandbox.

## Authorization Failure Diagnosis

OCI returns `NotAuthorizedOrNotFound` for both missing permissions and missing resources — deliberately, so that probing cannot enumerate resources. When Terraform reports it on an OCID you believe exists:

1. Confirm the OCID's compartment matches the compartment in the policy
2. Confirm the verb tier covers the operation (`use` cannot create)
3. Confirm the resource family name is right (`instance-family`, `virtual-network-family`, `object-family`)
4. For instance principals, confirm the dynamic group matching rule actually matches
5. Remember policy changes can take a short time to propagate

## Best Practices

- Model the compartment hierarchy before writing any resource code
- Set `enable_delete = false` on compartments unless you genuinely want Terraform to remove them
- Route every identity resource through a home-region provider alias
- Scope policies to the narrowest compartment that works
- Prefer dynamic groups and principals over user API keys for automation
- Apply quotas and budgets to every non-production compartment
- Keep the compartment tree in a dedicated root module owned by the platform team
- Validate OCID variable shapes so typos fail at plan time
