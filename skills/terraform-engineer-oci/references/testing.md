# Testing OCI Terraform

## Validation Layers

Order matters — each layer is cheaper than the next and catches different failures.

| Layer | Cost | Catches |
|-------|------|---------|
| `fmt` / `validate` | Instant | Syntax, type errors, missing required arguments |
| `tflint` | Seconds | Provider-specific misuse, deprecated arguments |
| Policy as code | Seconds | Governance violations in the plan |
| `terraform test` (plan mode) | Seconds | Logic errors in locals, conditionals, validation |
| `terraform test` (apply mode) | Minutes to hours | Real API behavior, resource interaction |
| Terratest | Minutes to hours | End-to-end behavior with live assertions |

Run the first four on every commit. Reserve the last two for pull requests to shared modules and scheduled runs.

## Static Checks

```bash
terraform fmt -recursive -check -diff
terraform init -backend=false
terraform validate
```

`-backend=false` matters in CI: it initializes providers without needing state credentials, so validation runs on a pull request from a fork.

## TFLint

TFLint has no OCI ruleset comparable to the AWS one, so it catches Terraform-language issues rather than OCI-specific misuse. Still worth running.

```hcl
# .tflint.hcl
plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

rule "terraform_required_version" {
  enabled = true
}

rule "terraform_required_providers" {
  enabled = true
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

rule "terraform_unused_declarations" {
  enabled = true
}

rule "terraform_typed_variables" {
  enabled = true
}
```

```bash
tflint --init
tflint --recursive
```

## terraform test

Native testing since Terraform 1.6. Plan-mode tests need no credentials and no infrastructure, which makes them the highest-value layer for module logic.

**`tests/vcn.tftest.hcl`**

```hcl
variables {
  compartment_ocid = "ocid1.compartment.oc1..aaaaaaaatest"
  name_prefix      = "test-vcn"
  vcn_cidr         = "10.0.0.0/16"
}

# Plan-only: validates logic without calling OCI
run "subnet_cidrs_are_derived_correctly" {
  command = plan

  variables {
    subnets = {
      public = { cidr_block = "10.0.0.0/24", is_public = true }
      app    = { cidr_block = "10.0.1.0/24" }
    }
  }

  assert {
    condition     = oci_core_vcn.this.display_name == "test-vcn-vcn"
    error_message = "VCN display name should be prefixed"
  }

  assert {
    condition     = oci_core_subnet.this["app"].prohibit_public_ip_on_vnic == true
    error_message = "Private subnets must prohibit public IPs"
  }

  assert {
    condition     = oci_core_subnet.this["public"].prohibit_public_ip_on_vnic == false
    error_message = "Public subnets must permit public IPs"
  }
}

run "nat_gateway_is_optional" {
  command = plan

  variables {
    enable_nat_gateway = false
  }

  assert {
    condition     = length(oci_core_nat_gateway.this) == 0
    error_message = "NAT gateway should not be created when disabled"
  }
}

run "internet_gateway_only_with_public_subnet" {
  command = plan

  variables {
    subnets = {
      app = { cidr_block = "10.0.1.0/24" }
    }
  }

  assert {
    condition     = length(oci_core_internet_gateway.this) == 0
    error_message = "Internet gateway should not exist without a public subnet"
  }
}
```

### Testing Variable Validation

`expect_failures` verifies that bad input is rejected — worth having for every validation block you write.

```hcl
run "rejects_invalid_compartment_ocid" {
  command = plan

  variables {
    compartment_ocid = "not-an-ocid"
  }

  expect_failures = [var.compartment_ocid]
}

run "rejects_oversized_vcn_cidr" {
  command = plan

  variables {
    vcn_cidr = "10.0.0.0/28"
  }

  expect_failures = [var.vcn_cidr]
}

run "rejects_uppercase_dns_label" {
  command = plan

  variables {
    dns_label = "MyVCN"
  }

  expect_failures = [var.dns_label]
}
```

### Apply-Mode Integration Tests

These create real resources and cost real money. Scope them to a disposable compartment with a quota.

```hcl
# tests/integration.tftest.hcl
variables {
  compartment_ocid = "ocid1.compartment.oc1..test-sandbox"
  name_prefix      = "tftest"
  vcn_cidr         = "10.99.0.0/16"
}

run "create_network" {
  command = apply

  variables {
    subnets = {
      app = { cidr_block = "10.99.1.0/24" }
    }
    enable_service_gateway = true
  }

  assert {
    condition     = oci_core_vcn.this.state == "AVAILABLE"
    error_message = "VCN should reach AVAILABLE"
  }

  assert {
    condition     = length(oci_core_service_gateway.this) == 1
    error_message = "Service gateway should be created"
  }
}

run "verify_subnet_reachability" {
  command = apply

  module {
    source = "./tests/fixtures/instance"
  }

  variables {
    subnet_id = run.create_network.subnet_ids["app"]
  }

  assert {
    condition     = oci_core_instance.probe.state == "RUNNING"
    error_message = "Probe instance should launch in the created subnet"
  }
}
```

