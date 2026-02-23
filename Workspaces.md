# Terraform Workspaces: Complete Guide — Beginner to Production-Ready

---

# 📘 BEGINNER LEVEL

## What Are Terraform Workspaces?

**In simple terms:** A workspace is an **isolated copy of your Terraform state** within the same configuration. It lets you use the exact same `.tf` files to manage multiple separate environments without duplicating code.

Every Terraform project starts with one workspace called **`default`**. When you create a new workspace, Terraform creates a completely separate state file for it — so resources in `dev` workspace are totally independent from resources in `prod` workspace.

**Why are they used?**
- Deploy the same infrastructure code to multiple environments (dev, staging, prod)
- Test infrastructure changes in isolation before promoting to production
- Give each developer their own isolated sandbox without separate code
- Avoid duplicating `.tf` files across environment folders

**Real-World Analogy:**

> Imagine you are a **chef with one master recipe** for a dish. You cook it in three separate kitchens — one for tasting/testing, one for family meals, one for a restaurant. Same recipe, same steps, but three completely independent results. Each kitchen has its own ingredients, its own pots, its own finished dishes.
>
> Terraform workspaces are your **separate kitchens**. The recipe (`.tf` files) is identical. But each workspace (kitchen) produces and tracks its own independent infrastructure (dishes). Changing something in the test kitchen doesn't affect the restaurant kitchen at all.

---

## How State Isolation Works

```
Without workspaces (default):           With workspaces:
                                         
terraform.tfstate                        terraform.tfstate.d/
  └── all resources                        ├── dev/
      mixed together                       │   └── terraform.tfstate
                                           ├── staging/
                                           │   └── terraform.tfstate
                                           └── prod/
                                               └── terraform.tfstate
                                         
                                         + terraform.tfstate  (default workspace)
```

Each workspace's state is **completely separate**. Creating an EC2 instance in `dev` workspace has zero effect on `prod` workspace.

---

## Basic Commands — The Workspace CLI

```bash
# See all workspaces (* = currently active)
terraform workspace list

# Output:
# * default
#   dev
#   staging
#   prod

# Create a new workspace
terraform workspace new dev

# Switch to an existing workspace
terraform workspace select prod

# Show the current workspace
terraform workspace show

# Delete a workspace (must not be the current one, must be empty)
terraform workspace delete dev
```

---

## Minimal Working Example (No Cloud Needed)

```hcl
# main.tf

terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# terraform.workspace is a built-in variable that returns the current workspace name
resource "local_file" "environment_config" {
  filename = "./output/config-${terraform.workspace}.txt"
  content  = <<-EOT
    Environment : ${terraform.workspace}
    Debug Mode  : ${terraform.workspace == "prod" ? "false" : "true"}
    Log Level   : ${terraform.workspace == "prod" ? "ERROR" : "DEBUG"}
  EOT
}

output "current_workspace" {
  value = terraform.workspace
}

output "config_file" {
  value = local_file.environment_config.filename
}
```

```bash
mkdir output

# Work in default workspace
terraform init
terraform apply -auto-approve
# Creates: ./output/config-default.txt

# Create and switch to dev workspace
terraform workspace new dev
terraform apply -auto-approve
# Creates: ./output/config-dev.txt  (completely separate state!)

# Create and switch to prod workspace
terraform workspace new prod
terraform apply -auto-approve
# Creates: ./output/config-prod.txt  (separate state again!)

# List all workspaces
terraform workspace list
# Output:
#   default
#   dev
# * prod   ← currently active

# Switch back to dev
terraform workspace select dev
terraform show   # Shows only dev resources — prod is invisible here
```

---

## Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Forgetting to check which workspace you're in before applying
terraform apply    # ❌ Are you in prod or dev right now?!
terraform workspace show && terraform apply  # ✅ Always verify first

# ❌ MISTAKE 2: Using workspaces for fundamentally different infrastructures
# Workspaces are for SAME infrastructure in different environments
# Not for completely different applications or architectures

# ❌ MISTAKE 3: Not using terraform.workspace in resource names
resource "aws_s3_bucket" "app_data" {
  bucket = "my-app-data"   # ❌ Same name in all workspaces = conflict!
}
resource "aws_s3_bucket" "app_data" {
  bucket = "my-app-data-${terraform.workspace}"  # ✅ Unique per workspace
}

# ❌ MISTAKE 4: Treating workspaces as a complete solution for production isolation
# Workspaces share the same backend credentials and provider config
# All workspaces in a project can access each other's state
# For strong isolation (separate AWS accounts), use separate root modules instead

# ❌ MISTAKE 5: Deleting a workspace that still has resources
terraform workspace select dev
terraform destroy -auto-approve  # ✅ Clean up resources FIRST
terraform workspace select default
terraform workspace delete dev   # ✅ THEN delete the workspace

# ❌ MISTAKE 6: Hardcoding environment-specific values
resource "aws_instance" "app" {
  instance_type = "t2.micro"  # ❌ Hardcoded — can't vary by workspace
}
# ✅ Use workspace-driven lookup maps (shown in intermediate section)
```

---

# 📗 INTERMEDIATE LEVEL

## How Workspaces Work Internally

### Local Backend State Layout

```
project/
├── main.tf
├── variables.tf
├── .terraform/
│   └── terraform.tfstate          ← Backend configuration state
├── terraform.tfstate              ← Default workspace state
└── terraform.tfstate.d/           ← Non-default workspace states
    ├── dev/
    │   └── terraform.tfstate      ← Dev workspace state
    ├── staging/
    │   └── terraform.tfstate      ← Staging workspace state
    └── prod/
        └── terraform.tfstate      ← Prod workspace state
