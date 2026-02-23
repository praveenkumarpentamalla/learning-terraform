# Terraform Testing & Linting: Complete Guide — Beginner to Production-Ready

---

# 📘 BEGINNER LEVEL

## What Is Testing & Linting in Terraform?

**In simple terms:** Testing means automatically verifying that your Terraform code **does what you expect**. Linting means automatically checking that your code is **written correctly, consistently, and safely** — before you ever run it.

Think of it as two separate quality gates:

```
LINTING                          TESTING
"Is the code written well?"      "Does the code do the right thing?"
       ↓                                    ↓
Check formatting                 Run against real or mocked infra
Check syntax                     Assert outputs are correct
Check style rules                Assert resources are created
Check security problems          Assert config values are right
Check best practices             Assert security rules hold
       ↓                                    ↓
  Runs in seconds                 Runs in minutes/hours
  No infrastructure               Creates real or mock infra
  No cloud access needed          Cloud access may be needed
```

**Why do you need both?**

Without linting you ship badly written code that works but is unmaintainable, insecure, and inconsistent across teams. Without testing you ship well-written code that is subtly wrong — the wrong ports are open, the wrong instance sizes are used, encryption is off.

Together they form a **quality firewall** that catches problems before they reach production.

**Real-World Analogy:**

> Imagine hiring a building contractor.
>
> **Linting** is the **building inspector checking the blueprints** before construction starts — wrong measurements, missing fire exits, code violations caught on paper for free.
>
> **Testing** is **test-fitting the actual building components** — does the door fit the frame? Do the pipes connect? Does the electrical circuit actually work?
>
> You want both. A perfect blueprint with bad construction is a disaster. A well-built version of a bad blueprint is equally disastrous.

---

## The Terraform Quality Toolchain

```
┌─────────────────────────────────────────────────────────────────┐
│                  TERRAFORM QUALITY TOOLCHAIN                     │
├──────────────────┬──────────────────┬───────────────────────────┤
│  TOOL            │  TYPE            │  PURPOSE                   │
├──────────────────┼──────────────────┼───────────────────────────┤
│  terraform fmt   │  Formatter       │  Auto-format code          │
│  terraform validate │ Validator     │  Check syntax/schema        │
│  TFLint          │  Linter          │  Style + provider rules     │
│  Checkov         │  SAST Scanner    │  Security & compliance      │
│  tfsec           │  SAST Scanner    │  Security scanning          │
│  Trivy           │  SAST Scanner    │  Security (all-in-one)      │
│  terraform test  │  Unit/Int Tests  │  Native HCL testing         │
│  Terratest       │  Integration     │  Go-based infra testing     │
│  Infracost       │  Cost Analysis   │  Cost estimation/guardrails │
│  pre-commit      │  Git Hooks       │  Run all tools automatically│
└──────────────────┴──────────────────┴───────────────────────────┘
```

---

## Common Beginner Mistakes

```bash
# ❌ MISTAKE 1: Only running terraform validate (not enough!)
terraform validate   # Only checks syntax — misses security, style, logic

# ❌ MISTAKE 2: Running terraform apply without any checks
terraform apply      # Deploying untested, unreviewed code to production

# ❌ MISTAKE 3: Not formatting code (causes noisy diffs in PRs)
# Every engineer has different spacing/indentation — PRs show 100s of changes
# ✅ Always run: terraform fmt -recursive before committing

# ❌ MISTAKE 4: Running linters only locally, not in CI
# "Works on my machine" problem — CI should enforce the same rules

# ❌ MISTAKE 5: Writing tests after the code breaks
# ✅ Write tests alongside the module, not as an afterthought

# ❌ MISTAKE 6: Using terraform test for everything
# terraform test is great for module unit tests
# Full integration tests of real infra → use Terratest

# ❌ MISTAKE 7: Not pinning tool versions
tflint --version     # What version? Might differ across team members!
# ✅ Pin versions in CI and use .tflint.hcl to pin plugin versions
```

---

# 📗 INTERMEDIATE LEVEL

---

# TOOL 1: `terraform fmt` — Auto-Formatter

## How It Works

`terraform fmt` reads your HCL files and rewrites them according to Terraform's canonical style. It is **non-negotiable** in professional teams — all code must be formatted before merging.

```bash
# ============================================
# COMMANDS
# ============================================

# Format all .tf files in current directory (modifies files in place)
terraform fmt

# Format recursively (includes modules, subdirectories)
terraform fmt -recursive

# Check only — exit code 0=formatted, 1=needs formatting (perfect for CI)
terraform fmt -check
terraform fmt -check -recursive

# Show diff of what would change without actually changing
terraform fmt -diff
terraform fmt -diff -recursive

# Format a specific file
terraform fmt main.tf
```

```hcl
# ============================================
# BEFORE terraform fmt
# ============================================
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"
    instance_type="t2.micro"
  tags={Name="web-server",Environment="prod"}
  count=3
}

# ============================================
# AFTER terraform fmt
# ============================================
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  count         = 3

  tags = {
    Name        = "web-server"
    Environment = "prod"
  }
}
```

What `terraform fmt` fixes automatically:
- Aligns `=` signs in attribute blocks
- Normalises indentation to 2 spaces
- Adds/removes blank lines between blocks
- Sorts meta-arguments to the top
- Fixes string quoting style

---

# TOOL 2: `terraform validate` — Configuration Validator

```bash
# ============================================
# COMMANDS
# ============================================

# Validate the current directory
terraform validate

# Validate and output as JSON (for CI parsing)
terraform validate -json

# Validate a specific module directory
cd modules/networking && terraform validate
```

What `terraform validate` checks:
- HCL syntax errors
- Required arguments are present
- Argument types match the expected type
- References to variables, locals, resources are valid
- Module call arguments match module variable declarations

What it does **NOT** check:
- Whether resources will actually be created successfully
- Security misconfigurations
- Style and formatting
- Provider-specific validation (wrong AMI ID, invalid region, etc.)

```bash
# Example output for a broken file:
terraform validate

# ╷
# │ Error: Missing required argument
# │
# │   on main.tf line 3, in resource "aws_instance" "web":
# │    3: resource "aws_instance" "web" {
# │
# │ The argument "ami" is required, but no definition was found.
# ╵

# JSON output for CI:
terraform validate -json
# {
#   "valid": false,
#   "error_count": 1,
#   "warning_count": 0,
#   "diagnostics": [
#     {
#       "severity": "error",
#       "summary": "Missing required argument",
#       "detail": "...",
#       "range": { "filename": "main.tf", "start": {...}, "end": {...} }
#     }
#   ]
# }
```

---

# TOOL 3: TFLint — The Professional Linter

## What Is TFLint?

TFLint goes far beyond `terraform validate`. It catches provider-specific errors (wrong instance types, invalid region names, deprecated arguments) and enforces team coding standards through rules and plugins.

## Installation

```bash
# macOS
brew install tflint

# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

# Docker (no installation)
docker run --rm -v $(pwd):/data -t ghcr.io/terraform-linters/tflint

# Windows
choco install tflint
```

## Configuration File (`.tflint.hcl`)

