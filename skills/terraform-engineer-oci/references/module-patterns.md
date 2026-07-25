# OCI Module Patterns

## Module Structure

```
modules/vcn/
├── main.tf          # resources
├── variables.tf     # inputs with validation
├── outputs.tf       # exported values
├── versions.tf      # terraform and provider constraints
├── README.md        # interface documentation
└── examples/
    ├── minimal/
    └── complete/
```

`versions.tf` in every module, including child modules:

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

If the module manages identity resources, declare the home-region alias it expects:

```hcl
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

## A Complete VCN Module

**`variables.tf`**

```hcl
variable "compartment_ocid" {
  description = "OCID of the compartment for network resources"
  type        = string

  validation {
    condition     = can(regex("^ocid1\\.(compartment|tenancy)\\.", var.compartment_ocid))
    error_message = "compartment_ocid must be a compartment or tenancy OCID."
  }
}

variable "name_prefix" {
  description = "Prefix for all resource display names"
  type        = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,28}[a-z0-9]$", var.name_prefix))
    error_message = "name_prefix must be lowercase alphanumeric with hyphens, 3-30 characters."
  }
}

variable "vcn_cidr" {
  description = "CIDR block for the VCN"
  type        = string

  validation {
    condition     = can(cidrhost(var.vcn_cidr, 0))
    error_message = "vcn_cidr must be a valid IPv4 CIDR block."
  }

  validation {
    condition     = tonumber(split("/", var.vcn_cidr)[1]) <= 24
    error_message = "vcn_cidr must be /24 or larger to allow subnet allocation."
  }
}

variable "dns_label" {
  description = "VCN DNS label; immutable after creation"
  type        = string
  default     = null

  validation {
    condition     = var.dns_label == null || can(regex("^[a-z][a-z0-9]{0,14}$", var.dns_label))
    error_message = "dns_label must start with a letter, be lowercase alphanumeric, max 15 characters."
  }
}

variable "subnets" {
  description = "Subnets to create, keyed by name"
  type = map(object({
    cidr_block  = string
    is_public   = optional(bool, false)
    dns_label   = optional(string)
  }))
  default = {}
}

variable "enable_nat_gateway" {
  description = "Whether to create a NAT gateway for private subnet egress"
  type        = bool
  default     = true
}

variable "enable_service_gateway" {
  description = "Whether to create a service gateway for private OCI service access"
  type        = bool
  default     = true
}

variable "defined_tags" {
  description = "Defined tags applied to all resources, keyed \"Namespace.Key\""
  type        = map(string)
  default     = {}
}

variable "freeform_tags" {
  description = "Freeform tags applied to all resources"
  type        = map(string)
  default     = {}
}
```

Multiple `validation` blocks on one variable are supported and read better than one compound condition, because each carries its own error message.

**`main.tf`**

```hcl
locals {
  tags = {
    defined  = var.defined_tags
    freeform = merge({ managed_by = "terraform" }, var.freeform_tags)
  }
}

resource "oci_core_vcn" "this" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = [var.vcn_cidr]
  display_name   = "${var.name_prefix}-vcn"
  dns_label      = var.dns_label

  defined_tags  = local.tags.defined
  freeform_tags = local.tags.freeform
}

resource "oci_core_internet_gateway" "this" {
  count = anytrue([for s in var.subnets : s.is_public]) ? 1 : 0

  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-igw"
  enabled        = true

  defined_tags  = local.tags.defined
  freeform_tags = local.tags.freeform
}

resource "oci_core_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0

  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-natgw"

  defined_tags  = local.tags.defined
  freeform_tags = local.tags.freeform
}

data "oci_core_services" "all" {
  count = var.enable_service_gateway ? 1 : 0

  filter {
    name   = "name"
    values = ["All .* Services In Oracle Services Network"]
    regex  = true
  }
}