```

### Remote Backend State Layout (S3)

```
s3://my-terraform-state/
├── env:/                          ← All non-default workspaces live here
│   ├── dev/
│   │   └── path/to/terraform.tfstate
│   ├── staging/
│   │   └── path/to/terraform.tfstate
│   └── prod/
│       └── path/to/terraform.tfstate
└── path/to/terraform.tfstate      ← Default workspace state
```

```hcl
# backend "s3" with workspaces — state paths are automatic
terraform {
  backend "s3" {
    bucket = "my-company-terraform-state"
    key    = "myapp/terraform.tfstate"
    region = "us-east-1"
    # Terraform automatically manages workspace paths:
    # default   → myapp/terraform.tfstate
    # dev       → env:/dev/myapp/terraform.tfstate
    # prod      → env:/prod/myapp/terraform.tfstate
  }
}
```

---

## The `terraform.workspace` Variable

The most important workspace feature — available everywhere in your config:

```hcl
# terraform.workspace returns the name of the current active workspace

# Use in resource names (most common pattern)
resource "aws_s3_bucket" "data" {
  bucket = "myapp-${terraform.workspace}-data-bucket"
  # dev  workspace → "myapp-dev-data-bucket"
  # prod workspace → "myapp-prod-data-bucket"
}

# Use in conditions
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t2.micro"
}

# Use in tags
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
    ManagedBy   = "terraform"
  }
}

# Use in locals
locals {
  env         = terraform.workspace
  is_prod     = terraform.workspace == "prod"
  is_dev      = terraform.workspace == "dev"
  name_prefix = "${var.project_name}-${terraform.workspace}"
}
```

---

## Workspace-Driven Configuration Maps (Core Pattern)

This is the **most important production pattern** for workspaces — a lookup map that returns different values based on the current workspace:

```hcl
# ============================================
# PATTERN: Workspace Configuration Map
# ============================================

locals {
  # Define all environment-specific values in ONE place
  workspace_config = {
    dev = {
      instance_type    = "t2.micro"
      instance_count   = 1
      min_size         = 1
      max_size         = 2
      db_instance_class = "db.t3.micro"
      db_allocated_storage = 20
      enable_deletion_protection = false
      log_retention_days = 7
      enable_enhanced_monitoring = false
      domain_prefix     = "dev"
    }
    staging = {
      instance_type    = "t2.small"
      instance_count   = 2
      min_size         = 1
      max_size         = 4
      db_instance_class = "db.t3.small"
      db_allocated_storage = 50
      enable_deletion_protection = false
      log_retention_days = 14
      enable_enhanced_monitoring = false
      domain_prefix     = "staging"
    }
    prod = {
      instance_type    = "t3.large"
      instance_count   = 3
      min_size         = 3
      max_size         = 10
      db_instance_class = "db.r5.large"
      db_allocated_storage = 200
      enable_deletion_protection = true
      log_retention_days = 90
      enable_enhanced_monitoring = true
      domain_prefix     = "app"
    }
  }

  # Select current workspace's config (with safety fallback to dev)
  config = lookup(local.workspace_config, terraform.workspace, local.workspace_config["dev"])

  # Convenience aliases — cleaner to reference throughout code
  instance_type    = local.config.instance_type
  instance_count   = local.config.instance_count
  is_prod          = terraform.workspace == "prod"
  name_prefix      = "${var.project_name}-${terraform.workspace}"
}

# Now your resources are clean and DRY
resource "aws_instance" "app" {
  count         = local.instance_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type          # Changes per workspace

  tags = {
    Name        = "${local.name_prefix}-app-${count.index}"
    Environment = terraform.workspace
  }
}

resource "aws_db_instance" "main" {
  instance_class     = local.config.db_instance_class
  allocated_storage  = local.config.db_allocated_storage
  deletion_protection = local.config.enable_deletion_protection

  lifecycle {
    prevent_destroy = local.is_prod ? true : false
    # ⚠️ Note: prevent_destroy can't use dynamic values in older TF versions
    # Use a static true in a prod-specific override file instead
  }
}