```hcl
# .tflint.hcl — Place in project root

config {
  # Use module inspection (follows module calls)
  call_module_type = "local"

  # Force all rules to be explicitly enabled (strict mode)
  # force = false
}

# ============================================
# AWS PLUGIN — Provider-specific rules
# ============================================
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# ============================================
# BUILT-IN RULES
# ============================================

# Warn when terraform_required_version is missing
rule "terraform_required_version" {
  enabled = true
}

# Warn when required_providers is missing
rule "terraform_required_providers" {
  enabled = true
}

# Warn about deprecated interpolation syntax ${var.name} in simple cases
rule "terraform_deprecated_interpolation" {
  enabled = true
}

# Disallow // comments (use # instead)
rule "terraform_comment_syntax" {
  enabled = true
}

# All variables must have descriptions
rule "terraform_documented_variables" {
  enabled = true
}

# All outputs must have descriptions
rule "terraform_documented_outputs" {
  enabled = true
}

# Naming conventions
rule "terraform_naming_convention" {
  enabled = true

  variable {
    format = "snake_case"
  }

  output {
    format = "snake_case"
  }

  resource {
    format = "snake_case"
  }

  module {
    format = "snake_case"
  }
}

# Warn when variables don't have types
rule "terraform_typed_variables" {
  enabled = true
}

# Warn about unused declarations
rule "terraform_unused_declarations" {
  enabled = true
}

# Warn when module sources aren't pinned
rule "terraform_module_pinned_source" {
  enabled = true
  style   = "semver"   # Require semantic versioning: ?ref=v1.2.3
}

# ============================================
# AWS-SPECIFIC RULES (from plugin)
# ============================================

# Catch invalid AWS instance types (e.g. "t9.mega" doesn't exist!)
rule "aws_instance_invalid_type" {
  enabled = true
}

# Catch deprecated instance types
rule "aws_instance_previous_type" {
  enabled = true
}

# Warn about missing tags on resources
rule "aws_resource_missing_tags" {
  enabled = true
  tags    = ["Environment", "Project", "ManagedBy"]
}

# Invalid AMI IDs
rule "aws_instance_invalid_ami" {
  enabled = false   # Disabled — requires AWS API access to verify
}
```

## Running TFLint

```bash
# ============================================
# TFLINT COMMANDS
# ============================================

# Initialize (downloads plugins from .tflint.hcl)
tflint --init

# Run in current directory
tflint

# Run recursively on all modules
tflint --recursive

# Run on a specific directory
tflint --chdir=modules/networking

# Output as JSON (for CI integration)
tflint --format=json

# Output as SARIF (for GitHub Security tab)
tflint --format=sarif

# Show only errors (not warnings)
tflint --minimum-failure-severity=error

# Ignore a specific rule for one run
tflint --disable-rule=terraform_documented_variables


# ============================================
# EXAMPLE OUTPUT
# ============================================
# $ tflint
#
# 4 issue(s) found:
#
# Warning: Missing version constraint for provider "aws" (terraform_required_providers)
#
#   on main.tf line 1:
#    1: provider "aws" {
#
# Warning: variable "instance_type" is missing a type declaration (terraform_typed_variables)
#
#   on variables.tf line 3:
#    3: variable "instance_type" {
#
# Error: "t9.large" is an invalid value as instance_type (aws_instance_invalid_type)
#
#   on main.tf line 15:
#   15:   instance_type = "t9.large"
#
# Warning: resource "aws_instance" "web" is missing the following required tags: Environment (aws_resource_missing_tags)
```

---

# TOOL 4: Checkov — Security & Compliance Scanner

## What Is Checkov?

Checkov is a **Static Application Security Testing (SAST)** tool specifically for Infrastructure as Code. It has 1000+ built-in security checks covering AWS, Azure, GCP, and Kubernetes. It catches real security disasters like open S3 buckets, unencrypted databases, overly permissive IAM roles, and disabled logging.

## Installation

```bash
# pip
pip install checkov

# Homebrew
brew install checkov

# Docker (no installation)
docker run --rm -v $(pwd):/tf bridgecrew/checkov -d /tf
```

## Running Checkov

```bash
# ============================================
# BASIC COMMANDS
# ============================================

# Scan current directory
checkov -d .

# Scan a specific file
checkov -f main.tf

# Scan with a specific framework
checkov -d . --framework terraform

# Output formats
checkov -d . --output cli          # Default human-readable
checkov -d . --output json         # JSON (for CI parsing)
checkov -d . --output sarif        # SARIF (for GitHub Security tab)
checkov -d . --output junitxml     # JUnit XML (for test reporting)

# Only show failures (skip passed checks)
checkov -d . --compact

# Run only specific checks
checkov -d . --check CKV_AWS_8,CKV_AWS_135

# Skip specific checks
checkov -d . --skip-check CKV_AWS_272

# Skip entire files or paths
checkov -d . --skip-path modules/legacy

# Show which checks are available
checkov -l --framework terraform


# ============================================
# EXAMPLE OUTPUT
# ============================================
# Check: CKV_AWS_8: "Ensure all data stored in the Launch configuration EBS is securely encrypted"
# PASSED for resource: aws_launch_template.app (main.tf:45)
#
# Check: CKV_AWS_135: "Ensure that EC2 instance should disable IMDS service or require IMDSv2"
# FAILED for resource: aws_instance.web (main.tf:12)
#   Guide: https://docs.bridgecrew.io/docs/bc_aws_general_31
#
# Check: CKV_AWS_23: "Ensure every security groups rule has a description"
# FAILED for resource: aws_security_group.web (main.tf:28)
#
# Passed checks: 15, Failed checks: 2, Skipped checks: 0
```

## Checkov Configuration File

```yaml
# .checkov.yaml — Project root

# Directories to scan
directory:
  - .

# Skip specific check IDs
skip-check:
  - CKV_AWS_272  # Legacy check not applicable
  - CKV2_AWS_12  # VPC flow logs — handled separately

# Only run checks from specific frameworks
framework:
  - terraform
  - secrets

# Output format
output:
  - cli
  - json

# Fail build on HIGH severity only
soft-fail-on:
  - LOW
  - MEDIUM

# External checks directory
external-checks-dir:
  - ./custom-checks/

# Compact output
compact: true

# Download external modules and check them too
download-external-modules: true
```

## Writing Custom Checkov Checks

```python
# custom-checks/check_required_tags.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class CheckRequiredTags(BaseResourceCheck):
    def __init__(self):
        name = "Ensure required company tags are present"
        id   = "CKV_CUSTOM_001"
        supported_resources = [
            "aws_instance",
            "aws_s3_bucket",
            "aws_db_instance",
            "aws_lambda_function",
        ]
        categories = [CheckCategories.GENERAL_SECURITY]
        super().__init__(name=name, id=id,
                         categories=categories,
                         supported_resources=supported_resources)

    def scan_resource_conf(self, conf):
        required_tags = {"Environment", "Project", "ManagedBy", "Team"}
        tags = conf.get("tags", [{}])

        # tags can be a list in Terraform's parsed format
        if isinstance(tags, list):
            tags = tags[0] if tags else {}

        if not isinstance(tags, dict):
            return CheckResult.FAILED

        existing_tags = set(tags.keys())
        missing_tags  = required_tags - existing_tags

        if missing_tags:
            print(f"Missing required tags: {missing_tags}")
            return CheckResult.FAILED

        return CheckResult.PASSED

scanner = CheckRequiredTags()
```

```bash
# Run with custom checks
checkov -d . --external-checks-dir ./custom-checks/
```

---

# TOOL 5: tfsec / Trivy — Security Scanners

## tfsec

```bash
# Install
brew install tfsec
# or
docker run --rm -it -v "$(pwd):/src" aquasec/tfsec /src

# Basic scan
tfsec .

# With output format
tfsec . --format json
tfsec . --format sarif

# Filter by severity
tfsec . --minimum-severity HIGH

# Ignore specific checks
tfsec . --exclude aws-s3-enable-versioning

# Exclude paths
tfsec . --exclude-path modules/legacy
```

## Trivy (Recommended — replaces tfsec)

```bash
# Install
brew install trivy
# or
docker run --rm -v $(pwd):/src aquasec/trivy config /src

# Scan Terraform configs
trivy config .

# Scan with severity filter
trivy config . --severity HIGH,CRITICAL

# Output formats
trivy config . --format table    # Default
trivy config . --format json
trivy config . --format sarif
trivy config . --format template --template "@contrib/junit.tpl"

# Exit code control (for CI)
trivy config . --exit-code 1      # Exit 1 if any issues found
trivy config . --exit-code 0      # Always exit 0 (don't fail CI)

# Ignore specific rules
trivy config . --skip-checks AVD-AWS-0028
```

## Trivy Configuration File