resource "oci_core_service_gateway" "this" {
  count = var.enable_service_gateway ? 1 : 0

  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-sgw"

  services {
    service_id = data.oci_core_services.all[0].services[0].id
  }

  defined_tags  = local.tags.defined
  freeform_tags = local.tags.freeform
}

resource "oci_core_subnet" "this" {
  for_each = var.subnets

  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  cidr_block     = each.value.cidr_block
  display_name   = "${var.name_prefix}-${each.key}"
  dns_label      = each.value.dns_label

  route_table_id = each.value.is_public ? oci_core_route_table.public[0].id : oci_core_route_table.private.id

  prohibit_public_ip_on_vnic = !each.value.is_public
  prohibit_internet_ingress  = !each.value.is_public

  defined_tags  = local.tags.defined
  freeform_tags = local.tags.freeform
}
```

**`outputs.tf`**

```hcl
output "vcn_id" {
  description = "OCID of the VCN"
  value       = oci_core_vcn.this.id
}

output "vcn_cidr" {
  description = "CIDR block of the VCN"
  value       = one(oci_core_vcn.this.cidr_blocks)
}

output "subnet_ids" {
  description = "Map of subnet name to OCID"
  value       = { for k, v in oci_core_subnet.this : k => v.id }
}

output "nat_gateway_id" {
  description = "OCID of the NAT gateway, null when disabled"
  value       = try(oci_core_nat_gateway.this[0].id, null)
}

output "service_gateway_id" {
  description = "OCID of the service gateway, null when disabled"
  value       = try(oci_core_service_gateway.this[0].id, null)
}
```

Always export a map of OCIDs rather than a list. Callers reference `module.vcn.subnet_ids["app"]`, which survives adding a subnet; a list index does not.

## Conditional Resources

`count` with a boolean is the idiomatic toggle. Guard the reference with `try` or `one` so a disabled resource does not break outputs.

```hcl
resource "oci_core_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0
  # ...
}

# Safe reference
locals {
  nat_id = try(oci_core_nat_gateway.this[0].id, null)
}
```

For a set of independently optional resources, `for_each` over a filtered map reads better than several `count` blocks:

```hcl
locals {
  enabled_gateways = {
    for k, v in var.gateways : k => v if v.enabled
  }
}

resource "oci_core_drg_attachment" "this" {
  for_each = local.enabled_gateways

  drg_id       = oci_core_drg.this.id
  display_name = "${var.name_prefix}-${each.key}"

  network_details {
    id   = each.value.vcn_id
    type = "VCN"
  }
}
```

## Dynamic Blocks

NSG rules are the natural place for dynamic blocks, since the port structure varies by protocol.

```hcl
variable "ingress_rules" {
  description = "NSG ingress rules"
  type = list(object({
    description = string
    protocol    = string
    source      = string
    source_type = optional(string, "CIDR_BLOCK")
    min_port    = optional(number)
    max_port    = optional(number)
  }))
  default = []
}

resource "oci_core_network_security_group_security_rule" "ingress" {
  for_each = { for idx, r in var.ingress_rules : idx => r }

  network_security_group_id = oci_core_network_security_group.this.id
  direction                 = "INGRESS"
  protocol                  = each.value.protocol
  source                    = each.value.source
  source_type               = each.value.source_type
  description               = each.value.description

  dynamic "tcp_options" {
    for_each = each.value.protocol == "6" && each.value.min_port != null ? [1] : []
    content {
      destination_port_range {
        min = each.value.min_port
        max = each.value.max_port
      }
    }
  }

  dynamic "udp_options" {
    for_each = each.value.protocol == "17" && each.value.min_port != null ? [1] : []
    content {
      destination_port_range {
        min = each.value.min_port
        max = each.value.max_port
      }
    }
  }
}
```

Keying `for_each` by list index means inserting a rule mid-list re-creates everything after it. Key by something stable when the rule set changes often:

```hcl
for_each = { for r in var.ingress_rules : "${r.protocol}-${r.source}-${coalesce(r.min_port, 0)}" => r }
```

## Module Composition

A root module wires child modules together and owns the tag model.

```hcl
locals {
  defined_tags = {
    "Operations.Environment" = var.environment
    "Operations.CostCenter"  = var.cost_center
    "Operations.Owner"       = var.owner_email
  }

  freeform_tags = {
    managed_by = "terraform"
    repo       = var.source_repository
  }
}

