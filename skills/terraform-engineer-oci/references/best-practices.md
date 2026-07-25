# OCI Terraform Best Practices

## Naming Conventions

OCI display names are not identifiers — they are mutable labels, and OCI does not enforce uniqueness on most of them. That makes a convention more important, not less, because nothing will stop you creating three VCNs called `vcn`.

```hcl
locals {
  # <environment>-<workload>-<resource>
  name_prefix = "${var.environment}-${var.workload}"
}

resource "oci_core_vcn" "this" {
  display_name = "${local.name_prefix}-vcn"
}

resource "oci_core_subnet" "app" {
  display_name = "${local.name_prefix}-app-subnet"
}
```

Constraints that differ by resource and bite at apply time:

| Attribute | Rule |
|-----------|------|
| `dns_label` (VCN, subnet) | Lowercase alphanumeric, max 15 chars, immutable |
| `hostname_label` (VNIC) | Lowercase alphanumeric and hyphens, max 63 chars |
| Bucket `name` | Unique per namespace and region, immutable |
| `oci_identity_compartment.name` | Unique among siblings, no spaces |
| Tag namespace / key names | Immutable, cannot be deleted — only retired |

Terraform resource names follow the standard convention: `snake_case`, and `this` for a module's single primary resource.

```hcl
# Good
resource "oci_core_vcn" "this" {}
resource "oci_core_subnet" "app_private" {}

# Avoid
resource "oci_core_vcn" "vcn" {}          # redundant
resource "oci_core_subnet" "subnet1" {}   # meaningless
```

## DRY Patterns

**Use `for_each` over repetition.**

```hcl
# Avoid
resource "oci_core_subnet" "app" { cidr_block = "10.0.1.0/24" }
resource "oci_core_subnet" "data" { cidr_block = "10.0.2.0/24" }
resource "oci_core_subnet" "mgmt" { cidr_block = "10.0.3.0/24" }

# Prefer
variable "subnets" {
  type = map(object({
    cidr_block = string
    is_public  = optional(bool, false)
  }))
}

resource "oci_core_subnet" "this" {
  for_each = var.subnets

  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  cidr_block     = each.value.cidr_block
  display_name   = "${local.name_prefix}-${each.key}"
}
```

`for_each` over `count` wherever the collection can change. With `count`, removing the first subnet re-creates every subnet after it; with `for_each` keyed by name, it removes exactly one.

**Derive CIDRs rather than listing them.**

```hcl
locals {
  subnet_cidrs = {
    public = cidrsubnet(var.vcn_cidr, 8, 0)
    app    = cidrsubnet(var.vcn_cidr, 8, 1)
    data   = cidrsubnet(var.vcn_cidr, 8, 2)
  }
}
```

**Look up rather than hardcode.**

```hcl
# Avoid — breaks in any other tenancy, and goes stale
locals {
  ad          = "kIdk:PHX-AD-1"
  image_ocid  = "ocid1.image.oc1.phx.aaaaaaaa..."
}

# Prefer
data "oci_identity_availability_domains" "this" {
  compartment_id = var.tenancy_ocid
}

data "oci_core_images" "ol8" {
  compartment_id           = var.compartment_ocid
  operating_system         = "Oracle Linux"
  operating_system_version = "8"
  shape                    = var.instance_shape
  sort_by                  = "TIMECREATED"
  sort_order               = "DESC"
}
```

One caveat: a floating image lookup means a new platform image silently replaces instances on the next apply. For stable fleets, resolve the image once and pin the OCID in a variable, or add `ignore_changes = [source_details[0].source_id]`.

## Security

**Never inline secrets.** OCI Vault is the destination for anything sensitive.

```hcl
resource "oci_kms_vault" "this" {
  compartment_id = var.security_compartment_ocid
  display_name   = "${local.name_prefix}-vault"
  vault_type     = "DEFAULT" # VIRTUAL_PRIVATE for dedicated HSM partition
}

resource "oci_kms_key" "data" {
  compartment_id      = var.security_compartment_ocid
  display_name        = "${local.name_prefix}-data-key"
  management_endpoint = oci_kms_vault.this.management_endpoint

  key_shape {
    algorithm = "AES"
    length    = 32
  }
}

resource "oci_vault_secret" "db_password" {
  compartment_id = var.security_compartment_ocid
  vault_id       = oci_kms_vault.this.id
  key_id         = oci_kms_key.data.id
  secret_name    = "${local.name_prefix}-db-password"

  secret_content {
    content_type = "BASE64"
    content      = base64encode(random_password.db.result)
  }
}
```