```bash
terraform test                          # all tests
terraform test -filter=tests/vcn.tftest.hcl
terraform test -verbose
```

Terraform destroys apply-mode resources automatically at the end of the run, including on failure. Because OCI compartment deletion is slow and asynchronous, use a long-lived sandbox compartment rather than creating one per test run.

## Terratest

Use Terratest when assertions need the OCI SDK — checking a work request completed, that an ADB accepts connections, or that a load balancer serves traffic.

```go
package test

import (
	"context"
	"fmt"
	"os"
	"testing"
	"time"

	"github.com/gruntwork-io/terratest/modules/random"
	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/oracle/oci-go-sdk/v65/common"
	"github.com/oracle/oci-go-sdk/v65/core"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestVcnModule(t *testing.T) {
	t.Parallel()

	compartmentID := mustEnv(t, "OCI_COMPARTMENT_OCID")
	prefix := fmt.Sprintf("tftest-%s", random.UniqueId())

	opts := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../examples/complete",
		Vars: map[string]interface{}{
			"compartment_ocid": compartmentID,
			"name_prefix":      prefix,
			"vcn_cidr":         "10.98.0.0/16",
		},
		// OCI work requests intermittently time out under load
		MaxRetries:         3,
		TimeBetweenRetries: 30 * time.Second,
	})

	defer terraform.Destroy(t, opts)
	terraform.InitAndApply(t, opts)

	vcnID := terraform.Output(t, opts, "vcn_id")
	require.NotEmpty(t, vcnID)

	provider := common.DefaultConfigProvider()
	client, err := core.NewVirtualNetworkClientWithConfigurationProvider(provider)
	require.NoError(t, err)

	resp, err := client.GetVcn(context.Background(), core.GetVcnRequest{VcnId: &vcnID})
	require.NoError(t, err)

	assert.Equal(t, core.VcnLifecycleStateAvailable, resp.Vcn.LifecycleState)
	assert.Equal(t, fmt.Sprintf("%s-vcn", prefix), *resp.Vcn.DisplayName)

	subnetIDs := terraform.OutputMap(t, opts, "subnet_ids")
	assert.Contains(t, subnetIDs, "app")

	subnetResp, err := client.GetSubnet(context.Background(), core.GetSubnetRequest{
		SubnetId: common.String(subnetIDs["app"]),
	})
	require.NoError(t, err)
	assert.True(t, *subnetResp.Subnet.ProhibitPublicIpOnVnic,
		"private subnet must prohibit public IPs")
}

func mustEnv(t *testing.T, key string) string {
	t.Helper()
	v := os.Getenv(key)
	require.NotEmpty(t, v, "%s must be set", key)
	return v
}
```

`common.DefaultConfigProvider()` reads `~/.oci/config` and the standard `OCI_*` environment variables, so tests authenticate the same way the provider does. In CI running on OCI compute, swap it for `auth.InstancePrincipalConfigurationProvider()`.

## Policy as Code

Conftest and OPA evaluate the plan JSON against rules. This is where OCI-specific governance belongs, since no linter enforces it.

```bash
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json
conftest test --policy policy/ tfplan.json
```

**`policy/oci.rego`**