module "network" {
  source = "../../modules/vcn"

  compartment_ocid = var.network_compartment_ocid
  name_prefix      = "${var.environment}-app"
  vcn_cidr         = var.vcn_cidr
  dns_label        = var.environment

  subnets = {
    public  = { cidr_block = cidrsubnet(var.vcn_cidr, 8, 0), is_public = true, dns_label = "public" }
    app     = { cidr_block = cidrsubnet(var.vcn_cidr, 8, 1), dns_label = "app" }
    data    = { cidr_block = cidrsubnet(var.vcn_cidr, 8, 2), dns_label = "data" }
  }

  enable_nat_gateway     = true
  enable_service_gateway = true

  defined_tags  = local.defined_tags
  freeform_tags = local.freeform_tags
}

module "database" {
  source = "../../modules/autonomous-db"

  compartment_ocid = var.data_compartment_ocid
  subnet_id        = module.network.subnet_ids["data"]
  nsg_ids          = [module.network.nsg_ids["db"]]
  name_prefix      = "${var.environment}-app"

  defined_tags  = local.defined_tags
  freeform_tags = local.freeform_tags
}
```

`cidrsubnet` beats hardcoded subnet CIDRs — the module stays correct when the VCN CIDR changes.

## Module Versioning

Pin module sources the same way you pin providers.

```hcl
# Git tag — preferred for internal modules
module "network" {
  source = "git::https://github.com/org/terraform-oci-modules.git//vcn?ref=v2.1.0"
}

# Terraform Registry
module "network" {
  source  = "oracle-terraform-modules/vcn/oci"
  version = "~> 3.5"
}

# Local path — development only, never in a shared root module
module "network" {
  source = "../../modules/vcn"
}
```

Oracle maintains verified modules under the `oracle-terraform-modules` namespace. They are a reasonable starting point but tend toward many inputs; wrapping one in a thin internal module that fixes your conventions is usually better than exposing all of it to callers.

## Handling Immutable Attributes

Several OCI attributes force replacement when changed. Guard the expensive ones.

```hcl
resource "oci_core_vcn" "this" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = [var.vcn_cidr]
  dns_label      = var.dns_label # immutable

  lifecycle {
    # A VCN replacement destroys every subnet and gateway inside it
    prevent_destroy = true

    ignore_changes = [
      defined_tags["Oracle-Tags.CreatedBy"],
      defined_tags["Oracle-Tags.CreatedOn"],
    ]
  }
}

resource "oci_core_instance" "app" {
  # source_details changes force replacement — pin the image explicitly
  source_details {
    source_type = "image"
    source_id   = var.image_ocid
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes = [
      # A new platform image should not silently rebuild the fleet
      source_details[0].source_id,
    ]
  }
}
```

`prevent_destroy` cannot reference variables — it must be a literal. To vary it by environment, split the resource or accept the constraint.

## Best Practices

- One module, one concern; a VCN module should not create databases
- Validate every OCID input with a regex on the resource type
- Export maps of OCIDs, never positional lists
- Accept `defined_tags` and `freeform_tags` as separate inputs
- Declare `configuration_aliases` when the module touches identity resources
- Use `cidrsubnet` rather than hardcoded subnet ranges
- Guard optional resource references with `try` or `one`
- Key `for_each` by a stable value, not a list index
- Pin module sources to a tag or version constraint
- Ship `examples/minimal` and `examples/complete` and keep them working