resource "aws_cloudwatch_log_group" "app" {
  name              = "/app/${terraform.workspace}/application"
  retention_in_days = local.config.log_retention_days
}
```

---

## Complete Workspace-Aware Configuration

```hcl
# ============================================
# variables.tf
# ============================================
variable "project_name" {
  type        = string
  description = "Project name used as prefix for all resources"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

# ============================================
# locals.tf
# ============================================
locals {
  # Workspace validation — fail fast if invalid workspace name used
  allowed_workspaces = ["dev", "staging", "prod"]
  workspace_is_valid = contains(local.allowed_workspaces, terraform.workspace)

  # Workspace-driven config map
  workspace_config = {
    dev = {
      instance_type         = "t2.micro"
      min_capacity          = 1
      max_capacity          = 2
      desired_capacity      = 1
      multi_az              = false
      backup_retention_days = 1
      alarm_evaluation_periods = 1
    }
    staging = {
      instance_type         = "t2.small"
      min_capacity          = 1
      max_capacity          = 4
      desired_capacity      = 2
      multi_az              = false
      backup_retention_days = 3
      alarm_evaluation_periods = 2
    }
    prod = {
      instance_type         = "t3.medium"
      min_capacity          = 3
      max_capacity          = 10
      desired_capacity      = 3
      multi_az              = true
      backup_retention_days = 14
      alarm_evaluation_periods = 3
    }
  }

  env    = terraform.workspace
  config = local.workspace_config[local.env]

  is_prod    = local.env == "prod"
  is_dev     = local.env == "dev"
  is_staging = local.env == "staging"

  name_prefix = "${var.project_name}-${local.env}"

  common_tags = {
    Project     = var.project_name
    Environment = local.env
    ManagedBy   = "terraform"
    Workspace   = terraform.workspace
  }
}

# ============================================
# validation.tf — Fail fast on wrong workspace
# ============================================
resource "null_resource" "workspace_validation" {
  count = local.workspace_is_valid ? 0 : 1
  # If workspace is invalid, this resource would be created (count=1)
  # but we intentionally make it fail with a provisioner

  provisioner "local-exec" {
    command = "echo 'ERROR: Invalid workspace. Must be one of: ${join(", ", local.allowed_workspaces)}' && exit 1"
  }
}

# Better approach with Terraform 1.2+ — use check blocks
check "valid_workspace" {
  assert {
    condition     = contains(["dev", "staging", "prod"], terraform.workspace)
    error_message = "Workspace must be dev, staging, or prod. Current: ${terraform.workspace}"
  }
}
```

---

## Workspace vs Other Approaches — When to Use Which

This is one of the **most debated topics** in the Terraform community. Understanding the tradeoffs is essential for interviews and production decisions.

```
┌─────────────────────────────────────────────────────────────────────┐
│         Workspaces vs Directory-per-Environment vs Branches          │
├──────────────────────┬─────────────────┬──────────────────────────  │
│ Feature              │ Workspaces      │ Directories / Root Modules  │
├──────────────────────┼─────────────────┼──────────────────────────  │
│ Code duplication     │ None (shared)   │ Some (separate tfvars)      │
│ State isolation      │ Same backend    │ Separate backends possible  │
│ Account isolation    │ ❌ Hard         │ ✅ Easy (diff providers)    │
│ Config differences   │ In-code logic   │ In tfvars files             │
│ Blast radius         │ ⚠️ Shared creds │ ✅ Isolated credentials     │
│ Team autonomy        │ Limited         │ High                        │
│ Complexity           │ Low             │ Medium                      │
│ Best for             │ Simple/similar  │ Complex/regulated envs      │
│                      │ environments    │ multi-account setups        │
└──────────────────────┴─────────────────┴──────────────────────────  │
```

**Use Workspaces When:**
- Environments are nearly identical (dev/staging/prod same config, different sizes)
- Small teams where one person manages all environments
- Rapid prototyping and experimentation
- Infrastructure that's the same across environments structurally

**Don't Use Workspaces When:**
- Different environments need completely different AWS accounts (compliance/security)
- Large teams where dev team shouldn't have any access to production state
- Environments have significantly different architecture (not just different sizes)
- You need strong blast radius protection (a mistake in dev shouldn't be able to affect prod)
- Environments are managed by separate teams

---

## Interview-Focused Points

> **Q: What is a Terraform workspace and what problem does it solve?**
> A workspace is an isolated state environment within the same Terraform configuration. It lets you deploy the same infrastructure code to multiple environments (dev/staging/prod) without duplicating `.tf` files, by maintaining separate state files per workspace.

> **Q: What is the default workspace? Can you delete it?**
> Every Terraform project has a `default` workspace created automatically. You cannot delete the default workspace — Terraform will reject that command with an error.

> **Q: How does `terraform.workspace` work?**
> It's a built-in string expression that returns the name of the currently active workspace. It's available anywhere in your HCL code — in resources, locals, variables, outputs. Common use: `"${var.name}-${terraform.workspace}"` to create unique resource names per environment.

> **Q: What's the difference between workspaces and separate Terraform state files?**
> Workspaces use one backend configuration but automatically namespace the state paths. Separate state files use completely separate backends — different S3 buckets, different access credentials, stronger isolation. Workspaces are simpler; separate backends provide stronger security boundaries.

> **Q: When would you NOT use workspaces?**
> When you need different AWS accounts per environment (regulatory compliance), when teams need isolated access (dev team shouldn't see prod state), or when environments have fundamentally different architectures. In those cases, use separate root modules with separate backends.

---

# 📕 ADVANCED LEVEL

## Deep Dive: State Backend Behavior with Workspaces

Understanding how different backends handle workspace state is critical for production:

```hcl
# ============================================
# S3 Backend — Workspace State Paths
# ============================================
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "services/myapp/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

# Resulting S3 paths:
# default   → s3://company-terraform-state/services/myapp/terraform.tfstate
# dev       → s3://company-terraform-state/env:/dev/services/myapp/terraform.tfstate
# staging   → s3://company-terraform-state/env:/staging/services/myapp/terraform.tfstate
# prod      → s3://company-terraform-state/env:/prod/services/myapp/terraform.tfstate


# ============================================
# Terraform Cloud Backend — Workspaces work differently!
# ============================================
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      # Option 1: Map ONE TF Cloud workspace to ONE local workspace
      name = "myapp-production"
      # ⚠️ In this mode, local workspace commands DON'T work
      # The TF Cloud workspace IS the environment

      # Option 2: Use tags to manage multiple workspaces
      tags = ["myapp"]
      # This maps to ALL TF Cloud workspaces tagged "myapp"
      # myapp-dev, myapp-staging, myapp-prod
    }
  }
}
# ⚠️ IMPORTANT: Terraform Cloud has its own workspace concept
# that is MORE powerful than CLI workspaces.
# Each TF Cloud workspace can have separate:
# - Variables and secrets
# - Team access controls
# - VCS triggers
# - Execution environments
```

---

## Advanced: Workspace-Driven Module Composition

```hcl
# ============================================
# Different module configurations per workspace
# ============================================

