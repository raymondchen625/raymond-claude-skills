# Tagging and Cost Management

## OCI's Two Tagging Systems

OCI has two distinct tag types, and the difference matters for Terraform because they behave differently on every resource.

| | Freeform tags | Defined tags |
|---|---|---|
| Schema | None | Namespace + key, pre-created |
| Terraform type | `map(string)` | `map(string)` keyed `"Namespace.Key"` |
| Governance | None | Policy-controlled per namespace |
| Cost tracking | No | Yes, when the key is flagged |
| Value validation | No | Optional enumerated values |
| Applied automatically | No | Yes, via tag defaults |

Freeform tags are for convenience. Defined tags are the ones that show up in cost reports and can be enforced. Anything used for chargeback must be a defined tag.

```hcl
resource "oci_core_instance" "app" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
  shape               = "VM.Standard.E4.Flex"
  display_name        = "app-01"

  freeform_tags = {
    managed_by = "terraform"
    repo       = "github.com/org/infra"
  }

  defined_tags = {
    "Operations.Environment" = var.environment
    "Operations.CostCenter"  = var.cost_center
    "Operations.Owner"       = var.owner_email
  }
}
```

## Creating a Tag Namespace

Tag namespaces and tag keys are identity resources — home region only. Create them before any resource that references them; a defined tag reference to a non-existent key fails at apply.

```hcl
resource "oci_identity_tag_namespace" "operations" {
  provider = oci.home

  compartment_id = var.tenancy_ocid
  name           = "Operations"
  description    = "Standard operational tags"

  # Namespaces cannot be deleted, only retired.
  is_retired = false
}

resource "oci_identity_tag" "environment" {
  provider = oci.home

  tag_namespace_id = oci_identity_tag_namespace.operations.id
  name             = "Environment"
  description      = "Deployment environment"
  is_cost_tracking = true

  validator {
    validator_type = "ENUM"
    values         = ["prod", "stage", "dev", "sandbox"]
  }
}

resource "oci_identity_tag" "cost_center" {
  provider = oci.home

  tag_namespace_id = oci_identity_tag_namespace.operations.id
  name             = "CostCenter"
  description      = "Chargeback code"
  is_cost_tracking = true
}

resource "oci_identity_tag" "owner" {
  provider = oci.home

  tag_namespace_id = oci_identity_tag_namespace.operations.id
  name             = "Owner"
  description      = "Owning team email"
  is_cost_tracking = false
}
```

Two constraints worth knowing up front:

- **`is_cost_tracking` is capped.** A tenancy allows a limited number of cost-tracking keys (10 by default). Choose them deliberately — environment, cost center, and application are usually the right three.
- **Namespaces and keys cannot be deleted.** They can only be retired (`is_retired = true`). Name them carefully; a typo is permanent.

The `ENUM` validator is worth using on environment-style keys. It rejects `Prod` when the allowed value is `prod`, which stops cost reports fragmenting across casing variants.

## Tag Defaults and the Perpetual Diff

A tag default applies a defined tag automatically to every resource created in a compartment.

```hcl
resource "oci_identity_tag_default" "environment" {
  provider = oci.home

  compartment_id    = oci_identity_compartment.env.id
  tag_definition_id = oci_identity_tag.environment.id
  value             = var.environment

  # Block resource creation when the tag cannot be applied
  is_required = true
}
```

This is excellent governance and a well-known Terraform annoyance. The service applies the tag, Terraform's configuration does not declare it, and every subsequent plan shows a diff trying to remove it.

Three ways to handle it, in order of preference:

**1. Declare the same tags in configuration.** Cleanest, keeps state truthful.

```hcl
locals {
  default_tags = {
    "Operations.Environment" = var.environment
    "Operations.CostCenter"  = var.cost_center
  }
}

resource "oci_core_instance" "app" {
  # ...
  defined_tags = merge(local.default_tags, {
    "Operations.Owner" = var.owner_email
  })
}
```

**2. Ignore the attribute** when the defaults are managed entirely outside Terraform.

```hcl
resource "oci_core_instance" "app" {
  # ...
  lifecycle {
    ignore_changes = [defined_tags]
  }
}
```

This hides genuine tag drift too, so use it only where tag management is deliberately external.

**3. Ignore specific keys** where the provider supports it. Terraform can target map elements:

```hcl
lifecycle {
  ignore_changes = [
    defined_tags["Oracle-Tags.CreatedBy"],
    defined_tags["Oracle-Tags.CreatedOn"],
  ]
}
```

The `Oracle-Tags` namespace is created automatically in most tenancies and stamps `CreatedBy` and `CreatedOn` on new resources. Those two keys are the most common source of unexpected diffs, and ignoring exactly them is usually right.

## Centralizing the Tag Model

Because OCI has no provider-level `default_tags` equivalent, tag consistency is a code convention. Build the maps once in locals and merge at each resource.

```hcl
locals {
  # Defined tags — governed, cost-tracked
  common_defined_tags = {
    "Operations.Environment" = var.environment
    "Operations.CostCenter"  = var.cost_center
    "Operations.Owner"       = var.owner_email
  }

  # Freeform tags — ad hoc, ungoverned
  common_freeform_tags = {
    managed_by = "terraform"
    module     = basename(abspath(path.module))
    repo       = var.source_repository
  }
}

resource "oci_core_vcn" "this" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = [var.vcn_cidr]
  display_name   = "${var.name_prefix}-vcn"

  defined_tags  = merge(local.common_defined_tags, var.additional_defined_tags)
  freeform_tags = merge(local.common_freeform_tags, var.additional_freeform_tags)
}
```

