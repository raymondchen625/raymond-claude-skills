# OCI Networking

## VCN Fundamentals

A VCN is OCI's virtual network. Differences from a VPC that change how you write Terraform:

- Subnets are **regional by default** — omit `availability_domain` and the subnet spans all ADs
- Traffic is filtered by **security lists** (subnet-scoped) and **network security groups** (VNIC-scoped), and both apply
- Internet, NAT, and service access are separate gateway resources
- The **service gateway** provides private access to OCI services without an internet path — there is no equivalent in the AWS "VPC endpoint per service" sense; one gateway covers a service bundle
- A VCN can carry multiple CIDR blocks via `cidr_blocks`

```hcl
resource "oci_core_vcn" "this" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = [var.vcn_cidr]
  display_name   = "${var.name_prefix}-vcn"
  dns_label      = var.dns_label # lowercase alphanumeric, max 15 chars

  defined_tags  = local.common_defined_tags
  freeform_tags = local.common_freeform_tags
}
```

`dns_label` enables internal DNS resolution as `<host>.<subnet-label>.<vcn-label>.oraclevcn.com`. It cannot be changed after creation, so set it deliberately or omit it entirely.

## Subnets

Prefer regional subnets. AD-specific subnets exist for legacy reasons and complicate failover for no benefit in most designs.

```hcl
# Private regional subnet
resource "oci_core_subnet" "private" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  cidr_block     = var.private_subnet_cidr
  display_name   = "${var.name_prefix}-private"
  dns_label      = "private"

  # Regional: no availability_domain set
  route_table_id    = oci_core_route_table.private.id
  security_list_ids = [oci_core_security_list.private.id]
  dhcp_options_id   = oci_core_vcn.this.default_dhcp_options_id

  # Hard guarantee that nothing in here gets a public IP
  prohibit_public_ip_on_vnic = true
  prohibit_internet_ingress  = true

  defined_tags = local.common_defined_tags
}

# Public regional subnet
resource "oci_core_subnet" "public" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  cidr_block     = var.public_subnet_cidr
  display_name   = "${var.name_prefix}-public"
  dns_label      = "public"

  route_table_id    = oci_core_route_table.public.id
  security_list_ids = [oci_core_security_list.public.id]

  prohibit_public_ip_on_vnic = false

  defined_tags = local.common_defined_tags
}
```

`prohibit_public_ip_on_vnic = true` on private subnets is a cheap, enforceable guarantee. It fails resource creation rather than silently exposing an instance.

## Gateways

### Internet Gateway

Bidirectional internet access for public subnets.

```hcl
resource "oci_core_internet_gateway" "this" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-igw"
  enabled        = true
}
```

### NAT Gateway

Outbound-only internet access for private subnets.

```hcl
resource "oci_core_nat_gateway" "this" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-natgw"

  # Set true to disable egress without destroying the gateway
  block_traffic = false
}
```

Unlike AWS, a NAT gateway is a single regional resource — no per-AZ deployment, no Elastic IP to manage, no per-AZ cost multiplication.

### Service Gateway

Private access to OCI services (Object Storage, ADB, monitoring, and others) without traversing the internet. This is the one most teams forget, and its absence forces unnecessary NAT traffic.

```hcl
data "oci_core_services" "all" {
  filter {
    name   = "name"
    values = ["All .* Services In Oracle Services Network"]
    regex  = true
  }
}

resource "oci_core_service_gateway" "this" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-sgw"

  services {
    service_id = data.oci_core_services.all.services[0].id
  }
}
```

Two service bundles exist: "All \<region\> Services In Oracle Services Network" (everything) and "OCI \<region\> Object Storage" (storage only). Pick the narrower one when only Object Storage access is needed.

### Dynamic Routing Gateway

On-premises connectivity (FastConnect, IPSec VPN) and VCN-to-VCN routing.

```hcl
resource "oci_core_drg" "this" {
  compartment_id = var.compartment_ocid
  display_name   = "${var.name_prefix}-drg"
}

resource "oci_core_drg_attachment" "vcn" {
  drg_id       = oci_core_drg.this.id
  display_name = "${var.name_prefix}-vcn-attachment"

  network_details {
    id   = oci_core_vcn.this.id
    type = "VCN"
  }
}
```