# Only create monitoring stack in staging and prod
module "monitoring" {
  count  = terraform.workspace != "dev" ? 1 : 0
  source = "./modules/monitoring"

  environment       = terraform.workspace
  alert_email       = local.config.alert_email
  enable_pagerduty  = local.is_prod
}

# Only create WAF in production
module "waf" {
  count  = local.is_prod ? 1 : 0
  source = "./modules/waf"

  alb_arn     = module.networking.alb_arn
  environment = terraform.workspace
}

# Different database module for prod (RDS Multi-AZ) vs dev (single instance)
module "database" {
  source = local.is_prod ? "./modules/rds-ha" : "./modules/rds-simple"

  environment       = terraform.workspace
  instance_class    = local.config.db_instance_class
  multi_az          = local.config.multi_az
}

# Networking with environment-specific CIDR ranges
locals {
  vpc_cidrs = {
    dev     = "10.10.0.0/16"
    staging = "10.20.0.0/16"
    prod    = "10.30.0.0/16"
  }
  vpc_cidr = local.vpc_cidrs[terraform.workspace]
}

module "networking" {
  source   = "./modules/networking"
  vpc_cidr = local.vpc_cidr
  name     = local.name_prefix
}
```

---

## Advanced: Workspace-Aware Remote State Reading

```hcl
# ============================================
# Read networking state for the CURRENT workspace
# ============================================

# Dynamically select the right networking state based on current workspace
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "company-terraform-state"
    # Use workspace name to build the correct state path
    key    = "env:/${terraform.workspace}/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Now your app module automatically reads the matching environment's network
resource "aws_instance" "app" {
  ami       = data.aws_ami.ubuntu.id
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
  # dev   workspace reads dev   networking state
  # prod  workspace reads prod  networking state
  # Automatic — no manual configuration needed!
}


# ============================================
# Cross-workspace state reading (advanced)
# ============================================

# Sometimes you need prod data from a dev workspace (e.g., for migration testing)
data "terraform_remote_state" "prod_database" {
  backend = "s3"
  config = {
    bucket = "company-terraform-state"
    key    = "env:/prod/database/terraform.tfstate"   # Hardcoded prod path
    region = "us-east-1"
  }
}

# Use prod DB endpoint in dev for read-replica testing
locals {
  db_endpoint = local.is_prod ? (
    aws_db_instance.primary.endpoint
  ) : (
    data.terraform_remote_state.prod_database.outputs.read_replica_endpoint
  )
}
```

---

## Advanced: CI/CD Pipeline Integration

This is how workspaces work in real production pipelines:

```bash
#!/bin/bash
# deploy.sh — Production CI/CD deployment script

set -euo pipefail

ENVIRONMENT="${1:-dev}"   # First argument: dev, staging, or prod
PROJECT="myapp"

