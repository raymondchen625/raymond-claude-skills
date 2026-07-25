# OCI Provider Configuration and Authentication

## Provider Source and Version

The OCI provider is published by Oracle, not HashiCorp. The source address is `oracle/oci`.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "~> 6.0"
    }
  }
}
```

Check the [registry](https://registry.terraform.io/providers/oracle/oci/latest) for the current major version before starting a new project. The provider tracks OCI service releases closely and ships frequently, so pin to a major version and upgrade deliberately.

## Authentication Methods

OCI supports five authentication modes. The `auth` argument selects the mode; API key is the default.

### API Key (default)

Signing-key based authentication. Suitable for local development and for CI systems that cannot use a principal.

```hcl
provider "oci" {
  auth             = "ApiKey" # default, may be omitted
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = var.private_key_path
  region           = var.region
}
```

Prefer reading from the standard CLI config file rather than passing key material through Terraform variables:

```hcl
provider "oci" {
  config_file_profile = "DEFAULT" # reads ~/.oci/config
  region              = var.region
}
```

Environment variable equivalents, which is what CI should use:

```bash
export TF_VAR_tenancy_ocid="ocid1.tenancy.oc1..aaaa..."
export TF_VAR_user_ocid="ocid1.user.oc1..aaaa..."
export TF_VAR_fingerprint="aa:bb:cc:..."
export TF_VAR_private_key_path="/secure/path/oci_api_key.pem"
export TF_VAR_region="us-phoenix-1"
```

The provider also accepts `private_key` (the PEM contents) and `private_key_password`. Pass these only from a secret store, never from a committed `.tfvars`.

### Instance Principal

For Terraform running on an OCI compute instance. No key material at all — the instance's identity is derived from a dynamic group.

```hcl
provider "oci" {
  auth   = "InstancePrincipal"
  region = var.region
}
```

Requires a dynamic group matching the instance and a policy granting it permissions. See `compartments-and-identity.md`.

### Resource Principal

For Terraform running inside OCI Functions or a similar managed runtime.

```hcl
provider "oci" {
  auth   = "ResourcePrincipal"
  region = var.region
}
```

### Security Token (session auth)

For local development with federated or MFA-protected users. Produced by `oci session authenticate`, which opens a browser and writes a short-lived token.

```hcl
provider "oci" {
  auth                = "SecurityToken"
  config_file_profile = "my-session-profile"
  region              = var.region
}
```

Tokens expire (one hour by default, refreshable up to the session limit). Refresh with `oci session refresh --profile my-session-profile`. Do not use this mode in automation.

### OKE Workload Identity

For Terraform running as a pod in an OKE cluster, authenticating as the Kubernetes service account.

```hcl
provider "oci" {
  auth   = "OkeWorkloadIdentity"
  region = var.region
}
```

## Home Region and Identity Resources

This is the single most common OCI-specific mistake. Identity resources exist only in the tenancy's **home region**:

- `oci_identity_compartment`
- `oci_identity_group`, `oci_identity_user`, `oci_identity_user_group_membership`
- `oci_identity_policy`
- `oci_identity_dynamic_group`
- `oci_identity_tag_namespace`, `oci_identity_tag`
- `oci_identity_domain`

Creating them against a non-home region fails or produces confusing authorization errors. Route them through a dedicated provider alias.

```hcl
variable "region" {
  description = "Region for regional resources"
  type        = string
}

variable "home_region" {
  description = "Tenancy home region, e.g. us-ashburn-1"
  type        = string
}

provider "oci" {
  region = var.region
}

provider "oci" {
  alias  = "home"
  region = var.home_region
}

resource "oci_identity_compartment" "app" {
  provider       = oci.home
  compartment_id = var.tenancy_ocid
  name           = "app-prod"
  description    = "Production application compartment"
}
```

Passing the home region as a variable keeps provider configuration static, which matters because provider blocks are evaluated before most data sources resolve. If you must discover it, do so in a separate root module and pass the value forward:

```hcl
data "oci_identity_region_subscriptions" "this" {
  tenancy_id = var.tenancy_ocid
}

output "home_region" {
  description = "Tenancy home region name"
  value = one([
    for r in data.oci_identity_region_subscriptions.this.region_subscriptions :
    r.region_name if r.is_home_region
  ])
}
```

## Multi-Region Configuration

OCI has no cross-region resource references, so multi-region deployments use one aliased provider per region.

```hcl
provider "oci" {
  alias  = "phoenix"
  region = "us-phoenix-1"
}