Reading a secret back in Terraform writes it to state in cleartext. Prefer having the application read from Vault at runtime over passing the value through Terraform at all.

```hcl
# Only when the value genuinely must be a resource argument
data "oci_secrets_secretbundle" "db_password" {
  secret_id = oci_vault_secret.db_password.id
}
```

**Encrypt with customer-managed keys.** OCI encrypts at rest by default with Oracle-managed keys; regulated workloads usually need your own.

```hcl
resource "oci_objectstorage_bucket" "data" {
  compartment_id = var.compartment_ocid
  namespace      = local.namespace
  name           = "${local.name_prefix}-data"
  access_type    = "NoPublicAccess"
  versioning     = "Enabled"
  kms_key_id     = oci_kms_key.data.id
}

resource "oci_core_volume" "data" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
  display_name        = "${local.name_prefix}-data"
  size_in_gbs         = 512
  kms_key_id          = oci_kms_key.data.id
}

resource "oci_database_autonomous_database" "this" {
  compartment_id           = var.compartment_ocid
  db_name                  = replace("${local.name_prefix}db", "-", "")
  display_name             = "${local.name_prefix}-adb"
  db_workload              = "OLTP"
  compute_model            = "ECPU"
  compute_count            = var.adb_ecpu_count
  data_storage_size_in_gbs = var.adb_storage_gb
  is_auto_scaling_enabled  = true

  # Private endpoint only — no public access
  subnet_id          = var.subnet_id
  nsg_ids            = var.nsg_ids
  is_mtls_connection_required = true

  admin_password = var.adb_admin_password # sensitive, from Vault or CI secret
}
```

**Mark sensitive variables.**

```hcl
variable "adb_admin_password" {
  description = "Autonomous Database ADMIN password"
  type        = string
  sensitive   = true

  validation {
    condition = (
      length(var.adb_admin_password) >= 12 &&
      length(var.adb_admin_password) <= 30 &&
      can(regex("[A-Z]", var.adb_admin_password)) &&
      can(regex("[a-z]", var.adb_admin_password)) &&
      can(regex("[0-9]", var.adb_admin_password)) &&
      !can(regex("\"", var.adb_admin_password))
    )
    error_message = "ADB password must be 12-30 chars with upper, lower, and digit, and no double quotes."
  }
}
```

`sensitive = true` redacts plan output; it does **not** encrypt state. State security is a backend concern — see `state-management.md`.

**Enable logging and posture management.**

```hcl
resource "oci_logging_log_group" "this" {
  compartment_id = var.compartment_ocid
  display_name   = "${local.name_prefix}-logs"
}

resource "oci_logging_log" "vcn_flow" {
  display_name = "${local.name_prefix}-vcn-flow"
  log_group_id = oci_logging_log_group.this.id
  log_type     = "SERVICE"

  configuration {
    source {
      category    = "all"
      resource    = oci_core_subnet.app.id
      service     = "flowlogs"
      source_type = "OCISERVICE"
    }
    compartment_id = var.compartment_ocid
  }

  is_enabled         = true
  retention_duration = 90
}

resource "oci_cloud_guard_cloud_guard_configuration" "this" {
  provider = oci.home

  compartment_id   = var.tenancy_ocid
  reporting_region = var.home_region
  status           = "ENABLED"
}
```

Audit logging is on by default tenancy-wide and needs no Terraform. VCN flow logs do not — enable them explicitly per subnet.

## Bastion Access

Do not attach public IPs to management instances. The OCI Bastion service brokers SSH without one.

```hcl
resource "oci_bastion_bastion" "this" {
  compartment_id               = var.compartment_ocid
  bastion_type                 = "STANDARD"
  target_subnet_id             = oci_core_subnet.private.id
  name                         = replace("${local.name_prefix}-bastion", "-", "")
  client_cidr_block_allow_list = var.admin_cidrs

  max_session_ttl_in_seconds = 10800
}
```

## Code Organization