# Validate environment argument
if [[ ! "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]]; then
  echo "ERROR: Environment must be dev, staging, or prod"
  exit 1
fi

echo "=== Deploying ${PROJECT} to ${ENVIRONMENT} ==="

# Initialize with the shared backend
terraform init \
  -backend-config="bucket=company-terraform-state" \
  -backend-config="key=${PROJECT}/terraform.tfstate" \
  -backend-config="region=us-east-1"

# Create workspace if it doesn't exist, then select it
terraform workspace select "${ENVIRONMENT}" 2>/dev/null || \
  terraform workspace new "${ENVIRONMENT}"

echo "Active workspace: $(terraform workspace show)"

# Always plan first, save the plan
terraform plan \
  -var="project_name=${PROJECT}" \
  -var-file="environments/${ENVIRONMENT}.tfvars" \
  -out="${ENVIRONMENT}.tfplan"

# In production: require manual approval for prod
if [[ "$ENVIRONMENT" == "prod" ]]; then
  echo "=== PRODUCTION DEPLOYMENT — Human approval required ==="
  read -p "Type 'yes' to proceed: " confirm
  if [[ "$confirm" != "yes" ]]; then
    echo "Deployment cancelled."
    exit 0
  fi
fi

# Apply the saved plan
terraform apply "${ENVIRONMENT}.tfplan"

echo "=== Deployment complete: ${PROJECT} → ${ENVIRONMENT} ==="
```

```yaml
# .github/workflows/terraform.yml — GitHub Actions CI/CD
name: Terraform Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "us-east-1"

jobs:
  plan-dev:
    name: Plan Dev
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::DEV_ACCOUNT:role/TerraformDeployRole
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Select Dev Workspace
        run: |
          terraform workspace select dev || terraform workspace new dev

      - name: Terraform Plan
        run: terraform plan -var-file="environments/dev.tfvars"

  deploy-dev:
    name: Deploy Dev
    needs: plan-dev
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: dev                    # GitHub environment protection rules
    steps:
      - uses: actions/checkout@v4
      # ... setup steps ...
      - name: Select Dev Workspace
        run: terraform workspace select dev

      - name: Terraform Apply
        run: terraform apply -auto-approve -var-file="environments/dev.tfvars"

  deploy-prod:
    name: Deploy Production
    needs: deploy-dev
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production              # Requires manual approval in GitHub
    steps:
      - uses: actions/checkout@v4
      # ... setup steps with PROD AWS credentials ...
      - name: Select Prod Workspace
        run: terraform workspace select prod

      - name: Terraform Plan
        run: terraform plan -var-file="environments/prod.tfvars" -out=prod.tfplan

      - name: Terraform Apply
        run: terraform apply prod.tfplan
```

---

## Edge Cases and Gotchas

```hcl
# ============================================
# EDGE CASE 1: lifecycle blocks can't use terraform.workspace
# ============================================

# ❌ This does NOT work — lifecycle args must be static
resource "aws_db_instance" "main" {
  instance_class = local.config.db_instance_class

  lifecycle {
    prevent_destroy = terraform.workspace == "prod" ? true : false
    # ERROR: The "lifecycle" meta-argument does not support computed values
  }
}

# ✅ Workaround 1: Always set prevent_destroy = true
# And rely on workspace discipline / CI guards for non-prod

# ✅ Workaround 2: Use a separate override file for prod
# environments/prod-override.tf — only exists for prod deployments
# resource "aws_db_instance" "main" {
#   lifecycle {
#     prevent_destroy = true
#   }
# }

# ✅ Workaround 3: Use a wrapper script that only applies in prod workspace
# if [[ "$(terraform workspace show)" == "prod" ]]; then
#   cp environments/prod-lifecycle-override.tf .
# fi


# ============================================
# EDGE CASE 2: S3 bucket names must be globally unique
# ============================================
resource "aws_s3_bucket" "data" {
  # ❌ workspace name alone might not be unique enough
  bucket = "data-${terraform.workspace}"
  # "data-prod" might already be taken by another AWS account!

  # ✅ Include account ID and region for true uniqueness
  bucket = "data-${data.aws_caller_identity.current.account_id}-${terraform.workspace}-${data.aws_region.current.name}"
}


# ============================================
# EDGE CASE 3: Workspace state lock conflicts
# ============================================
# If two engineers run `terraform apply` on the same workspace simultaneously:
# Engineer 1 gets the DynamoDB lock — proceeds
# Engineer 2 gets: "Error: Error locking state: Error acquiring the state lock"
#
# This is CORRECT BEHAVIOR — prevents state corruption
# Engineer 2 must wait for Engineer 1 to finish
# Or: terraform force-unlock LOCK_ID  (use with extreme caution!)


# ============================================
# EDGE CASE 4: workspace new vs workspace select in scripts
# ============================================

# This fails if workspace doesn't exist:
terraform workspace select prod   # ❌ Error if "prod" doesn't exist

# This fails if workspace already exists:
terraform workspace new prod      # ❌ Error if "prod" already exists

# ✅ Idempotent pattern for CI/CD:
terraform workspace select prod 2>/dev/null || terraform workspace new prod


# ============================================
# EDGE CASE 5: Default workspace naming confusion
# ============================================
resource "aws_instance" "web" {
  tags = {
    Environment = terraform.workspace
    # If someone runs in "default" workspace:
    # Environment = "default" ← confusing tag in AWS console!
  }
}

# ✅ Guard against default workspace usage in production code
locals {
  env = terraform.workspace == "default" ? "dev" : terraform.workspace
  # Or: explicitly block default workspace
}

check "no_default_workspace" {
  assert {
    condition     = terraform.workspace != "default"
    error_message = "Do not use the 'default' workspace. Use 'dev', 'staging', or 'prod'."
  }
}
```

---

## Performance Considerations

```bash
# ============================================
# PERFORMANCE: Large state files and workspaces
# ============================================

# Problem: As environments grow, state files get large
# Each workspace's state is independent but both are read during operations

# Solution 1: -target to work on specific resources
terraform plan -target=module.compute

# Solution 2: Split large configs into separate state files
# Instead of one monolithic workspace system:
# networking/  → manages VPC, subnets (separate state per workspace)
# compute/     → manages EC2, ASG (separate state per workspace)
# database/    → manages RDS (separate state per workspace)

# Solution 3: Use -refresh=false during development iteration
terraform plan -refresh=false   # Skips re-reading all data sources

# Solution 4: Targeted refresh
terraform apply -refresh-only -target=data.aws_ami.ubuntu
```

---

## Debugging Workspaces

```bash
# ============================================
# DEBUGGING TOOLKIT FOR WORKSPACES
# ============================================

# 1. Always know your current workspace
terraform workspace show

# 2. List all workspaces and their state
terraform workspace list
# * prod   ← active workspace (star = current)
#   dev
#   staging

# 3. Inspect state per workspace
terraform workspace select dev
terraform state list                      # All dev resources
terraform state show aws_instance.app[0] # Specific dev resource

terraform workspace select prod
terraform state list                      # All prod resources — completely separate

# 4. Compare plans across workspaces
terraform workspace select dev
terraform plan -out=dev.tfplan
terraform workspace select prod
terraform plan -out=prod.tfplan
# Review both plans independently

# 5. Debug workspace variable resolution
terraform workspace select staging
terraform console
> terraform.workspace
"staging"
> local.config.instance_type
"t2.small"
> local.name_prefix
"myapp-staging"

# 6. Check S3 state paths for each workspace
aws s3 ls s3://company-terraform-state/env:/ --recursive
# Lists all non-default workspace state files

# 7. Verify state lock table in DynamoDB
aws dynamodb scan \
  --table-name terraform-state-lock \
  --query "Items[*].{LockID:LockID.S,Who:Info.S}" \
  --output table
# Shows any active locks across all workspaces

# 8. Force unlock (emergency use only!)
terraform force-unlock LOCK_ID
# DANGER: Only use if you're 100% certain no apply is running!

# 9. Validate workspace config map completeness
terraform console
> local.workspace_config
# Prints full config map — verify all workspaces are defined

# 10. Dry-run workspace switch (safe check)
terraform workspace select staging
terraform plan -detailed-exitcode
# Exit code 0: No changes
# Exit code 1: Error
# Exit code 2: Changes present
```

---

# 🏗️ REAL-WORLD PRODUCTION ARCHITECTURE

## Complete: Multi-Environment Web Application with Workspaces

```hcl
# ============================================
# PROJECT STRUCTURE:
# infrastructure/
# ├── main.tf
# ├── variables.tf
# ├── locals.tf
# ├── networking.tf
# ├── compute.tf
# ├── database.tf
# ├── monitoring.tf
# ├── outputs.tf
# └── environments/
#     ├── dev.tfvars
#     ├── staging.tfvars
#     └── prod.tfvars
# ============================================


# environments/dev.tfvars
project_name = "webshop"
aws_region   = "us-east-1"

# environments/prod.tfvars
project_name = "webshop"
aws_region   = "us-east-1"


# variables.tf
variable "project_name" { type = string }
variable "aws_region"   { type = string }


# locals.tf — The entire environment config in one place
locals {
  env = terraform.workspace

  workspace_config = {
    dev = {
      # Compute
      instance_type    = "t2.micro"
      min_capacity     = 1
      max_capacity     = 2
      desired_capacity = 1
      # Database
      db_class         = "db.t3.micro"
      db_storage       = 20
      db_multi_az      = false
      db_backup_days   = 1
      # Networking
      vpc_cidr          = "10.10.0.0/16"
      # Features
      enable_cdn        = false
      enable_waf        = false
      enable_monitoring = false
      alarm_email       = "dev-alerts@company.com"
      log_retention     = 7
    }
    staging = {
      instance_type    = "t2.small"
      min_capacity     = 1
      max_capacity     = 4
      desired_capacity = 2
      db_class         = "db.t3.small"
      db_storage       = 50
      db_multi_az      = false
      db_backup_days   = 3
      vpc_cidr          = "10.20.0.0/16"
      enable_cdn        = true
      enable_waf        = false
      enable_monitoring = true
      alarm_email       = "staging-alerts@company.com"
      log_retention     = 14
    }
    prod = {
      instance_type    = "t3.medium"
      min_capacity     = 3
      max_capacity     = 10
      desired_capacity = 3
      db_class         = "db.r5.large"
      db_storage       = 500
      db_multi_az      = true
      db_backup_days   = 30
      vpc_cidr          = "10.30.0.0/16"
      enable_cdn        = true
      enable_waf        = true
      enable_monitoring = true
      alarm_email       = "prod-alerts@company.com"
      log_retention     = 90
    }
  }

  config      = local.workspace_config[local.env]
  is_prod     = local.env == "prod"
  name_prefix = "${var.project_name}-${local.env}"

  common_tags = {
    Project     = var.project_name
    Environment = local.env
    ManagedBy   = "terraform"
    Workspace   = terraform.workspace
  }
}


# networking.tf
resource "aws_vpc" "main" {
  cidr_block           = local.config.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(local.config.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(local.config.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Tier = "private"
  })
}


# compute.tf
resource "aws_autoscaling_group" "app" {
  name                = "${local.name_prefix}-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  min_size            = local.config.min_capacity
  max_size            = local.config.max_capacity
  desired_capacity    = local.config.desired_capacity

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes = [desired_capacity]
  }

  dynamic "tag" {
    for_each = merge(local.common_tags, { Name = "${local.name_prefix}-instance" })
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}

resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = local.config.instance_type

  user_data = base64encode(templatefile("${path.module}/templates/userdata.sh.tpl", {
    environment  = local.env
    project      = var.project_name
    db_endpoint  = aws_db_instance.main.endpoint
  }))

  lifecycle {
    create_before_destroy = true
  }
}


# database.tf
resource "aws_db_instance" "main" {
  identifier        = "${local.name_prefix}-postgres"
  engine            = "postgres"
  engine_version    = "14"
  instance_class    = local.config.db_class
  allocated_storage = local.config.db_storage
  multi_az          = local.config.db_multi_az

  db_name  = replace(var.project_name, "-", "_")
  username = "dbadmin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string

  backup_retention_period = local.config.db_backup_days
  deletion_protection     = local.is_prod
  skip_final_snapshot     = !local.is_prod

  final_snapshot_identifier = local.is_prod ? "${local.name_prefix}-final-snapshot" : null

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-postgres"
  })
}


