# State Management on OCI

## The Core Constraint

Terraform ships no native `oci` backend. There is no equivalent of `backend "s3"` or `backend "azurerm"` maintained for Oracle Cloud. That leaves three viable options, and the choice has real consequences for state locking.

| Option | Locking | Best for |
|--------|---------|----------|
| OCI Resource Manager | Yes, native | Teams wanting a managed, audited service |
| S3-compatible backend to Object Storage | Conditional — verify | Teams already running standard Terraform CLI |
| HTTP backend / Terraform Cloud | Yes | Existing multi-cloud tooling |

Decide this before the first `terraform apply`. Migrating state later is possible but disruptive.

## Option 1: OCI Resource Manager

Resource Manager is Oracle's managed Terraform service. It stores state, provides locking, records job history, and integrates with OCI IAM — the closest analogue to Terraform Cloud, billed as part of OCI.

Concepts: a **stack** is a Terraform configuration plus its variables and state; a **job** is a plan, apply, or destroy run against a stack.

```hcl
resource "oci_resourcemanager_stack" "app" {
  compartment_id = var.compartment_ocid
  display_name   = "app-infrastructure"
  description    = "Application infrastructure managed by Resource Manager"

  config_source {
    config_source_type = "ZIP_UPLOAD"
    zip_file_base64encoded = filebase64("${path.module}/config.zip")
    working_directory      = ""
  }

  variables = {
    compartment_ocid = var.compartment_ocid
    region           = var.region
    environment      = var.environment
  }

  terraform_version = "1.5.x"

  defined_tags = local.common_defined_tags
}
```

Git-backed stacks avoid the zip-upload cycle:

```hcl
resource "oci_resourcemanager_stack" "app" {
  compartment_id = var.compartment_ocid
  display_name   = "app-infrastructure"

  config_source {
    config_source_type        = "GIT_CONFIG_SOURCE"
    configuration_source_provider_id = oci_resourcemanager_configuration_source_provider.github.id
    repository_url            = "https://github.com/org/infra.git"
    branch_name               = "main"
    working_directory         = "environments/prod"
  }

  terraform_version = "1.5.x"
}
```

Strengths: locking works, state never touches a developer laptop, every job is audited against an IAM identity, and drift detection is built in. Cost: you are running Terraform inside a managed service, so local iteration is slower and the CLI experience is indirect.

Resource Manager is the recommended choice for teams with no existing Terraform automation.

## Option 2: S3-Compatible Backend to Object Storage

OCI Object Storage exposes an Amazon S3-compatible API, and Terraform's `s3` backend can target it. This is the most widely used approach because it keeps the standard CLI workflow.

### Prerequisites

**A Customer Secret Key.** This is an OCI user credential that produces an S3-style access key and secret. It is not the API signing key.

```hcl
resource "oci_identity_customer_secret_key" "terraform_state" {
  provider = oci.home

  user_id      = var.terraform_user_ocid
  display_name = "terraform-state-backend"
}

output "state_access_key" {
  description = "S3-compatible access key ID"
  value       = oci_identity_customer_secret_key.terraform_state.id
}

output "state_secret_key" {
  description = "S3-compatible secret access key"
  value       = oci_identity_customer_secret_key.terraform_state.key
  sensitive   = true
}
```

Bootstrapping problem: the bucket and credential holding your state cannot themselves be managed by state in that bucket. Create them in a separate root module with local state committed nowhere, or by hand.

**A bucket, versioned.**

```hcl
data "oci_objectstorage_namespace" "this" {
  compartment_id = var.tenancy_ocid
}

resource "oci_objectstorage_bucket" "state" {
  compartment_id = var.compartment_ocid
  namespace      = data.oci_objectstorage_namespace.this.namespace
  name           = "terraform-state"
  access_type    = "NoPublicAccess"

  # Non-negotiable: state history is your recovery path
  versioning = "Enabled"

  # Customer-managed encryption
  kms_key_id = oci_kms_key.state.id

  defined_tags = local.common_defined_tags
}
```

### Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "us-phoenix-1"

    endpoints = {
      s3 = "https://<namespace>.compat.objectstorage.us-phoenix-1.oraclecloud.com"
    }

    # Required for any non-AWS S3 implementation
    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true
    skip_s3_checksum            = true
    use_path_style              = true
  }
}
```

Every skip flag is load-bearing. Without them the backend attempts AWS-specific calls (STS identity lookup, IMDS probe, checksum headers OCI rejects) and fails in ways whose error messages point nowhere useful.

Credentials come from the standard AWS environment variables, holding the OCI Customer Secret Key values:

```bash
export AWS_ACCESS_KEY_ID="<customer secret key id>"
export AWS_SECRET_ACCESS_KEY="<customer secret key>"
```

Keep the endpoint out of committed code by using partial configuration:

```hcl
# backend.tf
terraform {
  backend "s3" {}
}
```

```hcl
# config/backend-prod.hcl
bucket = "terraform-state"
key    = "prod/network/terraform.tfstate"
region = "us-phoenix-1"

endpoints = {
  s3 = "https://mynamespace.compat.objectstorage.us-phoenix-1.oraclecloud.com"
}

skip_region_validation      = true
skip_credentials_validation = true
skip_metadata_api_check     = true
skip_requesting_account_id  = true
skip_s3_checksum            = true
use_path_style              = true
```

```bash
terraform init -backend-config=config/backend-prod.hcl
```

### The Locking Caveat

**Read this before putting a team on the S3-compatible backend.**

The classic S3 locking mechanism uses a DynamoDB table. OCI has no DynamoDB, so `dynamodb_table` is unavailable. Historically this meant OCI Object Storage state had **no locking at all** — two concurrent applies would silently race and one would lose its writes.

Terraform 1.10 introduced `use_lockfile`, which implements locking through S3 conditional writes (an `If-None-Match` precondition) instead of DynamoDB:

```hcl
terraform {
  backend "s3" {
    bucket       = "terraform-state"
    key          = "prod/network/terraform.tfstate"
    use_lockfile = true
    # ... endpoint and skip flags as above
  }
}
```

Whether this works against OCI Object Storage depends on that implementation supporting conditional writes with the same semantics. **Verify it in your own tenancy before relying on it**, with a concrete test:

```bash
# Terminal 1 — hold a lock
terraform apply

# Terminal 2 — while the first is mid-apply, this MUST fail to acquire
terraform apply
```

If the second run proceeds, you have no locking. In that case treat the backend as single-writer and enforce serialization elsewhere: a CI pipeline concurrency group, an environment lock, or a move to Resource Manager. Do not assume it works because the argument was accepted.

## Option 3: Terraform Cloud / HTTP Backend

If the organization already runs Terraform Cloud or a self-hosted state server, keep using it. The OCI provider is unremarkable in that setting and locking is a solved problem.

```hcl
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "oci-prod-network"
    }
  }
}
```

## State File Organization

Split state by blast radius, not by convenience. Networking and identity change slowly and break everything; application resources change constantly.

```
terraform-state/
├── prod/
│   ├── identity/terraform.tfstate      # compartments, policies, tags
│   ├── network/terraform.tfstate       # VCN, gateways, NSGs
│   ├── data/terraform.tfstate          # ADB, object storage
│   └── app/terraform.tfstate           # compute, load balancers
├── stage/
│   ├── network/terraform.tfstate
│   └── app/terraform.tfstate
└── shared/
    └── tooling/terraform.tfstate
```

Cross-state references use `terraform_remote_state`, though passing OCIDs as variables is often simpler and avoids coupling:

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "terraform-state"
    key    = "prod/network/terraform.tfstate"
    region = "us-phoenix-1"

    endpoints = {
      s3 = "https://${var.namespace}.compat.objectstorage.us-phoenix-1.oraclecloud.com"
    }

    skip_region_validation      = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_requesting_account_id  = true
    use_path_style              = true
  }
}

resource "oci_core_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.private_subnet_id
}
```