```rego
package main

import future.keywords.in

resources[r] {
	r := input.resource_changes[_]
	r.mode == "managed"
	r.change.actions[_] in ["create", "update"]
}

# Every compartment-scoped resource must carry cost-tracking tags
deny[msg] {
	r := resources[_]
	r.change.after.compartment_id
	not r.change.after.defined_tags["Operations.CostCenter"]
	msg := sprintf("%s is missing Operations.CostCenter defined tag", [r.address])
}

# Private subnets must prohibit public IPs
deny[msg] {
	r := resources[_]
	r.type == "oci_core_subnet"
	r.change.after.prohibit_public_ip_on_vnic == false
	not startswith(r.change.after.display_name, "public")
	msg := sprintf("%s permits public IPs but is not named as a public subnet", [r.address])
}

# No unrestricted SSH from the internet
deny[msg] {
	r := resources[_]
	r.type == "oci_core_network_security_group_security_rule"
	r.change.after.direction == "INGRESS"
	r.change.after.source == "0.0.0.0/0"
	r.change.after.tcp_options[_].destination_port_range[_].min <= 22
	r.change.after.tcp_options[_].destination_port_range[_].max >= 22
	msg := sprintf("%s exposes SSH to the internet", [r.address])
}

# Object storage buckets must not be public
deny[msg] {
	r := resources[_]
	r.type == "oci_objectstorage_bucket"
	r.change.after.access_type != "NoPublicAccess"
	msg := sprintf("%s allows public access", [r.address])
}

# Block volumes and buckets must use a customer-managed key
deny[msg] {
	r := resources[_]
	r.type in ["oci_objectstorage_bucket", "oci_core_volume"]
	not r.change.after.kms_key_id
	msg := sprintf("%s is not encrypted with a customer-managed key", [r.address])
}

# Nothing may be created directly in the tenancy root
deny[msg] {
	r := resources[_]
	r.change.after.compartment_id
	startswith(r.change.after.compartment_id, "ocid1.tenancy.")
	r.type != "oci_identity_compartment"
	msg := sprintf("%s would be created in the tenancy root compartment", [r.address])
}
```

## Secret and Misconfiguration Scanning

Checkov and Trivy both ship OCI rules and read Terraform directly.

```bash
checkov -d . --framework terraform --compact
trivy config --severity HIGH,CRITICAL .
```

Both catch unencrypted volumes, public buckets, and permissive NSG rules without you writing policy. Use them alongside Conftest, not instead of it — they will not know your tagging conventions.

## Pre-Commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.1
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
        args: ["--hook-config=--retry-once-with-cleanup=true"]
      - id: terraform_tflint
        args: ["--args=--config=__GIT_WORKING_DIR__/.tflint.hcl"]
      - id: terraform_docs
        args: ["--hook-config=--path-to-file=README.md"]
      - id: terraform_checkov
        args: ["--args=--quiet", "--args=--compact"]

  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.21.2
    hooks:
      - id: gitleaks
```

Gitleaks matters more on OCI than elsewhere: a leaked API signing key PEM or a Customer Secret Key is immediately usable, and neither is scoped by default.

## CI Pipeline

```yaml
name: Terraform OCI

on:
  pull_request:
    paths: ["**.tf", "**.tftest.hcl"]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.8

      - run: terraform fmt -recursive -check -diff

      - run: terraform init -backend=false
      - run: terraform validate

      - uses: terraform-linters/setup-tflint@v4
      - run: tflint --init && tflint --recursive

      # Plan-mode tests need no OCI credentials
      - run: terraform test -filter=tests/unit.tftest.hcl

  policy:
    runs-on: ubuntu-latest
    needs: validate
    env:
      TF_VAR_tenancy_ocid: ${{ secrets.OCI_TENANCY_OCID }}
      TF_VAR_user_ocid: ${{ secrets.OCI_USER_OCID }}
      TF_VAR_fingerprint: ${{ secrets.OCI_FINGERPRINT }}
      TF_VAR_region: ${{ vars.OCI_REGION }}
    steps:
      - uses: actions/checkout@v4

      - name: Write API signing key
        run: |
          install -m 600 /dev/null "$RUNNER_TEMP/oci_api_key.pem"
          echo "${{ secrets.OCI_PRIVATE_KEY }}" > "$RUNNER_TEMP/oci_api_key.pem"
          echo "TF_VAR_private_key_path=$RUNNER_TEMP/oci_api_key.pem" >> "$GITHUB_ENV"

      - uses: hashicorp/setup-terraform@v3

      - run: terraform init
      - run: terraform plan -out=tfplan
      - run: terraform show -json tfplan > tfplan.json

      - uses: open-policy-agent/conftest-action@v0.2.0
        with:
          files: tfplan.json
          policy: policy/
```

Split the jobs so the credential-free checks run on every pull request including forks, and only the plan needs secrets.

## Best Practices

- Write plan-mode tests first; they need no credentials and run in seconds
- Cover every `validation` block with a matching `expect_failures` test
- Run apply-mode tests in a dedicated sandbox compartment with a quota
- Assert on `state == "AVAILABLE"` rather than mere resource existence
- Retry Terratest operations — OCI work requests time out intermittently
- Encode tagging and compartment rules as Conftest policy; no linter knows them
- Run Checkov or Trivy for the generic misconfiguration classes
- Scan for secrets on every commit; OCI key material is immediately usable
- Keep `examples/` directories working and test against them
- Never point tests at a compartment that holds production resources