DRG v2 supports VCN-to-VCN routing directly, which makes local peering gateways unnecessary for hub-and-spoke topologies within a region.

## Route Tables

```hcl
resource "oci_core_route_table" "public" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-public-rt"

  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_internet_gateway.this.id
  }
}

resource "oci_core_route_table" "private" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-private-rt"

  # Internet-bound traffic via NAT
  route_rules {
    destination       = "0.0.0.0/0"
    destination_type  = "CIDR_BLOCK"
    network_entity_id = oci_core_nat_gateway.this.id
  }

  # OCI services via service gateway — keeps this traffic off the NAT
  route_rules {
    destination       = data.oci_core_services.all.services[0].cidr_block
    destination_type  = "SERVICE_CIDR_BLOCK"
    network_entity_id = oci_core_service_gateway.this.id
  }

  # On-prem via DRG
  dynamic "route_rules" {
    for_each = var.on_prem_cidrs
    content {
      destination       = route_rules.value
      destination_type  = "CIDR_BLOCK"
      network_entity_id = oci_core_drg.this.id
    }
  }
}
```

Note `destination_type = "SERVICE_CIDR_BLOCK"` for service gateway routes — using `CIDR_BLOCK` there is a common error that produces a confusing rejection.

## Network Security Groups vs Security Lists

Both filter traffic and both apply. Understanding which to use is the main OCI networking decision.

| | Security list | Network security group |
|---|---|---|
| Attaches to | Subnet | VNIC |
| Granularity | Everything in the subnet | Individual resources |
| Can reference | CIDRs, services | CIDRs, services, **other NSGs** |
| Max per resource | 5 per subnet | 5 NSGs per VNIC |

**Use NSGs for new work.** The ability to write a rule whose source is another NSG produces self-documenting, tier-aware rules that do not break when CIDRs change.

```hcl
resource "oci_core_network_security_group" "lb" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-lb-nsg"
}

resource "oci_core_network_security_group" "app" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-app-nsg"
}

resource "oci_core_network_security_group" "db" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-db-nsg"
}

# Internet -> load balancer, HTTPS only
resource "oci_core_network_security_group_security_rule" "lb_ingress_https" {
  network_security_group_id = oci_core_network_security_group.lb.id
  direction                 = "INGRESS"
  protocol                  = "6" # TCP
  source                    = "0.0.0.0/0"
  source_type               = "CIDR_BLOCK"
  description               = "HTTPS from internet"

  tcp_options {
    destination_port_range {
      min = 443
      max = 443
    }
  }
}

# Load balancer -> app, by NSG reference rather than CIDR
resource "oci_core_network_security_group_security_rule" "app_ingress_from_lb" {
  network_security_group_id = oci_core_network_security_group.app.id
  direction                 = "INGRESS"
  protocol                  = "6"
  source                    = oci_core_network_security_group.lb.id
  source_type               = "NETWORK_SECURITY_GROUP"
  description               = "App traffic from load balancer"

  tcp_options {
    destination_port_range {
      min = 8080
      max = 8080
    }
  }
}

# App -> database
resource "oci_core_network_security_group_security_rule" "db_ingress_from_app" {
  network_security_group_id = oci_core_network_security_group.db.id
  direction                 = "INGRESS"
  protocol                  = "6"
  source                    = oci_core_network_security_group.app.id
  source_type               = "NETWORK_SECURITY_GROUP"
  description               = "SQL*Net from application tier"

  tcp_options {
    destination_port_range {
      min = 1521
      max = 1521
    }
  }
}

# Explicit egress — required, OCI does not imply it
resource "oci_core_network_security_group_security_rule" "app_egress" {
  network_security_group_id = oci_core_network_security_group.app.id
  direction                 = "EGRESS"
  protocol                  = "all"
  destination               = "0.0.0.0/0"
  destination_type          = "CIDR_BLOCK"
  description               = "Allow all egress"
}
```

Attach NSGs at the resource:

```hcl
resource "oci_core_instance" "app" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
  shape               = "VM.Standard.E4.Flex"

  create_vnic_details {
    subnet_id                 = oci_core_subnet.private.id
    nsg_ids                   = [oci_core_network_security_group.app.id]
    assign_public_ip          = false
    hostname_label            = "app01"
  }

  shape_config {
    ocpus         = 2
    memory_in_gbs = 32
  }
}
```

Protocol numbers are IANA values as strings: `"1"` ICMP, `"6"` TCP, `"17"` UDP, `"58"` ICMPv6, `"all"` for everything.

### Minimal Security Lists

Where NSGs carry the real rules, keep security lists minimal rather than duplicating logic in both places. Managing the same traffic path in both is the most common source of "the rule exists but traffic is blocked" confusion.

```hcl
resource "oci_core_security_list" "private" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.this.id
  display_name   = "${var.name_prefix}-private-sl"

  egress_security_rules {
    destination      = "0.0.0.0/0"
    destination_type = "CIDR_BLOCK"
    protocol         = "all"
    description      = "Allow all egress; ingress governed by NSGs"
  }

  ingress_security_rules {
    source      = var.vcn_cidr
    source_type = "CIDR_BLOCK"
    protocol    = "all"
    description = "Intra-VCN traffic"
  }
}
```

## Fault Domains

Fault domains are anti-affinity groups within an availability domain — three per AD. Spreading instances across them protects against localized hardware failure and costs nothing.

```hcl
data "oci_identity_fault_domains" "this" {
  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[0]
}

resource "oci_core_instance" "app" {
  count = var.instance_count

  compartment_id      = var.compartment_ocid
  availability_domain = local.ad_names[count.index % length(local.ad_names)]
  fault_domain        = data.oci_identity_fault_domains.this.fault_domains[count.index % 3].name
  shape               = "VM.Standard.E4.Flex"
  display_name        = "${var.name_prefix}-app-${count.index + 1}"
}
```

Single-AD regions are common in OCI. Code that assumes three ADs breaks in them — always derive the AD list and modulo against its actual length rather than a literal `3`.

## Load Balancers

Two distinct services with separate resource types:

- **Load Balancer (LBaaS)** — layer 7, HTTP/HTTPS aware, shape-based bandwidth
- **Network Load Balancer (NLB)** — layer 4, higher throughput, preserves source IP

```hcl
resource "oci_load_balancer_load_balancer" "this" {
  compartment_id = var.compartment_ocid
  display_name   = "${var.name_prefix}-lb"
  subnet_ids     = [oci_core_subnet.public.id]
  is_private     = false

  network_security_group_ids = [oci_core_network_security_group.lb.id]

  shape = "flexible"
  shape_details {
    minimum_bandwidth_in_mbps = 10
    maximum_bandwidth_in_mbps = 100
  }

  defined_tags = local.common_defined_tags
}

resource "oci_load_balancer_backend_set" "app" {
  load_balancer_id = oci_load_balancer_load_balancer.this.id
  name             = "app-backend"
  policy           = "LEAST_CONNECTIONS"

  health_checker {
    protocol            = "HTTP"
    port                = 8080
    url_path            = "/healthz"
    interval_ms         = 10000
    timeout_in_millis   = 3000
    retries             = 3
    return_code         = 200
  }
}
```

Use `shape = "flexible"` for new load balancers. The fixed shapes (`10Mbps`, `100Mbps`, `400Mbps`, `8000Mbps`) are legacy and bill for capacity you may not use.

## Best Practices

- Default to regional subnets; use AD-specific ones only for a concrete reason
- Set `prohibit_public_ip_on_vnic = true` on every private subnet
- Add a service gateway and route OCI service traffic through it, not the NAT gateway
- Use NSGs for new rules and reference other NSGs rather than CIDRs
- Do not manage the same traffic path in both a security list and an NSG
- Write explicit egress rules; OCI does not infer them
- Spread instances across fault domains, deriving counts from data sources
- Never hardcode an AD name — the tenancy prefix makes it non-portable
- Use `destination_type = "SERVICE_CIDR_BLOCK"` for service gateway routes
- Set `dns_label` deliberately at creation; it is immutable