```yaml
# .trivyignore.yaml — Project root

# Ignore specific vulnerability IDs
ignores:
  - id: AVD-AWS-0028
    paths:
      - "modules/legacy/main.tf"
    expires: "2024-12-31"
    statement: "Legacy module being decommissioned by end of year"

  - id: AVD-AWS-0086
    statement: "VPC flow logs managed by central logging team"
```

---

# TOOL 6: `terraform test` — Native Testing Framework

## Complete Reference

```
tests/
├── unit/
│   ├── basic.tftest.hcl         ← Plan-only tests with mocks (fast)
│   └── validation.tftest.hcl    ← Test variable validation rules
└── integration/
    └── full.tftest.hcl          ← Full apply tests (slow, costs money)
```

### Full Test File Structure

```hcl
# tests/unit/networking.tftest.hcl

# ============================================
# SECTION 1: Mock provider definitions
# (No real API calls — instant feedback)
# ============================================
mock_provider "aws" {
  alias = "us_east"

  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
      state = "available"
    }
  }

  mock_data "aws_caller_identity" {
    defaults = {
      account_id = "123456789012"
      arn        = "arn:aws:iam::123456789012:root"
      user_id    = "AIDAIOSFODNN7EXAMPLE"
    }
  }

  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-0mock1234567890abc"
      arn        = "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-0mock"
      cidr_block = "10.0.0.0/16"
      state      = "available"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-0mock1234567890abc"
      arn               = "arn:aws:ec2:us-east-1:123456789012:subnet/subnet-0mock"
      availability_zone = "us-east-1a"
    }
  }

  mock_resource "aws_internet_gateway" {
    defaults = {
      id  = "igw-0mock1234567890abc"
      arn = "arn:aws:ec2:us-east-1:123456789012:internet-gateway/igw-0mock"
    }
  }
}


# ============================================
# SECTION 2: Shared variable defaults
# (Applied to ALL run blocks unless overridden)
# ============================================
variables {
  project_name = "test-project"
  environment  = "dev"
  vpc_cidr     = "10.0.0.0/16"
  az_count     = 2
}


# ============================================
# SECTION 3: Test run blocks
# ============================================

# TEST 1: Verify basic plan succeeds
run "plan_succeeds_with_valid_inputs" {
  command = plan   # Don't create real resources

  assert {
    condition     = plan.resource_changes != null
    error_message = "Plan should not be null"
  }
}


# TEST 2: Verify output values
run "vpc_cidr_matches_variable" {
  command = plan

  assert {
    condition     = output.vpc_cidr == var.vpc_cidr
    error_message = "VPC CIDR output '${output.vpc_cidr}' should match var '${var.vpc_cidr}'"
  }
}


# TEST 3: Verify naming convention
run "resources_use_correct_name_prefix" {
  command = plan

  assert {
    condition     = startswith(output.vpc_name, "${var.project_name}-${var.environment}")
    error_message = "VPC name should start with '${var.project_name}-${var.environment}'"
  }
}


# TEST 4: Verify correct subnet count
run "creates_correct_number_of_subnets" {
  command = plan

  assert {
    condition     = output.public_subnet_count == var.az_count
    error_message = "Expected ${var.az_count} public subnets, got ${output.public_subnet_count}"
  }

  assert {
    condition     = output.private_subnet_count == var.az_count
    error_message = "Expected ${var.az_count} private subnets, got ${output.private_subnet_count}"
  }
}


# TEST 5: Test with DIFFERENT variables (overrides shared variables)
run "larger_deployment_creates_more_subnets" {
  command = plan

  variables {
    az_count = 3   # Override just this one
  }

  assert {
    condition     = output.public_subnet_count == 3
    error_message = "Expected 3 public subnets with az_count=3"
  }
}


# TEST 6: Test that VALIDATION RULES work correctly
run "rejects_invalid_environment" {
  command = plan

  variables {
    environment = "production"   # Invalid — should trigger validation
  }

  expect_failures = [var.environment]   # This run EXPECTS a failure
}

run "rejects_invalid_cidr" {
  command = plan

  variables {
    vpc_cidr = "not-a-cidr"   # Invalid CIDR — should fail validation
  }

  expect_failures = [var.vpc_cidr]
}

run "rejects_too_small_prefix" {
  command = plan

  variables {
    vpc_cidr = "10.0.0.0/8"   # Too large (prefix < 16)
  }

  expect_failures = [var.vpc_cidr]
}


# TEST 7: Test production-specific behavior
run "production_enables_multi_az" {
  command = plan

  variables {
    environment = "prod"
  }

  assert {
    condition     = output.multi_az_enabled == true
    error_message = "Multi-AZ must be enabled in production"
  }
}

run "dev_disables_multi_az" {
  command = plan

  variables {
    environment = "dev"
  }

  assert {
    condition     = output.multi_az_enabled == false
    error_message = "Multi-AZ should be disabled in dev to save costs"
  }
}
```

```hcl
# tests/integration/full_deploy.tftest.hcl
# ⚠️ This creates REAL infrastructure — costs money, takes minutes!

# No mock_provider here — uses real AWS credentials

variables {
  project_name = "tftest"
  environment  = "test"
  vpc_cidr     = "10.99.0.0/16"
  az_count     = 2
}

# TEST 1: Full apply — create everything
run "deploy_networking" {
  command = apply   # Creates real resources!

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID should not be empty after apply"
  }

  assert {
    condition     = can(regex("^vpc-", output.vpc_id))
    error_message = "VPC ID should start with 'vpc-'"
  }

  assert {
    condition     = output.public_subnet_count == 2
    error_message = "Should have created 2 public subnets"
  }
}

# TEST 2: Apply is idempotent (running again makes no changes)
run "second_apply_makes_no_changes" {
  command = plan   # Plan after apply — should show 0 changes

  assert {
    condition     = !plan.has_changes
    error_message = "Second plan should show no changes — infrastructure should be idempotent"
  }
}

# TEST 3: Modify configuration — verify update works
run "update_tags" {
  command = apply

  variables {
    project_name = "tftest-updated"
  }

  assert {
    condition     = startswith(output.vpc_name, "tftest-updated")
    error_message = "VPC name should update to reflect new project name"
  }
}
```

```bash
# ============================================
# RUNNING TERRAFORM TESTS
# ============================================

# Run all tests
terraform test

# Run only unit tests (mock provider)
terraform test tests/unit/

# Run specific test file
terraform test -filter=tests/unit/networking.tftest.hcl

# Run with verbose output (see all assert results)
terraform test -verbose

# Run and show JSON output
terraform test -json

# Run and keep infrastructure after test (debugging)
terraform test -no-cleanup

# Run in a specific directory
terraform test -chdir=modules/networking
```

---

# TOOL 7: Terratest — Go-Based Integration Testing

## What Is Terratest?

Terratest is a Go library from Gruntwork that lets you write **real integration tests** for your Terraform modules. It actually deploys infrastructure, runs assertions against it, and cleans up. It's the most powerful Terraform testing tool for production modules.

## Project Structure

```
modules/
└── web-server/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── test/
        ├── go.mod
        ├── go.sum
        └── web_server_test.go
```

## Installing Terratest

```bash
# In your test directory
cd modules/web-server/test

# Initialize Go module
go mod init github.com/your-org/infrastructure/test

# Add Terratest
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/http-helper
go get github.com/gruntwork-io/terratest/modules/ssh
go get github.com/gruntwork-io/terratest/modules/aws
go get github.com/stretchr/testify/assert

go mod tidy
```

## Writing Terratest Tests