provider "oci" {
  alias  = "ashburn"
  region = "us-ashburn-1"
}

resource "oci_core_vcn" "primary" {
  provider       = oci.phoenix
  compartment_id = var.compartment_ocid
  cidr_blocks    = ["10.0.0.0/16"]
  display_name   = "primary-vcn"
}

resource "oci_core_vcn" "dr" {
  provider       = oci.ashburn
  compartment_id = var.compartment_ocid
  cidr_blocks    = ["10.1.0.0/16"]
  display_name   = "dr-vcn"
}
```

Passing aliased providers into a module:

```hcl
module "network_phoenix" {
  source = "./modules/network"

  providers = {
    oci = oci.phoenix
  }

  compartment_ocid = var.compartment_ocid
  vcn_cidr         = "10.0.0.0/16"
}
```

The module declares what it expects:

```hcl
# modules/network/versions.tf
terraform {
  required_providers {
    oci = {
      source                = "oracle/oci"
      version               = "~> 6.0"
      configuration_aliases = [oci.home]
    }
  }
}
```

## Cross-Tenancy Access

OCI has no equivalent of AWS `assume_role`. Cross-tenancy access is granted declaratively with `Endorse`, `Admit`, and `Define` statements on both sides, and Terraform still authenticates as a principal in one tenancy. Model each tenancy as its own provider alias with its own credentials rather than trying to assume into one.

## Retry and Timeout Behavior

Many OCI operations are asynchronous and return a work request. The provider polls these. Two knobs matter:

```hcl
provider "oci" {
  region = var.region

  # Retry duration for retryable service errors (seconds)
  retry_duration_seconds = 120

  # Disable auto-retries entirely (rarely useful)
  disable_auto_retries = false
}
```

For individual slow resources, raise the resource-level timeouts rather than the global retry window:

```hcl
resource "oci_containerengine_cluster" "this" {
  compartment_id     = var.compartment_ocid
  kubernetes_version = var.kubernetes_version
  name               = "prod-oke"
  vcn_id             = oci_core_vcn.this.id

  timeouts {
    create = "60m"
    update = "60m"
    delete = "60m"
  }
}
```

## Realms and Non-Commercial Regions

The commercial realm is OC1. Government and sovereign realms (OC2, OC3, OC4, and others) use different endpoint domains and separate tenancies. If you target one, the region identifier alone is not enough — verify the provider supports the realm and that your Object Storage endpoints use the right domain suffix.

## Common Data Sources

These replace hardcoded values that differ per tenancy or region.

```hcl
# Availability domains — names carry a tenancy-specific prefix like "kIdk:PHX-AD-1"
data "oci_identity_availability_domains" "this" {
  compartment_id = var.tenancy_ocid
}

# Fault domains within an AD
data "oci_identity_fault_domains" "this" {
  compartment_id      = var.compartment_ocid
  availability_domain = data.oci_identity_availability_domains.this.availability_domains[0].name
}

# Object Storage namespace — unique per tenancy
data "oci_objectstorage_namespace" "this" {
  compartment_id = var.tenancy_ocid
}

# Latest platform image for a shape
data "oci_core_images" "ol8" {
  compartment_id           = var.compartment_ocid
  operating_system         = "Oracle Linux"
  operating_system_version = "8"
  shape                    = "VM.Standard.E4.Flex"
  sort_by                  = "TIMECREATED"
  sort_order               = "DESC"
}

locals {
  ad_names  = [for ad in data.oci_identity_availability_domains.this.availability_domains : ad.name]
  image_id  = data.oci_core_images.ol8.images[0].id
  namespace = data.oci_objectstorage_namespace.this.namespace
}
```

## Provider Best Practices

- Pin the provider to a major version and upgrade in a non-production tenancy first
- Use instance or resource principals in automation; reserve API keys for local work
- Keep provider blocks free of computed values so `init` and `plan` stay stable
- Declare `configuration_aliases` in modules that need a home-region provider
- Never commit `~/.oci/config` or the PEM private key
- Rotate API signing keys on a schedule and remove stale keys from the user
- Set `region` from a variable so the same module serves every region
- Prefer `config_file_profile` over inlining key material in Terraform variables