# monitoring.tf — Only created in staging and prod
module "monitoring" {
  count  = local.config.enable_monitoring ? 1 : 0
  source = "./modules/monitoring"

  name_prefix   = local.name_prefix
  environment   = local.env
  alarm_email   = local.config.alarm_email
  asg_name      = aws_autoscaling_group.app.name
  db_identifier = aws_db_instance.main.identifier
  tags          = local.common_tags
}

# WAF — Only in production
module "waf" {
  count  = local.config.enable_waf ? 1 : 0
  source = "./modules/waf"

  name        = "${local.name_prefix}-waf"
  environment = local.env
  tags        = local.common_tags
}


# outputs.tf
output "workspace_summary" {
  value = {
    workspace        = terraform.workspace
    instance_type    = local.config.instance_type
    min_capacity     = local.config.min_capacity
    max_capacity     = local.config.max_capacity
    vpc_cidr         = local.config.vpc_cidr
    db_class         = local.config.db_class
    multi_az         = local.config.db_multi_az
    cdn_enabled      = local.config.enable_cdn
    waf_enabled      = local.config.enable_waf
    monitoring       = local.config.enable_monitoring
  }
  description = "Summary of current workspace configuration"
}

output "connection_info" {
  sensitive = true
  value = {
    db_endpoint = aws_db_instance.main.endpoint
    asg_name    = aws_autoscaling_group.app.name
    vpc_id      = aws_vpc.main.id
  }
}
```

---

# 🧪 HANDS-ON LAB

## Lab: Multi-Environment File System Manager

**Objective:** Build a workspace-driven system that generates environment-specific configurations, demonstrates state isolation, and enforces workspace validation.

**Requirements — uses local provider, no cloud needed:**
1. Use `terraform.workspace` to drive all configuration differences
2. Create a workspace config map with dev, staging, and prod settings
3. Generate environment-specific files with different content per workspace
4. Block usage of the `default` workspace with a check block
5. Use `count = 0` to conditionally create resources based on workspace
6. Show that state is truly isolated between workspaces

**Expected behaviour:**
```bash
# In dev workspace:
terraform apply
# Creates: output/dev/app.conf, output/dev/summary.txt
# Does NOT create: output/dev/monitoring.conf (dev has no monitoring)