```go
// test/web_server_test.go
package test

import (
    "fmt"
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/http-helper"
    "github.com/gruntwork-io/terratest/modules/ssh"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// ============================================
// TEST 1: Basic deployment test
// ============================================
func TestWebServerDeployment(t *testing.T) {
    t.Parallel()   // Run tests in parallel for speed

    // Configure Terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        // Path to the Terraform code to test
        TerraformDir: "../",

        // Variables to pass — use unique names to avoid conflicts
        Vars: map[string]interface{}{
            "project_name": fmt.Sprintf("test-%d", time.Now().Unix()),
            "environment":  "test",
            "instance_type": "t2.micro",
            "aws_region":   "us-east-1",
        },

        // Environment variables for AWS credentials
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "us-east-1",
        },

        // Retry logic for eventually-consistent AWS resources
        MaxRetries:         3,
        TimeBetweenRetries: 5 * time.Second,
    })

    // CRITICAL: Destroy everything at the end of the test
    // defer runs even if the test panics or fails
    defer terraform.Destroy(t, terraformOptions)

    // Deploy the infrastructure
    terraform.InitAndApply(t, terraformOptions)

    // ---- ASSERTIONS ----

    // 1. Check Terraform outputs
    instanceID := terraform.Output(t, terraformOptions, "instance_id")
    assert.NotEmpty(t, instanceID, "Instance ID should not be empty")

    publicIP := terraform.Output(t, terraformOptions, "public_ip")
    assert.NotEmpty(t, publicIP, "Public IP should not be empty")

    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.Regexp(t, "^vpc-", vpcID, "VPC ID should start with vpc-")

    // 2. Check the actual AWS resource via AWS SDK
    region := "us-east-1"
    instance := aws.GetEc2InstanceById(t, instanceID, region)

    assert.Equal(t, "running", aws.GetEc2InstanceState(t, instanceID, region))
    assert.Equal(t, "t2.micro", string(instance.InstanceType))

    // 3. Check tags on the resource
    tags := aws.GetTagsForEc2Instance(t, region, instanceID)
    assert.Equal(t, "test", tags["Environment"])
    assert.Equal(t, "terraform", tags["ManagedBy"])

    // 4. Check HTTP response from the web server
    url := fmt.Sprintf("http://%s:80", publicIP)
    http_helper.HttpGetWithRetry(
        t,
        url,
        nil,                // No TLS config
        200,                // Expected status code
        "Hello, World!",    // Expected body (partial match)
        30,                 // Max retries
        10 * time.Second,   // Sleep between retries
    )

    // 5. Check security group settings via AWS SDK
    sgID := terraform.Output(t, terraformOptions, "security_group_id")
    sg   := aws.GetSecurityGroupById(t, sgID, region)

    // Verify SSH is NOT open to the world
    for _, rule := range sg.IpPermissions {
        if *rule.FromPort == 22 {
            for _, ipRange := range rule.IpRanges {
                assert.NotEqual(t, "0.0.0.0/0", *ipRange.CidrIp,
                    "SSH should not be open to 0.0.0.0/0!")
            }
        }
    }
}


// ============================================
// TEST 2: Test module outputs directly (no infrastructure)
// ============================================
func TestWebServerPlanOutputs(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "project_name": "plan-test",
            "environment":  "dev",
        },
    }

    // Only init and plan — don't apply (much faster!)
    terraform.Init(t, terraformOptions)
    planOutput := terraform.InitAndPlan(t, terraformOptions)

    // Assert plan contains expected resources
    resourceCount := terraform.GetResourceCount(t, planOutput)
    assert.Equal(t, 5, resourceCount.Add,    "Should plan to add 5 resources")
    assert.Equal(t, 0, resourceCount.Change, "Should plan 0 changes")
    assert.Equal(t, 0, resourceCount.Destroy,"Should plan 0 destroys")
}


// ============================================
// TEST 3: Test idempotency
// ============================================
func TestWebServerIdempotency(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "project_name": fmt.Sprintf("idempotency-test-%d", time.Now().Unix()),
            "environment":  "test",
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    // First apply
    terraform.InitAndApply(t, terraformOptions)

    // Second apply — should make zero changes
    terraform.ApplyAndIdempotent(t, terraformOptions)
    // This method fails the test if the second apply makes any changes!
}


// ============================================
// TEST 4: SSH into the instance and run commands
// ============================================
func TestWebServerConfiguration(t *testing.T) {
    t.Parallel()

    keyPairName := fmt.Sprintf("test-key-%d", time.Now().Unix())
    region      := "us-east-1"

    // Create a temporary SSH key pair
    keyPair := aws.CreateAndImportEC2KeyPair(t, region, keyPairName)
    defer aws.DeleteEC2KeyPair(t, keyPair)

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "project_name": fmt.Sprintf("ssh-test-%d", time.Now().Unix()),
            "environment":  "test",
            "key_pair_name": keyPairName,
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    publicIP := terraform.Output(t, terraformOptions, "public_ip")

    // SSH into the instance
    sshHost := ssh.Host{
        Hostname:    publicIP,
        SshUserName: "ubuntu",
        SshKeyPair:  keyPair.KeyPair,
    }

    // Wait for SSH to become available
    ssh.CheckSshConnectionWithRetry(t, sshHost, 30, 5 * time.Second)

    // Run commands on the instance
    nginx_status, err := ssh.CheckSshCommandE(t, sshHost, "systemctl is-active nginx")
    require.NoError(t, err)
    assert.Equal(t, "active", strings.TrimSpace(nginx_status))

    node_version, _ := ssh.CheckSshCommandE(t, sshHost, "node --version")
    assert.Regexp(t, `^v18\.`, strings.TrimSpace(node_version))
}


// ============================================
// TEST 5: Table-driven tests (test many configurations)
// ============================================
func TestWebServerEnvironmentConfigs(t *testing.T) {
    t.Parallel()

    testCases := []struct {
        name             string
        environment      string
        expectedType     string
        expectedMinSize  int
    }{
        {"dev configuration",     "dev",     "t2.micro",  1},
        {"staging configuration", "staging", "t2.small",  2},
        {"prod configuration",    "prod",    "t3.medium", 3},
    }

    for _, tc := range testCases {
        tc := tc   // Capture range variable for parallel tests

        t.Run(tc.name, func(t *testing.T) {
            t.Parallel()

            terraformOptions := &terraform.Options{
                TerraformDir: "../",
                Vars: map[string]interface{}{
                    "project_name": fmt.Sprintf("env-test-%s", tc.environment),
                    "environment":  tc.environment,
                },
            }

            terraform.Init(t, terraformOptions)
            planOutput := terraform.InitAndPlan(t, terraformOptions)

            // Parse plan to get planned values
            // (In real test would check instance_type from plan output)
            assert.NotEmpty(t, planOutput)
        })
    }
}
```

```bash
# ============================================
# RUNNING TERRATEST
# ============================================

# Run all tests (full integration — creates real AWS resources)
cd modules/web-server/test
go test -v -timeout 30m ./...

# Run a specific test
go test -v -timeout 30m -run TestWebServerDeployment ./...

# Run in parallel (multiple tests at once)
go test -v -timeout 60m -parallel 4 ./...

# Run with verbose logging
TF_LOG=INFO go test -v -timeout 30m ./...

# Run plan-only tests (no infrastructure, instant)
go test -v -run TestWebServerPlanOutputs ./...

# Skip cleanup (keep infrastructure for debugging)
SKIP_DESTROY=true go test -v -timeout 30m ./...
```

---

# TOOL 8: Infracost — Cost Estimation & Guardrails

## What Is Infracost?

Infracost shows you the **cost impact of your Terraform changes** before you apply them. Critical for preventing accidentally expensive deployments.

```bash
# Install
brew install infracost

# Setup (one-time — get free API key)
infracost auth login

# ============================================
# BASIC COMMANDS
# ============================================

# Show cost breakdown of current configuration
infracost breakdown --path .

# Show cost DIFF between current and planned change
infracost diff --path . --compare-to infracost-base.json

# Output formats
infracost breakdown --path . --format table    # Default
infracost breakdown --path . --format json     # For CI
infracost breakdown --path . --format html     # HTML report

# Scan all Terraform projects in a directory
infracost breakdown --path . --terraform-var-file=environments/prod.tfvars


# ============================================
# EXAMPLE OUTPUT
# ============================================
# Project: myapp (production)
#
# Name                           Monthly Qty  Unit       Monthly Cost
#
# aws_instance.web_servers[0]
# ├─ Instance usage (Linux/UNIX)         730  hours            $8.47
# ├─ EC2 detailed monitoring               1  months           $2.10
# └─ root_block_device
#    └─ Storage (general purpose SSD)    30  GB               $3.00
#
# aws_db_instance.main
# ├─ Database instance (db.r5.large)    730  hours           $175.20
# ├─ Storage (general purpose SSD)      500  GB              $50.00
# └─ Backup storage                     500  GB              $10.00
#
# OVERALL TOTAL                                              $248.77
```

