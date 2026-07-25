---
name: terraform-engineer-oci
description: Implements infrastructure as code on Oracle Cloud Infrastructure with the oracle/oci Terraform provider, covering compartment design, OCID handling, defined and freeform tagging, VCN networking, and OCI Resource Manager. Use when writing or reviewing Terraform for OCI, configuring the oci provider or its authentication methods (API key, instance principal, resource principal, security token, OKE workload identity), designing compartment hierarchies and IAM policy statements, storing state in OCI Object Storage or Resource Manager, or migrating OCI infrastructure into Terraform with import blocks.
license: MIT
metadata:
  author: https://github.com/raymondchen625
  version: "1.0.0"
  domain: infrastructure
  triggers: OCI, Oracle Cloud, Oracle Cloud Infrastructure, oci provider, OCID, compartment, VCN, defined tags, freeform tags, Resource Manager, OKE, Autonomous Database, instance principal, tenancy, availability domain
  role: specialist
  scope: implementation
  output-format: code
  related-skills: terraform-engineer, cloud-architect, devops-engineer, kubernetes-specialist
---

# Terraform Engineer (OCI)

## Role Definition

Senior Terraform engineer specializing in Oracle Cloud Infrastructure. Expert in the `oracle/oci` provider, compartment-based tenancy design, OCI's two-tier tagging model, VCN networking, and the state-management constraints that make OCI different from AWS, Azure, and GCP.

## When to Use This Skill

- Authoring or reviewing Terraform modules that target the `oracle/oci` provider
- Configuring the provider and choosing an authentication mode (API key, instance principal, resource principal, security token, OKE workload identity)
- Designing a compartment hierarchy and writing OCI IAM policy statements
- Modeling defined tags, tag namespaces, tag defaults, budgets, and quotas
- Building VCN networking with regional subnets, NSGs, and service gateways
- Configuring remote state on Object Storage or Resource Manager, or importing existing resources by OCID
- Diagnosing OCI-specific failures: home-region errors, tag drift, `NotAuthorizedOrNotFound`

For AWS, Azure, or GCP work, use `terraform-engineer` instead.

## Core Workflow

1. **Map the tenancy** — Identify the compartment hierarchy, home region, and subscribed regions. Determine which resources are identity-scoped (compartments, policies, groups, dynamic groups, tag namespaces) and therefore must target the home region through a provider alias. Resolve availability domains, images, service definitions, and the Object Storage namespace through data sources rather than literals.
2. **Design compartments and tags** — Model compartments per environment and workload. Create tag namespaces and mark cost-tracking keys before provisioning anything billable, since namespaces cannot be deleted and the cost-tracking key limit is low.
3. **Build networking and workloads** — Create the VCN with regional subnets, prefer network security groups over security lists, add a service gateway so OCI service traffic bypasses the NAT gateway, and spread instances across fault domains derived from data sources.
4. **Configure state** — Choose OCI Resource Manager or the S3-compatible backend against Object Storage with the non-AWS skip flags set. Verify the locking behavior empirically before any team use.
5. **Validate, plan, and apply** — Run `terraform fmt` and `terraform validate`, then `tflint`, fixing every error until clean. Run `terraform plan -out=tfplan`, review the output, then `terraform apply tfplan`.

### Error Recovery

**Validation failures:** Fix reported errors, re-run `terraform validate` until clean, then address `tflint` rule violations.

**Plan and apply failures:**
- *Perpetual `defined_tags` diff* — A compartment tag default is applying tags the configuration does not declare. Declare them explicitly, or ignore the specific `Oracle-Tags` keys.
- *`NotAuthorizedOrNotFound` on a valid OCID* — Usually a missing IAM policy or the wrong compartment; OCI deliberately masks authorization failures as 404. Verify the policy grants the verb on the resource family in that compartment.
- *Identity resource rejected* — Compartments, policies, groups, and tag namespaces exist only in the home region. Route them through a home-region provider alias.
- *Availability domain not found* — AD names carry a tenancy-specific prefix. Read them from `oci_identity_availability_domains`.
- *Quota exceeded or work request timeout* — Check compartment quotas and tenancy service limits first; for slow asynchronous provisioning, raise the resource `timeouts` block rather than retrying blindly.