# In prod workspace:
terraform apply
# Creates: output/prod/app.conf, output/prod/summary.txt,
#          output/prod/monitoring.conf (prod has monitoring!)
```

**Try it yourself before reading the solution!**

---

## Lab Solution

```hcl
# ============================================
# main.tf
# ============================================
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# ============================================
# locals.tf
# ============================================
locals {
  workspace_config = {
    dev = {
      log_level          = "DEBUG"
      replicas           = 1
      enable_monitoring  = false
      enable_cache       = false
      db_size            = "small"
      feature_flags      = ["dark-mode", "beta-ui"]
      retention_days     = 7
    }
    staging = {
      log_level          = "INFO"
      replicas           = 2
      enable_monitoring  = true
      enable_cache       = false
      db_size            = "medium"
      feature_flags      = ["dark-mode"]
      retention_days     = 14
    }
    prod = {
      log_level          = "WARN"
      replicas           = 5
      enable_monitoring  = true
      enable_cache       = true
      db_size            = "large"
      feature_flags      = []
      retention_days     = 90
    }
  }

  env    = terraform.workspace
  config = lookup(local.workspace_config, local.env, null)

  output_dir = "./output/${local.env}"
}

# ============================================
# Validation: block default workspace
# ============================================
check "workspace_not_default" {
  assert {
    condition     = terraform.workspace != "default"
    error_message = "Do not use 'default' workspace. Run: terraform workspace new dev"
  }
}

check "workspace_is_valid" {
  assert {
    condition     = contains(keys(local.workspace_config), terraform.workspace)
    error_message = "Invalid workspace '${terraform.workspace}'. Use: dev, staging, or prod."
  }
}

# ============================================
# Resource 1: App configuration file (all envs)
# ============================================
resource "local_file" "app_config" {
  filename = "${local.output_dir}/app.conf"
  content  = <<-EOT
    # App Configuration — ${upper(local.env)} Environment
    # Generated by Terraform Workspace: ${terraform.workspace}

    [server]
    log_level  = ${local.config.log_level}
    replicas   = ${local.config.replicas}

    [database]
    size       = ${local.config.db_size}

    [cache]
    enabled    = ${local.config.enable_cache}

    [features]
    flags      = ${length(local.config.feature_flags) > 0 ? join(",", local.config.feature_flags) : "none"}

    [retention]
    days       = ${local.config.retention_days}
  EOT
}

# ============================================
# Resource 2: Summary file (all envs)
# ============================================
resource "local_file" "summary" {
  filename = "${local.output_dir}/summary.txt"
  content  = <<-EOT
    =========================================
    WORKSPACE: ${upper(terraform.workspace)}
    =========================================
    Replicas    : ${local.config.replicas}
    DB Size     : ${local.config.db_size}
    Monitoring  : ${local.config.enable_monitoring ? "ENABLED" : "DISABLED"}
    Cache       : ${local.config.enable_cache ? "ENABLED" : "DISABLED"}
    Log Level   : ${local.config.log_level}
    Log Retention: ${local.config.retention_days} days
    Features    : ${length(local.config.feature_flags) > 0 ? join(", ", local.config.feature_flags) : "none"}
    =========================================
  EOT
}

# ============================================
# Resource 3: Monitoring config — staging and prod ONLY
# ============================================
resource "local_file" "monitoring_config" {
  count    = local.config.enable_monitoring ? 1 : 0
  filename = "${local.output_dir}/monitoring.conf"
  content  = <<-EOT
    # Monitoring Configuration — ${upper(local.env)}
    [alerts]
    enabled    = true
    channel    = ${local.env}-alerts

    [metrics]
    interval   = ${local.env == "prod" ? "30" : "60"}s
    retention  = ${local.config.retention_days}d
  EOT
}

# ============================================
# outputs.tf
# ============================================
output "workspace_info" {
  value = {
    active_workspace = terraform.workspace
    config           = local.config
    files_created    = concat(
      [local_file.app_config.filename, local_file.summary.filename],
      local_file.monitoring_config[*].filename
    )
  }
}

output "monitoring_status" {
  value = local.config.enable_monitoring ? "Monitoring ENABLED for ${local.env}" : "Monitoring DISABLED for ${local.env}"
}
```

```bash
# ============================================
# Run the Lab
# ============================================

# Setup
mkdir -p output
terraform init

# Try using default workspace (should warn)
terraform plan
# check "workspace_not_default" will trigger warning