## Infracost in CI/CD (Cost Policy)

```bash
# infracost-ci.sh — Run in CI to block expensive PRs

# Generate cost for current branch
infracost breakdown \
  --path . \
  --format json \
  --out-file infracost-current.json

# Generate cost for base branch
git stash
infracost breakdown \
  --path . \
  --format json \
  --out-file infracost-base.json
git stash pop

# Show the diff
infracost diff \
  --path infracost-current.json \
  --compare-to infracost-base.json

# Check if monthly cost increase exceeds threshold ($100)
COST_INCREASE=$(infracost diff \
  --path infracost-current.json \
  --compare-to infracost-base.json \
  --format json | jq '.diffTotalMonthlyCost | tonumber')

if (( $(echo "$COST_INCREASE > 100" | bc -l) )); then
  echo "❌ Cost increase ($${COST_INCREASE}/month) exceeds $100 threshold!"
  echo "Please review and get approval for expensive changes."
  exit 1
fi

echo "✅ Cost increase within acceptable range: $${COST_INCREASE}/month"
```

---

# TOOL 9: pre-commit — Automated Local Enforcement

## What Is pre-commit?

pre-commit runs a suite of tools **automatically every time you `git commit`**, catching issues before they ever leave your local machine.

## Installation & Configuration

```bash
# Install pre-commit
pip install pre-commit
# or
brew install pre-commit

# Install hooks into the repo
pre-commit install

# Run manually against all files
pre-commit run --all-files

# Run a specific hook
pre-commit run terraform-fmt
```

```yaml
# .pre-commit-config.yaml — Project root

repos:
  # ============================================
  # General file checks
  # ============================================
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: detect-private-key        # Never commit private keys!
      - id: check-merge-conflict

  # ============================================
  # Terraform hooks
  # ============================================
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.86.0
    hooks:
      # Format all .tf files
      - id: terraform_fmt
        args:
          - --args=-recursive

      # Validate all modules
      - id: terraform_validate
        args:
          - --init-args=-upgrade

      # Run TFLint
      - id: terraform_tflint
        args:
          - --args=--recursive
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl

      # Run Checkov security scan
      - id: terraform_checkov
        args:
          - --args=--config-file __GIT_WORKING_DIR__/.checkov.yaml
          - --args=--compact

      # Run Trivy security scan
      - id: terraform_trivy
        args:
          - --args=--severity=HIGH,CRITICAL

      # Generate module documentation (terraform-docs)
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
          - --hook-config=--create-file-if-not-exist=true

  # ============================================
  # Secret detection
  # ============================================
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        exclude: "terraform.tfstate"
```

---

# 📕 ADVANCED LEVEL

## Deep Dive: Building a Complete CI/CD Quality Pipeline

The following is a **production-grade GitHub Actions pipeline** that enforces the full quality toolchain on every PR and merge.

```yaml
# .github/workflows/terraform-quality.yml

name: Terraform Quality Pipeline

on:
  pull_request:
    branches: [main]
    paths:
      - "**.tf"
      - "**.tfvars"
      - ".tflint.hcl"
      - ".checkov.yaml"
      - ".github/workflows/terraform-quality.yml"
  push:
    branches: [main]
    paths:
      - "**.tf"

env:
  TF_VERSION:       "1.7.0"
  TFLINT_VERSION:   "0.50.0"
  CHECKOV_VERSION:  "3.1.0"
  TRIVY_VERSION:    "0.48.0"
  INFRACOST_VERSION: "0.10.29"

jobs:
  # ============================================
  # JOB 1: Format Check (fastest — runs first)
  # ============================================
  format:
    name: Terraform Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Check Formatting
        id: fmt
        run: terraform fmt -check -recursive -diff
        continue-on-error: false

      - name: Comment PR on Failure
        if: failure() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ **Terraform Format Failed**\n\nRun `terraform fmt -recursive` locally and commit the changes.'
            })


  # ============================================
  # JOB 2: Validate all modules
  # ============================================
  validate:
    name: Terraform Validate
    runs-on: ubuntu-latest
    needs: format
    strategy:
      matrix:
        module:
          - "."
          - "modules/networking"
          - "modules/compute"
          - "modules/database"
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init -backend=false
        working-directory: ${{ matrix.module }}

      - name: Terraform Validate
        run: terraform validate
        working-directory: ${{ matrix.module }}


  # ============================================
  # JOB 3: TFLint
  # ============================================
  tflint:
    name: TFLint
    runs-on: ubuntu-latest
    needs: format
    steps:
      - uses: actions/checkout@v4

      - name: Setup TFLint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: v${{ env.TFLINT_VERSION }}

      - name: Cache TFLint plugins
        uses: actions/cache@v4
        with:
          path: ~/.tflint.d/plugins
          key: ${{ runner.os }}-tflint-${{ hashFiles('.tflint.hcl') }}

      - name: Init TFLint
        run: tflint --init

      - name: Run TFLint
        run: tflint --recursive --format=sarif > tflint-results.sarif
        continue-on-error: true

      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: tflint-results.sarif
          category: tflint

      - name: Fail on Errors
        run: tflint --recursive --minimum-failure-severity=error


  # ============================================
  # JOB 4: Security Scanning (parallel)
  # ============================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: validate
    strategy:
      matrix:
        tool: [checkov, trivy]
      fail-fast: false

    steps:
      - uses: actions/checkout@v4

      # -- Checkov --
      - name: Run Checkov
        if: matrix.tool == 'checkov'
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: .
          framework: terraform
          output_format: sarif
          output_file_path: checkov-results.sarif
          soft_fail: false
          skip_check: CKV_AWS_272

      - name: Upload Checkov SARIF
        if: matrix.tool == 'checkov' && always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: checkov-results.sarif
          category: checkov

      # -- Trivy --
      - name: Run Trivy
        if: matrix.tool == 'trivy'
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: .
          severity: HIGH,CRITICAL
          format: sarif
          output: trivy-results.sarif
          exit-code: "1"

      - name: Upload Trivy SARIF
        if: matrix.tool == 'trivy' && always()
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
          category: trivy


  # ============================================
  # JOB 5: Cost Estimation
  # ============================================
  cost:
    name: Cost Estimation
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Setup Infracost
        uses: infracost/actions/setup@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Generate cost on base branch
        run: |
          git checkout ${{ github.base_ref }}
          infracost breakdown \
            --path . \
            --format json \
            --out-file /tmp/infracost-base.json

      - name: Generate cost on PR branch
        run: |
          git checkout ${{ github.head_ref }}
          infracost breakdown \
            --path . \
            --format json \
            --out-file /tmp/infracost-pr.json

      - name: Post cost diff as PR comment
        run: |
          infracost diff \
            --path /tmp/infracost-pr.json \
            --compare-to /tmp/infracost-base.json \
            --format github-comment \
            --out-file /tmp/cost-comment.md
          cat /tmp/cost-comment.md | gh pr comment ${{ github.event.pull_request.number }} --body-file -
        env:
          GH_TOKEN: ${{ github.token }}


  # ============================================
  # JOB 6: Native Terraform Tests
  # ============================================
  test:
    name: Terraform Tests
    runs-on: ubuntu-latest
    needs: [validate, security]
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Run Unit Tests (mock providers)
        run: terraform test tests/unit/
        working-directory: modules/networking

      - name: Run Unit Tests (all modules)
        run: |
          for module in modules/*/; do
            if [ -d "${module}tests" ]; then
              echo "Testing ${module}..."
              terraform -chdir="${module}" test tests/unit/ -verbose
            fi
          done


  # ============================================
  # JOB 7: Integration Tests (on merge to main only)
  # ============================================
  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: testing   # GitHub environment with AWS credentials
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TEST_AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Run Integration Tests
        run: terraform test tests/integration/
        working-directory: modules/networking
        timeout-minutes: 30


  # ============================================
  # JOB 8: Terratest (only for critical modules)
  # ============================================
  terratest:
    name: Terratest
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: testing
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.TEST_AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Run Terratest
        run: go test -v -timeout 45m ./...
        working-directory: modules/web-server/test
        env:
          AWS_DEFAULT_REGION: us-east-1


  # ============================================
  # JOB 9: Summary Gate (all must pass)
  # ============================================
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    needs: [format, validate, tflint, security, cost, test]
    if: always()
    steps:
      - name: Check all jobs passed
        run: |
          if [[ "${{ needs.format.result }}"   != "success" ]] ||
             [[ "${{ needs.validate.result }}" != "success" ]] ||
             [[ "${{ needs.tflint.result }}"   != "success" ]] ||
             [[ "${{ needs.security.result }}" != "success" ]]; then
            echo "❌ Quality gate failed — one or more checks did not pass"
            exit 1
          fi
          echo "✅ All quality checks passed!"
```