Expose both as module inputs so callers can extend without editing the module:

```hcl
variable "additional_defined_tags" {
  description = "Extra defined tags, keyed \"Namespace.Key\""
  type        = map(string)
  default     = {}
}

variable "additional_freeform_tags" {
  description = "Extra freeform tags"
  type        = map(string)
  default     = {}
}
```

## Budgets

Budgets attach to a compartment and alert on actual or forecast spend.

```hcl
resource "oci_budget_budget" "env" {
  compartment_id = var.tenancy_ocid # budgets live in the tenancy
  target_type    = "COMPARTMENT"
  targets        = [oci_identity_compartment.env.id]

  display_name = "${var.environment}-monthly"
  description  = "Monthly spend cap for ${var.environment}"
  amount       = var.monthly_budget
  reset_period = "MONTHLY"

  freeform_tags = local.common_freeform_tags
}

resource "oci_budget_alert_rule" "forecast_90" {
  budget_id      = oci_budget_budget.env.id
  display_name   = "forecast-90pct"
  type           = "FORECAST"
  threshold      = 90
  threshold_type = "PERCENTAGE"
  message        = "${var.environment} is forecast to exceed 90% of budget"
  recipients     = join(",", var.budget_alert_emails)
}

resource "oci_budget_alert_rule" "actual_100" {
  budget_id      = oci_budget_budget.env.id
  display_name   = "actual-100pct"
  type           = "ACTUAL"
  threshold      = 100
  threshold_type = "PERCENTAGE"
  message        = "${var.environment} has exceeded its monthly budget"
  recipients     = join(",", var.budget_alert_emails)
}
```

A budget can also target a cost-tracking tag instead of a compartment, which is how you budget per application across compartments:

```hcl
resource "oci_budget_budget" "per_app" {
  compartment_id = var.tenancy_ocid
  target_type    = "TAG"
  targets        = ["Operations.Application.${var.application_name}"]
  amount         = var.monthly_budget
  reset_period   = "MONTHLY"
  display_name   = "${var.application_name}-monthly"
}
```

`FORECAST` alerts are the useful ones — an `ACTUAL` alert at 100% tells you the money is already spent.

## Cost-Aware Sizing

OCI's flexible shapes make environment-based sizing straightforward, since OCPU and memory are independent.

```hcl
locals {
  shape_config = {
    prod = {
      shape  = "VM.Standard.E4.Flex"
      ocpus  = 8
      memory = 128
    }
    stage = {
      shape  = "VM.Standard.E4.Flex"
      ocpus  = 2
      memory = 32
    }
    dev = {
      shape  = "VM.Standard.A1.Flex" # Ampere, materially cheaper
      ocpus  = 1
      memory = 6
    }
  }

  selected = local.shape_config[var.environment]
}

resource "oci_core_instance" "app" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
  shape               = local.selected.shape

  shape_config {
    ocpus         = local.selected.ocpus
    memory_in_gbs = local.selected.memory
  }
}
```

Cost levers specific to OCI worth encoding in modules:

- **Ampere A1 shapes** cost substantially less per core than x86 for workloads that run on ARM
- **Autonomous Database auto-scaling** (`is_auto_scaling_enabled`) scales to 3x base and bills actual usage
- **Block volume performance tiers** — `vpus_per_gb = 0` (Lower Cost) through `120` (Ultra High). Boot volumes rarely need above `10`
- **Object Storage lifecycle rules** move data to Infrequent Access and Archive automatically
- **Compute capacity reservations** only pay off at sustained high utilization

```hcl
resource "oci_core_volume" "data" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
  display_name        = "app-data"
  size_in_gbs         = 512

  # 10 = Balanced, adequate for most workloads
  vpus_per_gb = var.environment == "prod" ? 20 : 0

  defined_tags = local.common_defined_tags
}

resource "oci_objectstorage_object_lifecycle_policy" "archive" {
  bucket    = oci_objectstorage_bucket.backups.name
  namespace = local.namespace

  rules {
    name        = "to-infrequent-access"
    action      = "INFREQUENT_ACCESS"
    time_amount = 30
    time_unit   = "DAYS"
    is_enabled  = true
  }

  rules {
    name        = "to-archive"
    action      = "ARCHIVE"
    time_amount = 90
    time_unit   = "DAYS"
    is_enabled  = true
  }

  rules {
    name        = "delete"
    action      = "DELETE"
    time_amount = 365
    time_unit   = "DAYS"
    is_enabled  = true
  }
}
```

## Best Practices

- Create tag namespaces and keys before anything that references them
- Use defined tags for anything that feeds chargeback; freeform tags for everything else
- Mark cost-tracking keys deliberately — the tenancy limit is low
- Add `ENUM` validators to environment and tier keys to stop casing drift
- Expect `Oracle-Tags.CreatedBy` and `CreatedOn` diffs and ignore those keys specifically
- Prefer declaring tag-default values in configuration over ignoring `defined_tags` wholesale
- Set a budget with a `FORECAST` alert on every compartment that can spend money
- Pair budgets with quotas — budgets alert, quotas actually stop spend
- Build tag maps in locals and merge; OCI has no provider-level default tags