# Create and test dev workspace
terraform workspace new dev
terraform apply -auto-approve
cat output/dev/app.conf
cat output/dev/summary.txt
ls output/dev/          # No monitoring.conf in dev!

# Create and test staging workspace
terraform workspace new staging
terraform apply -auto-approve
cat output/staging/summary.txt
ls output/staging/      # monitoring.conf exists in staging!

# Create and test prod workspace
terraform workspace new prod
terraform apply -auto-approve
cat output/prod/summary.txt
ls output/prod/          # monitoring.conf exists in prod!

# Prove state isolation — switch back to dev
terraform workspace select dev
terraform state list     # Only shows dev resources!
# output: local_file.app_config, local_file.summary
#         NO monitoring_config (dev doesn't have it)

terraform workspace select prod
terraform state list     # Shows prod resources (including monitoring!)
# output: local_file.app_config, local_file.summary, local_file.monitoring_config[0]

# List all workspaces
terraform workspace list
#   default
#   dev
#   staging
# * prod

# Try invalid workspace name
terraform workspace new invalid-env
terraform apply -auto-approve
# check "workspace_is_valid" triggers error!

# Clean up
terraform workspace select dev && terraform destroy -auto-approve
terraform workspace select staging && terraform destroy -auto-approve
terraform workspace select prod && terraform destroy -auto-approve
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

---

# 📋 SUMMARY CHEAT SHEET

## Key Concepts

| Concept | Detail |
|---|---|
| Default workspace | Always exists, cannot be deleted |
| `terraform.workspace` | Built-in variable — current workspace name |
| State isolation | Each workspace has completely separate state |
| Local state path | `terraform.tfstate.d/<workspace>/terraform.tfstate` |
| S3 state path | `env:/<workspace>/<key>` |
| Best for | Same infra, multiple environments (dev/staging/prod) |
| Not for | Different AWS accounts, radically different architectures |

---

## Quick Command Reference

```bash
terraform workspace list              # List all workspaces
terraform workspace show              # Show current workspace
terraform workspace new <name>        # Create new workspace
terraform workspace select <name>     # Switch workspace
terraform workspace delete <name>     # Delete workspace (must be empty)

# Idempotent select-or-create (for CI/CD)
terraform workspace select prod 2>/dev/null || terraform workspace new prod

# Always verify workspace before applying
terraform workspace show && terraform plan
```

---

## Quick Syntax Reference

```hcl
# Reference current workspace name
terraform.workspace

# Workspace-driven config map (core pattern)
locals {
  config = {
    dev  = { instance_type = "t2.micro",  replicas = 1 }
    prod = { instance_type = "t3.large",  replicas = 5 }
  }[terraform.workspace]
}

# Conditional resource creation
resource "aws_instance" "monitor" {
  count = terraform.workspace == "prod" ? 1 : 0
}

# Unique resource names per workspace
resource "aws_s3_bucket" "data" {
  bucket = "myapp-${terraform.workspace}-data"
}

# Workspace validation (Terraform 1.2+)
check "valid_workspace" {
  assert {
    condition     = contains(["dev", "staging", "prod"], terraform.workspace)
    error_message = "Invalid workspace: ${terraform.workspace}"
  }
}

# Block default workspace
check "no_default" {
  assert {
    condition     = terraform.workspace != "default"
    error_message = "Do not use the default workspace."
  }
}
```

---

## Top Interview Questions on Workspaces

**Q1: What is a Terraform workspace and how does it work?**
> A workspace is a named, isolated copy of Terraform state within the same backend configuration. The same `.tf` code can be applied in multiple workspaces, each maintaining completely separate state. The current workspace name is available via `terraform.workspace`.

**Q2: What is the `default` workspace? Can it be deleted?**
> `default` is the workspace every Terraform project starts with and always falls back to. It cannot be deleted — Terraform enforces this. Best practice is to not use `default` for any real environment — create named workspaces like `dev`, `staging`, `prod` instead.

**Q3: How are workspace state files stored in S3?**
> The default workspace state is stored at the configured key path directly. Non-default workspaces are stored under the prefix `env:/<workspace-name>/<key>`. So `key = "myapp/terraform.tfstate"` in `dev` workspace becomes `env:/dev/myapp/terraform.tfstate`.

**Q4: What are the limitations of workspaces versus separate root modules?**
> Workspaces share the same backend credentials and provider configuration, so all workspaces have equal access to the cloud account. They can't provide account-level isolation. Separate root modules with separate backends can use different AWS accounts and IAM roles per environment — much stronger security boundary.

**Q5: How do you prevent someone from accidentally running terraform destroy in production workspace?**
> Several layers: use `prevent_destroy` lifecycle on critical resources; add `check` blocks that error if workspace is `prod` and certain variables aren't set; use CI/CD pipeline gates requiring manual approval for prod; restrict who has AWS credentials that allow prod access; use Terraform Cloud's team-based access controls.

**Q6: Can you use `terraform.workspace` inside a `lifecycle` block?**
> No — lifecycle block arguments must be static values, not computed expressions. `prevent_destroy = terraform.workspace == "prod"` is a common mistake that fails. Workarounds include always setting `prevent_destroy = true` and relying on CI/CD controls, or using prod-specific override files.

**Q7: What is the idiomatic pattern for workspace-specific configuration?**
> A locals-based workspace config map: define a map keyed by workspace name containing all environment-specific values, then select the current environment's config with `local.workspace_config[terraform.workspace]`. This centralises all environment differences in one readable place.

---