---

## Advanced: Testing Module Contracts

```hcl
# ============================================
# Testing that modules expose the right interface
# ============================================

# tests/unit/module_contract.tftest.hcl

mock_provider "aws" {
  mock_resource "aws_vpc"            { defaults = { id = "vpc-mock" } }
  mock_resource "aws_subnet"         { defaults = { id = "subnet-mock" } }
  mock_resource "aws_security_group" { defaults = { id = "sg-mock"  } }
}

variables {
  project_name = "contract-test"
  environment  = "test"
  vpc_cidr     = "10.0.0.0/16"
}

# Test: all required outputs are present and non-null
run "all_required_outputs_present" {
  command = plan

  assert {
    condition     = output.vpc_id != null
    error_message = "Module must output vpc_id"
  }

  assert {
    condition     = output.public_subnet_ids != null
    error_message = "Module must output public_subnet_ids"
  }

  assert {
    condition     = output.private_subnet_ids != null
    error_message = "Module must output private_subnet_ids"
  }

  assert {
    condition     = length(output.public_subnet_ids) > 0
    error_message = "public_subnet_ids must not be empty"
  }
}

# Test: outputs have correct types
run "output_types_are_correct" {
  command = plan

  assert {
    condition     = can(tostring(output.vpc_id))
    error_message = "vpc_id must be a string"
  }

  assert {
    condition     = can(tolist(output.public_subnet_ids))
    error_message = "public_subnet_ids must be a list"
  }
}

# Test: environment-specific behaviour
run "production_outputs_reflect_ha_config" {
  command = plan

  variables {
    environment = "prod"
  }

  assert {
    condition     = output.multi_az_enabled == true
    error_message = "Production must enable multi-AZ"
  }

  assert {
    condition     = length(output.private_subnet_ids) >= 3
    error_message = "Production requires at least 3 private subnets"
  }
}
```

---

## Advanced: Snapshot Testing Pattern

```hcl
# ============================================
# Pattern: Golden file / snapshot testing
# Compare plan output against known-good baseline
# ============================================

# golden/networking-plan.json — committed to repo
# Generated by: terraform plan -json > tests/golden/networking-plan.json

# In CI, compare current plan to golden file:
# terraform plan -json > /tmp/current-plan.json
# diff tests/golden/networking-plan.json /tmp/current-plan.json
# If diff exists — review whether change is intentional

# Automate in shell:
#!/bin/bash
# scripts/golden-test.sh

MODULE_PATH="${1:-.}"
GOLDEN_DIR="tests/golden"
PLAN_FILE="/tmp/current-plan-$(basename $MODULE_PATH).json"

# Generate current plan
terraform -chdir="$MODULE_PATH" init -backend=false
terraform -chdir="$MODULE_PATH" plan -json > "$PLAN_FILE" 2>/dev/null

# Extract just the resource_changes (ignore metadata like timestamps)
jq '[.resource_changes[] | {
  address: .address,
  type: .type,
  change: {actions: .change.actions}
}] | sort_by(.address)' "$PLAN_FILE" > /tmp/normalized-plan.json

GOLDEN_FILE="$GOLDEN_DIR/$(basename $MODULE_PATH)-resources.json"

if [ ! -f "$GOLDEN_FILE" ]; then
  echo "No golden file found. Creating: $GOLDEN_FILE"
  cp /tmp/normalized-plan.json "$GOLDEN_FILE"
  echo "✅ Golden file created. Commit this file."
  exit 0
fi

if diff "$GOLDEN_FILE" /tmp/normalized-plan.json > /dev/null; then
  echo "✅ Plan matches golden file — no unexpected changes"
else
  echo "❌ Plan differs from golden file!"
  echo "Changed resources:"
  diff "$GOLDEN_FILE" /tmp/normalized-plan.json
  echo ""
  echo "If this change is intentional, update the golden file:"
  echo "  cp /tmp/normalized-plan.json $GOLDEN_FILE"
  exit 1
fi
```

---

## Performance Considerations for Testing

```bash
# ============================================
# MAKING TESTS FAST
# ============================================

# 1. Use mock providers for unit tests (instant vs minutes)
#    terraform test with mock_provider → ~1 second
#    terraform test with real AWS     → ~5 minutes

# 2. Run unit tests in pre-commit, integration in CI
#    Pre-commit: fmt, validate, tflint, unit tests
#    CI on PR:   all of above + security + cost
#    CI on merge: integration tests + terratest

# 3. Parallelize Go tests
go test -parallel 10 -timeout 60m ./...

# 4. Cache provider downloads in CI
- name: Cache Terraform providers
  uses: actions/cache@v4
  with:
    path: ~/.terraform.d/plugin-cache
    key: ${{ runner.os }}-terraform-${{ hashFiles('**/.terraform.lock.hcl') }}

# 5. Use -target in integration tests to test only changed modules
terraform test -filter=tests/integration/networking.tftest.hcl

# 6. Run security scans in parallel (checkov + trivy simultaneously)
# See the matrix strategy in the GitHub Actions example above

# 7. Skip integration tests on documentation-only PRs
if: "!contains(github.event.pull_request.labels.*.name, 'docs-only')"
```

---

# 🏗️ REAL-WORLD PRODUCTION ARCHITECTURE

## Complete: Quality Pipeline for a Terraform Monorepo

```
infrastructure/
├── .github/
│   └── workflows/
│       ├── terraform-quality.yml    ← Full quality pipeline
│       └── terraform-deploy.yml     ← Deployment pipeline
│
├── .pre-commit-config.yaml          ← Local quality enforcement
├── .tflint.hcl                      ← TFLint rules
├── .checkov.yaml                    ← Checkov config
├── .trivyignore.yaml                ← Trivy suppressions
├── .infracost.yml                   ← Cost config
│
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── README.md               ← Auto-generated by terraform-docs
│   │   └── tests/
│   │       ├── unit/
│   │       │   ├── basic.tftest.hcl
│   │       │   ├── validation.tftest.hcl
│   │       │   └── contract.tftest.hcl
│   │       └── integration/
│   │           └── full.tftest.hcl
│   ├── compute/
│   │   └── test/
│   │       ├── go.mod
│   │       └── compute_test.go      ← Terratest
│   └── database/
│
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── production/
│
├── scripts/
│   ├── lint.sh                     ← Run all linters locally
│   ├── test.sh                     ← Run all tests locally
│   └── golden-test.sh              ← Snapshot testing
│
└── custom-checks/
    └── check_required_tags.py      ← Custom Checkov checks
```