```
terraform/
├── bootstrap/                # state bucket, secret keys — local state
├── global/
│   ├── identity/             # compartments, policies, tag namespaces (home region)
│   └── budgets/
├── environments/
│   ├── prod/
│   │   ├── network/
│   │   ├── data/
│   │   └── app/
│   ├── stage/
│   └── dev/
├── modules/
│   ├── vcn/
│   ├── oke/
│   ├── autonomous-db/
│   └── instance-pool/
├── policy/                   # Conftest rego
└── README.md
```

Two OCI-specific reasons for this shape:

1. **`global/identity` is separate** because everything in it targets the home region and changes rarely. Mixing it with regional resources forces every root module to carry a home-region alias.
2. **`bootstrap/` is separate** because the state bucket and its Customer Secret Key cannot be managed by the state they hold.

## Environment Separation

Prefer directory-per-environment over workspaces. Workspaces share one configuration and one backend key prefix, which makes divergent environments awkward and a wrong `terraform workspace select` catastrophic.

```hcl
# environments/prod/network/main.tf
module "network" {
  source = "../../../modules/vcn"

  compartment_ocid = var.network_compartment_ocid
  name_prefix      = "prod-app"
  vcn_cidr         = "10.0.0.0/16"

  subnets = {
    public = { cidr_block = "10.0.0.0/24", is_public = true }
    app    = { cidr_block = "10.0.1.0/24" }
    data   = { cidr_block = "10.0.2.0/24" }
  }

  defined_tags = local.defined_tags
}
```

Each environment directory carries its own `backend.tf` key and its own `terraform.tfvars`. Compartments provide the isolation that workspaces cannot.

## Resource Lifecycle

```hcl
resource "oci_core_vcn" "this" {
  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      defined_tags["Oracle-Tags.CreatedBy"],
      defined_tags["Oracle-Tags.CreatedOn"],
    ]
  }
}

resource "oci_database_autonomous_database" "this" {
  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      # Auto-scaling adjusts this outside Terraform
      compute_count,
    ]
  }
}
```

`prevent_destroy` takes a literal only — it cannot reference `var.environment`. Where per-environment behavior is needed, rely on the directory split and set it in the production configuration.

## Availability Design

OCI's AD topology varies by region, and single-AD regions are common. Never assume three.

```hcl
locals {
  ad_names = [for ad in data.oci_identity_availability_domains.this.availability_domains : ad.name]
  ad_count = length(local.ad_names)
}

resource "oci_core_instance" "app" {
  count = var.instance_count

  # Correct in one-AD and three-AD regions alike
  availability_domain = local.ad_names[count.index % local.ad_count]
  fault_domain        = data.oci_identity_fault_domains.this.fault_domains[count.index % 3].name
}
```

In a single-AD region, fault domains are the only anti-affinity available — use them.

## Best Practices Checklist

- [ ] Provider pinned to a major version; `required_version` set
- [ ] Identity resources routed through a home-region provider alias
- [ ] No hardcoded OCIDs, AD names, or Object Storage namespaces
- [ ] `compartment_id` set explicitly on every resource that takes one
- [ ] Nothing created in the tenancy root compartment
- [ ] Tag namespaces and cost-tracking keys created before billable resources
- [ ] `Oracle-Tags` keys ignored to avoid perpetual diffs
- [ ] Remote state configured, with locking verified empirically
- [ ] State bucket versioned and encrypted with a customer-managed key
- [ ] Private subnets set `prohibit_public_ip_on_vnic = true`
- [ ] NSGs used for new rules, not duplicated in security lists
- [ ] Service gateway present and routed for OCI service traffic
- [ ] Secrets in OCI Vault; sensitive variables marked `sensitive = true`
- [ ] Customer-managed keys on buckets, volumes, and databases
- [ ] VCN flow logs enabled on subnets that carry workload traffic
- [ ] Budgets with forecast alerts on every spending compartment
- [ ] Quotas on non-production compartments
- [ ] Instances spread across fault domains, derived from data sources
- [ ] `for_each` preferred over `count` for collections
- [ ] Input variables validated, including OCID shape
- [ ] `terraform fmt` and `terraform validate` clean
- [ ] Plan-mode tests cover module logic and validation blocks
- [ ] Policy-as-code enforces tagging and compartment rules