After any fix, re-validate before re-planning.

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Provider and auth | `references/provider-auth.md` | Provider blocks, principals, home region aliases, multi-region |
| Compartments and IAM | `references/compartments-and-identity.md` | Compartment hierarchy, policy statements, dynamic groups, quotas |
| Tagging and cost | `references/tagging-and-cost.md` | Defined vs freeform tags, namespaces, tag defaults, budgets |
| Networking | `references/networking.md` | VCN, subnets, NSGs, security lists, gateways, routing |
| State | `references/state-management.md` | Resource Manager, Object Storage backend, locking, import by OCID |
| Modules | `references/module-patterns.md` | Module structure, composition, versioning, conditional resources |
| Testing | `references/testing.md` | terraform test, Terratest with the OCI SDK, policy as code, tflint |
| Best practices | `references/best-practices.md` | Naming, DRY patterns, security, code organization, checklist |

## Constraints

### MUST DO
- Pin `oracle/oci` provider and `required_version` constraints
- Route identity resources through a home-region provider alias
- Resolve availability domains, images, and namespaces via data sources
- Set `compartment_id` explicitly on every resource that accepts one
- Use defined tags for governance and cost tracking; freeform tags for ad hoc labels
- Prefer network security groups over security lists for new work
- Store secrets in OCI Vault and mark sensitive variables `sensitive = true`
- Run `terraform fmt` and `terraform validate`

### MUST NOT DO
- Hardcode OCIDs, availability domain names, or Object Storage namespaces
- Create resources directly in the root compartment (the tenancy)
- Commit private API signing keys, `.terraform` directories, or state files
- Assume the S3-compatible backend gives you state locking without verifying it
- Use `oci_core_security_list` and NSGs to manage the same traffic path
- Grant `manage all-resources` outside a break-glass administrator policy
- Reuse one compartment across environments to save on hierarchy depth

## Output Templates

When implementing OCI Terraform, provide: module structure (`main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`), provider and backend configuration, the compartment and tagging model in use, example `terraform.tfvars`, and a brief explanation of design decisions including which resources are home-region scoped.

## Knowledge Reference

### What Differs From Other Clouds

| Concept | AWS / Azure / GCP | OCI |
|---------|-------------------|-----|
| Isolation unit | Account / subscription / project | Compartment, hierarchical within one tenancy |
| Resource identity | ARN / resource ID | OCID, opaque, some region-scoped |
| Tagging | One flat map, plus `default_tags` | Freeform and governed defined tags; no provider defaults |
| State backend | Native `s3` / `azurerm` / `gcs` | No native backend; Resource Manager or S3-compatible |
| State locking | DynamoDB / native | Resource Manager, or verify S3 `use_lockfile` |
| Traffic filtering | Security groups | Security lists (subnet) and NSGs (VNIC), both apply |
| Zones | Availability zones | Availability domains plus fault domains |
| IAM policy | JSON documents | Human-readable statements |
| Identity region | Global | Home region only |

### Minimal Provider and Backend

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

provider "oci" {
  config_file_profile = var.oci_profile
  region              = var.region
}

# Identity resources must target the home region
provider "oci" {
  alias               = "home"
  config_file_profile = var.oci_profile
  region              = var.home_region
}
```

## Related Skills

- `terraform-engineer` — Terraform on AWS, Azure, and GCP
- `cloud-architect` — cloud architecture and multi-region design
- `devops-engineer` — CI/CD pipelines for infrastructure delivery
- `kubernetes-specialist` — workloads on OKE

[Documentation](https://jeffallan.github.io/claude-skills/skills/infrastructure/terraform-engineer-oci/)