```bash
# scripts/lint.sh — Run all linters (same as CI)
#!/bin/bash
set -euo pipefail

echo "=== Terraform Format Check ==="
terraform fmt -check -recursive -diff

echo "=== Terraform Validate ==="
for dir in . modules/*/; do
  echo "  Validating: $dir"
  terraform -chdir="$dir" init -backend=false -input=false > /dev/null
  terraform -chdir="$dir" validate
done

echo "=== TFLint ==="
tflint --recursive

echo "=== Checkov ==="
checkov -d . --compact --framework terraform

echo "=== Trivy ==="
trivy config . --severity HIGH,CRITICAL

echo "=== All linting checks passed! ✅ ==="


# scripts/test.sh — Run all tests
#!/bin/bash
set -euo pipefail

echo "=== Unit Tests (mock providers — fast) ==="
for module in modules/*/; do
  if [ -d "${module}tests/unit" ]; then
    echo "  Testing: $module"
    terraform -chdir="$module" test tests/unit/ -verbose
  fi
done

echo "=== Infracost Estimate ==="
infracost breakdown --path . --format table

echo "=== All tests passed! ✅ ==="
```

---

# 🧪 HANDS-ON LAB

## Lab: Build a Fully Tested & Linted Module

**Objective:** Create a complete networking module with full quality toolchain — formatting, linting, security scanning, unit tests with mock providers, and validation tests.

**Requirements:**
1. Create a `modules/vpc/` with `main.tf`, `variables.tf`, `outputs.tf`
2. Configure `.tflint.hcl` with naming conventions and documentation rules
3. Write **4 unit test cases** with mock providers using `terraform test`
4. Write **2 validation failure tests** using `expect_failures`
5. Configure `pre-commit` to enforce quality automatically
6. All checks must pass: `terraform fmt`, `terraform validate`, `tflint`, `terraform test`

**Try it yourself before reading the solution!**

---

## Lab Solution

```hcl
# modules/vpc/variables.tf

variable "project_name" {
  type        = string
  description = "Project name used as prefix for all resources."

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,28}[a-z0-9]$", var.project_name))
    error_message = "project_name must be 3-30 chars, lowercase alphanumeric and hyphens."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC (e.g. '10.0.0.0/16')."
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid CIDR block."
  }

  validation {
    condition = (
      tonumber(split("/", var.vpc_cidr)[1]) >= 16 &&
      tonumber(split("/", var.vpc_cidr)[1]) <= 24
    )
    error_message = "vpc_cidr prefix must be between /16 and /24."
  }
}

variable "az_count" {
  type        = number
  description = "Number of availability zones to deploy across (2 or 3)."
  default     = 2

  validation {
    condition     = contains([2, 3], var.az_count)
    error_message = "az_count must be 2 or 3."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources."
  default     = {}
}
```

```hcl
# modules/vpc/main.tf

terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  name_prefix = "${var.project_name}-${var.environment}"
  az_names    = slice(data.aws_availability_zones.available.names, 0, var.az_count)

  public_cidrs  = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_cidrs = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]

  common_tags = merge(var.tags, {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_cidrs[count.index]
  availability_zone = local.az_names[count.index]

  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = local.az_names[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Tier = "private"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count          = var.az_count
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID of the created VPC."
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC."
  value       = aws_vpc.main.cidr_block
}

output "vpc_name" {
  description = "Name tag of the VPC."
  value       = aws_vpc.main.tags["Name"]
}

output "public_subnet_ids" {
  description = "List of public subnet IDs."
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs."
  value       = aws_subnet.private[*].id
}

output "public_subnet_count" {
  description = "Number of public subnets created."
  value       = length(aws_subnet.public)
}

output "private_subnet_count" {
  description = "Number of private subnets created."
  value       = length(aws_subnet.private)
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway."
  value       = aws_internet_gateway.main.id
}
```

```hcl
# modules/vpc/tests/unit/basic.tftest.hcl

mock_provider "aws" {
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
      state = "available"
    }
  }

  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-0mock1234567890"
      cidr_block = "10.0.0.0/16"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-0mock1234567890"
      availability_zone = "us-east-1a"
    }
  }

  mock_resource "aws_internet_gateway" {
    defaults = {
      id = "igw-0mock1234567890"
    }
  }

  mock_resource "aws_route_table" {
    defaults = {
      id = "rtb-0mock1234567890"
    }
  }

  mock_resource "aws_route_table_association" {
    defaults = {
      id = "rtbassoc-0mock1234567890"
    }
  }
}

# Shared defaults
variables {
  project_name = "myproject"
  environment  = "dev"
  vpc_cidr     = "10.0.0.0/16"
  az_count     = 2
}

# ---- TEST 1: Outputs match inputs ----
run "vpc_cidr_output_matches_input" {
  command = plan

  assert {
    condition     = output.vpc_cidr == var.vpc_cidr
    error_message = "VPC CIDR output '${output.vpc_cidr}' must match input '${var.vpc_cidr}'"
  }
}

# ---- TEST 2: Correct resource counts ----
run "correct_subnet_counts" {
  command = plan

  assert {
    condition     = output.public_subnet_count == var.az_count
    error_message = "Expected ${var.az_count} public subnets, got ${output.public_subnet_count}"
  }

  assert {
    condition     = output.private_subnet_count == var.az_count
    error_message = "Expected ${var.az_count} private subnets, got ${output.private_subnet_count}"
  }
}

# ---- TEST 3: Naming convention ----
run "vpc_name_follows_convention" {
  command = plan

  assert {
    condition     = output.vpc_name == "${var.project_name}-${var.environment}-vpc"
    error_message = "VPC name '${output.vpc_name}' should be '${var.project_name}-${var.environment}-vpc'"
  }
}

# ---- TEST 4: Three AZs when requested ----
run "three_az_deployment" {
  command = plan

  variables {
    az_count = 3
  }

  assert {
    condition     = output.public_subnet_count == 3
    error_message = "Expected 3 public subnets with az_count=3"
  }

  assert {
    condition     = output.private_subnet_count == 3
    error_message = "Expected 3 private subnets with az_count=3"
  }
}

# ---- TEST 5: All outputs are non-null ----
run "all_outputs_non_null" {
  command = plan

  assert {
    condition     = output.vpc_id != null
    error_message = "vpc_id must not be null"
  }

  assert {
    condition     = output.internet_gateway_id != null
    error_message = "internet_gateway_id must not be null"
  }

  assert {
    condition     = length(output.public_subnet_ids) > 0
    error_message = "public_subnet_ids must not be empty"
  }
}
```

```hcl
# modules/vpc/tests/unit/validation.tftest.hcl
# Tests that all validation rules CORRECTLY REJECT bad inputs

mock_provider "aws" {
  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b"]
      state = "available"
    }
  }
  mock_resource "aws_vpc"                    { defaults = { id = "vpc-mock" } }
  mock_resource "aws_subnet"                 { defaults = { id = "subnet-mock" } }
  mock_resource "aws_internet_gateway"       { defaults = { id = "igw-mock" } }
  mock_resource "aws_route_table"            { defaults = { id = "rt-mock" } }
  mock_resource "aws_route_table_association"{ defaults = { id = "rta-mock" } }
}

variables {
  project_name = "validtest"
  environment  = "dev"
  vpc_cidr     = "10.0.0.0/16"
  az_count     = 2
}

# Validation test 1: Invalid environment
run "rejects_invalid_environment" {
  command = plan
  variables {
    environment = "production"
  }
  expect_failures = [var.environment]
}

# Validation test 2: Invalid CIDR format
run "rejects_invalid_cidr_format" {
  command = plan
  variables {
    vpc_cidr = "not-a-cidr"
  }
  expect_failures = [var.vpc_cidr]
}

# Validation test 3: CIDR prefix too large
run "rejects_cidr_prefix_too_large" {
  command = plan
  variables {
    vpc_cidr = "10.0.0.0/8"   # /8 is < /16 minimum
  }
  expect_failures = [var.vpc_cidr]
}

# Validation test 4: CIDR prefix too small
run "rejects_cidr_prefix_too_small" {
  command = plan
  variables {
    vpc_cidr = "10.0.0.0/28"  # /28 is > /24 maximum
  }
  expect_failures = [var.vpc_cidr]
}

# Validation test 5: Invalid az_count
run "rejects_invalid_az_count" {
  command = plan
  variables {
    az_count = 4   # Only 2 or 3 allowed
  }
  expect_failures = [var.az_count]
}

# Validation test 6: Invalid project name
run "rejects_invalid_project_name" {
  command = plan
  variables {
    project_name = "My_Project!"   # Uppercase and special chars not allowed
  }
  expect_failures = [var.project_name]
}
```