## Importing Existing Resources

OCI adoption almost always starts with resources created through the console. Use `import` blocks (Terraform 1.5+) rather than the CLI — they are reviewable, planable, and can generate configuration.

```hcl
import {
  to = oci_core_vcn.this
  id = "ocid1.vcn.oc1.phx.amaaaaaa..."
}

import {
  to = oci_core_subnet.private
  id = "ocid1.subnet.oc1.phx.aaaaaaaa..."
}
```

Generate the configuration rather than writing it by hand:

```bash
terraform plan -generate-config-out=generated.tf
```

Review the output carefully before adopting it. Generated configuration includes computed and default attributes that will produce noisy diffs, and it will not include `lifecycle` blocks you need.

Import identifiers are almost always the plain OCID, but composite resources differ. Check the resource documentation — some take a compound form:

```bash
# Some resources use compound IDs
terraform import oci_core_default_security_list.default \
  "ocid1.securitylist.oc1.phx.aaaa..."

# Bucket imports use namespace and name, not an OCID
terraform import oci_objectstorage_bucket.state \
  "n/<namespace>/b/<bucket-name>"
```

For bulk adoption, the OCI CLI enumerates resources to import:

```bash
oci search resource structured-search \
  --query-text "query vcn resources where compartmentId = '<ocid>'" \
  --query 'data.items[*].identifier' --raw-output
```

## State Operations

```bash
# Inspect
terraform state list
terraform state show oci_core_vcn.this

# Refactor without destroying
terraform state mv oci_core_vcn.main oci_core_vcn.this
terraform state mv 'oci_core_subnet.private' 'module.network.oci_core_subnet.private'

# Stop managing without destroying
terraform state rm oci_core_instance.legacy

# Back up before anything destructive
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate
```

Prefer `moved` blocks over `state mv` for refactors that ship to other people:

```hcl
moved {
  from = oci_core_vcn.main
  to   = oci_core_vcn.this
}
```

## State Security

State files contain resource attributes in cleartext, including anything the provider marks sensitive — ADB admin passwords, generated keys, secret contents. Treat the state bucket as a secrets store.

```hcl
resource "oci_kms_key" "state" {
  compartment_id      = var.security_compartment_ocid
  display_name        = "terraform-state-key"
  management_endpoint = oci_kms_vault.this.management_endpoint

  key_shape {
    algorithm = "AES"
    length    = 32
  }
}

resource "oci_objectstorage_bucket" "state" {
  compartment_id = var.compartment_ocid
  namespace      = data.oci_objectstorage_namespace.this.namespace
  name           = "terraform-state"
  access_type    = "NoPublicAccess"
  versioning     = "Enabled"
  kms_key_id     = oci_kms_key.state.id
}
```

Restrict access with a tight policy:

```hcl
resource "oci_identity_policy" "state_access" {
  provider = oci.home

  compartment_id = var.security_compartment_ocid
  name           = "terraform-state-access"
  description    = "Only CI may read or write Terraform state"

  statements = [
    "Allow dynamic-group terraform-runners to manage objects in compartment security where target.bucket.name = 'terraform-state'",
    "Allow dynamic-group terraform-runners to read buckets in compartment security",
  ]
}
```

## Best Practices

- Choose the backend before the first apply; migrating later is disruptive
- Verify locking empirically with two concurrent applies — never assume it
- Enable bucket versioning; it is the only practical state recovery path
- Encrypt the state bucket with a customer-managed key
- Split state by blast radius: identity, network, data, application
- Never commit state files or `.terraform` directories
- Bootstrap the state bucket and credentials outside the state they hold
- Use `import` blocks and `moved` blocks over their CLI equivalents
- Back up state before any `state rm`, `state mv`, or provider major upgrade
- Rotate Customer Secret Keys on the same schedule as API signing keys