```bash
# Run all lab tests
cd modules/vpc

terraform init -backend=false
terraform validate
terraform fmt -check -recursive

# Run unit tests
terraform test tests/unit/ -verbose

# Expected output:
# tests/unit/basic.tftest.hcl... in progress
#   run "vpc_cidr_output_matches_input"... pass
#   run "correct_subnet_counts"... pass
#   run "vpc_name_follows_convention"... pass
#   run "three_az_deployment"... pass
#   run "all_outputs_non_null"... pass
# tests/unit/basic.tftest.hcl... tearing down
# tests/unit/basic.tftest.hcl... pass
#
# tests/unit/validation.tftest.hcl... in progress
#   run "rejects_invalid_environment"... pass
#   run "rejects_invalid_cidr_format"... pass
#   run "rejects_cidr_prefix_too_large"... pass
#   run "rejects_cidr_prefix_too_small"... pass
#   run "rejects_invalid_az_count"... pass
#   run "rejects_invalid_project_name"... pass
# tests/unit/validation.tftest.hcl... pass
#
# Success! 11 passed, 0 failed.
```

---

# 📋 SUMMARY CHEAT SHEET

## Tool Selection Guide

```
QUESTION                                     TOOL
─────────────────────────────────────────────────────────────────────
Is my HCL properly formatted?              terraform fmt -check
Does my HCL have syntax errors?            terraform validate
Am I using wrong instance types?           tflint (aws plugin)
Am I following naming conventions?         tflint (naming rules)
Is my S3 bucket public by accident?        checkov / trivy
Is my database unencrypted?                checkov / trivy
Is my IAM policy too permissive?           checkov / trivy
Does my module output the right values?    terraform test (mock)
Does my validation reject bad input?       terraform test (expect_failures)
Does it work with real AWS?                terraform test (apply) / Terratest
Will this cost too much?                   infracost
Is a secret committed to git?              pre-commit (detect-secrets)
```

---

## Quick Command Reference

```bash
# FORMAT
terraform fmt -recursive              # Fix formatting
terraform fmt -check -recursive       # Check only (CI)

# VALIDATE
terraform validate                    # Syntax check
terraform validate -json              # JSON output for CI

# TFLINT
tflint --init                         # Download plugins
tflint --recursive                    # Lint all modules
tflint --format=sarif > out.sarif     # SARIF for GitHub

# CHECKOV
checkov -d . --compact                # Quick scan
checkov -d . --output sarif           # SARIF output
checkov -d . --skip-check CKV_AWS_272 # Skip specific check

# TRIVY
trivy config .                                        # Scan all
trivy config . --severity HIGH,CRITICAL               # Filter severity
trivy config . --exit-code 1                          # Fail on findings

# TERRAFORM TEST
terraform test                                        # Run all tests
terraform test tests/unit/ -verbose                   # Unit tests only
terraform test -filter=tests/unit/basic.tftest.hcl    # Specific file

# TERRATEST
go test -v -timeout 30m ./...         # All tests
go test -run TestSpecificTest ./...   # Specific test
go test -parallel 4 ./...             # Parallel

# INFRACOST
infracost breakdown --path .          # Cost estimate
infracost diff --compare-to base.json # Cost diff

# PRE-COMMIT
pre-commit install                    # Install hooks
pre-commit run --all-files            # Run all hooks now
pre-commit run terraform-fmt          # Run specific hook
```

---

## Test Types Decision Tree

```
Need to test Terraform code?
│
├── Is it FAST validation (no infra)?
│   ├── Syntax/schema → terraform validate
│   ├── Style/naming  → tflint
│   └── Security      → checkov / trivy
│
├── Is it MODULE LOGIC (outputs, naming, conditionals)?
│   └── terraform test with mock_provider
│       ├── Test happy paths    → run { command = plan; assert {} }
│       └── Test error paths    → run { expect_failures = [var.x] }
│
├── Is it REAL INFRA CREATION (does it actually deploy)?
│   ├── Simple   → terraform test with real provider (command = apply)
│   └── Complex  → Terratest (Go)
│       ├── HTTP checks    → http_helper.HttpGetWithRetry
│       ├── SSH checks     → ssh.CheckSshCommand
│       └── AWS SDK checks → aws.GetEc2InstanceById
│
└── Is it COST?
    └── infracost breakdown / diff
```

---

## Top Interview Questions

**Q1: What is the difference between `terraform fmt`, `terraform validate`, and TFLint?**
> `terraform fmt` only fixes whitespace and formatting — it knows nothing about logic. `terraform validate` checks syntax and schema — it verifies variables and resources are correctly typed and referenced, but doesn't check provider-specific values. TFLint goes deeper: it catches provider-specific errors like invalid instance types, enforces naming conventions, flags missing descriptions, and applies custom rules. You need all three for a complete quality check.

**Q2: What is Checkov and what kind of errors does it catch that TFLint misses?**
> Checkov is a security and compliance scanner that checks for real-world security misconfigurations — unencrypted S3 buckets, publicly accessible databases, overly permissive security groups, IMDSv1 usage, disabled logging, missing deletion protection. TFLint focuses on code style and provider API validity. They are complementary: TFLint for code quality, Checkov for security posture.

**Q3: When would you use `terraform test` vs Terratest?**
> Use `terraform test` (native, HCL-based) for unit testing module logic — it works with mock providers for instant feedback and is easy to write for anyone who knows HCL. Use Terratest (Go-based) for true integration tests that need real infrastructure — when you need to SSH into a server, make HTTP requests, check AWS SDK resource attributes, or test complex multi-resource interactions. Terratest is more powerful but requires Go knowledge and longer test cycles.

**Q4: What is `expect_failures` in `terraform test` and why is it important?**
> `expect_failures` tells Terraform that a specific `run` block is expected to produce a validation error. Without it, you can only test the happy path. With it, you can verify that your `validation` blocks, `precondition`s, and `variable` type constraints correctly reject invalid inputs. Testing that bad input fails is just as important as testing that good input succeeds — it proves your safety guardrails actually work.

**Q5: How do you prevent secret leakage in Terraform code?**
> Several layers: `detect-secrets` in pre-commit scans for credential patterns before commit; never pass secrets via `tfvars` files committed to git; use AWS Secrets Manager or SSM Parameter Store with data sources at runtime; mark sensitive outputs with `sensitive = true`; encrypt the state backend (S3 with SSE); use OIDC/IAM roles instead of long-lived access keys in CI/CD; and scan with Checkov which has built-in secret detection rules.

**Q6: How do you structure a CI/CD pipeline for Terraform quality?**
> A production pipeline has these stages in order: (1) format check — fastest, gates everything else; (2) validate — catches syntax errors per module; (3) linting — TFLint for style/provider rules; (4) security scanning — Checkov and Trivy in parallel; (5) cost estimation — Infracost posting to the PR; (6) unit tests — terraform test with mock providers; (7) integration tests — only on merge to main, with real infrastructure. Each stage must pass before the next runs. The quality gate job at the end requires all previous jobs to succeed before the PR can merge.

**Q7: What is a mock provider in `terraform test` and what are its limitations?**
> A mock provider replaces real API calls with predefined return values, letting tests run instantly without cloud access. You define `mock_resource` and `mock_data` blocks with `defaults` that specify what the provider returns. The main limitation is that mock providers don't validate your configuration against real API constraints — a mock will happily accept an invalid AMI ID or a non-existent subnet. That's why you still need integration tests with real providers for full confidence. Mock tests are best for testing HCL logic: naming conventions, conditional expressions, count/for_each behavior, and variable validations.

---

