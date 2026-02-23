# 🌍 TOPIC 6: Workspaces & Environment Management — Deep Dive

---

## 🟢 BEGINNER LEVEL

### What Are Workspaces & Environment Management? (Simple Terms)

Imagine you're a **painter who works on multiple houses** on the same street. Each house needs the same rooms (kitchen, bedroom, bathroom) but different colors, sizes, and finishes.

You have two approaches:
- **Bad approach:** Use one notebook for all houses, cross things out, get confused about which house you're painting
- **Good approach:** Have a **separate notebook per house** — same template, different details, completely independent

**Environment management in Terraform is about keeping your dev, staging, and prod infrastructure completely independent** — same code, different configurations, zero risk of prod being affected by dev changes.

```
DEV environment     →  Experiments OK, small sizes, cheap
STAGING environment →  Mirror of prod, testing final changes
PROD environment    →  Real users, high availability, expensive
                        NEVER break this accidentally
```

---

### What Is a Workspace?

A Terraform workspace is a **named state file slot** — same backend, same code, but isolated state per workspace.

```
Default setup (no workspaces):
  S3 bucket → terraform.tfstate   ← ONE state for everything

With workspaces:
  S3 bucket → env:/dev/terraform.tfstate
  S3 bucket → env:/staging/terraform.tfstate
  S3 bucket → env:/prod/terraform.tfstate
```

Think of workspaces like **browser profiles** — same browser (Terraform), same bookmarks template (your `.tf` files), but different cookies, history, and logged-in accounts (state) per profile.

---

### Real-World Analogy

```
HOTEL ANALOGY:
─────────────────────────────────────────────────────────────────
Hotel Building      →  Your Terraform code (same blueprint)
Hotel Room          →  Workspace / Environment
Room Number         →  Workspace name ("dev", "prod")
Room State          →  Terraform state file (what's in the room)
Guest               →  Your application
Housekeeping Sheet  →  State file tracking room contents

Room 101 (dev)  — small bed, basic TV, low cost
Room 201 (prod) — king bed, 4K TV, minibar, high cost

Same hotel, same blueprint, completely independent rooms.
Cleaning room 101 doesn't affect room 201.
```

---

### The Two Approaches to Environment Management

```
APPROACH 1: WORKSPACES
────────────────────────────────────────────
One directory, one codebase
Multiple state files via workspace names
terraform workspace new dev
terraform workspace new prod
├── main.tf        (same code)
├── variables.tf   (same vars)
└── terraform.tfvars  (different per env)


APPROACH 2: DIRECTORY SEPARATION (Recommended)
────────────────────────────────────────────────
Separate directories per environment
Each has its own state, its own backend
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

---

### Basic Workspace Commands

```bash
# Show current workspace
terraform workspace show
# default

# List all workspaces
terraform workspace list
# * default   ← asterisk = current

# Create a new workspace
terraform workspace new dev
# Created and switched to workspace "dev"!

# Switch workspace
terraform workspace select prod

# Delete workspace (must not be current, must be empty)
terraform workspace select default
terraform workspace delete dev
```

---

### Using Workspace Name in Config

```hcl
# main.tf — using terraform.workspace
resource "aws_instance" "web" {
  # Different size per environment
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

resource "aws_s3_bucket" "app" {
  # Unique bucket name per workspace
  bucket = "myapp-${terraform.workspace}-assets-12345"
}
```

---

### Common Beginner Mistakes

```bash
# ❌ MISTAKE 1: Forgetting to switch workspaces before applying
terraform workspace select prod   # ALWAYS verify before apply
terraform workspace show          # Confirm you're on right workspace
terraform apply                   # NOW apply

# ❌ MISTAKE 2: Using default workspace for real work
# "default" workspace = no isolation safety net
# Always create named workspaces: dev, staging, prod

# ❌ MISTAKE 3: Assuming workspaces = complete isolation
# Workspaces share the SAME:
# - backend configuration
# - provider credentials
# - IAM permissions
# They only isolate STATE

# ❌ MISTAKE 4: Using workspaces for different AWS accounts
# Workspaces can't switch credentials per workspace
# For different accounts: use directory separation

# ❌ MISTAKE 5: Deleting a workspace without destroying resources
# terraform workspace delete dev  ← state deleted, resources STILL EXIST in AWS
# Always: terraform destroy THEN workspace delete
```

---

## 🟡 INTERMEDIATE LEVEL

### Workspaces — Internal Mechanics

```
HOW WORKSPACE STATE IS STORED:
─────────────────────────────────────────────────────────────────
Backend: local
  terraform.tfstate           ← default workspace
  terraform.tfstate.d/
  └── dev/
      └── terraform.tfstate   ← dev workspace

Backend: S3
  <key>                       ← default workspace
  env:/<workspace>/<key>      ← named workspaces

Example with key = "services/app/terraform.tfstate":
  s3://my-bucket/services/app/terraform.tfstate         ← default
  s3://my-bucket/env:/dev/services/app/terraform.tfstate    ← dev
  s3://my-bucket/env:/prod/services/app/terraform.tfstate   ← prod
```

---

### Workspace-Based Configuration Patterns

```hcl
# Pattern 1: Lookup table (cleanest approach)
locals {
  # All environment configs in one place
  workspace_config = {
    default = {
      instance_type   = "t3.micro"
      instance_count  = 1
      db_class        = "db.t3.micro"
      multi_az        = false
      retention_days  = 1
      alarm_enabled   = false
    }
    dev = {
      instance_type   = "t3.micro"
      instance_count  = 1
      db_class        = "db.t3.micro"
      multi_az        = false
      retention_days  = 7
      alarm_enabled   = false
    }
    staging = {
      instance_type   = "t3.small"
      instance_count  = 2
      db_class        = "db.t3.small"
      multi_az        = false
      retention_days  = 14
      alarm_enabled   = true
    }
    prod = {
      instance_type   = "t3.large"
      instance_count  = 5
      db_class        = "db.t3.large"
      multi_az        = true
      retention_days  = 90
      alarm_enabled   = true
    }
  }

  # Get config for current workspace (with fallback)
  config = lookup(
    local.workspace_config,
    terraform.workspace,
    local.workspace_config["default"]
  )
}

# Use config values
resource "aws_instance" "web" {
  count         = local.config.instance_count
  instance_type = local.config.instance_type

  tags = {
    Environment = terraform.workspace
    Name        = "web-${terraform.workspace}-${count.index + 1}"
  }
}

resource "aws_db_instance" "main" {
  instance_class = local.config.db_class
  multi_az       = local.config.multi_az

  backup_retention_period = local.config.retention_days
}
```

```hcl
# Pattern 2: Workspace-specific tfvars files
# Structure:
# ├── main.tf
# ├── variables.tf
# ├── dev.tfvars
# ├── staging.tfvars
# └── prod.tfvars

# Apply with specific vars file:
# terraform workspace select dev
# terraform apply -var-file="dev.tfvars"

# dev.tfvars
instance_type  = "t3.micro"
instance_count = 1
environment    = "dev"

# prod.tfvars
instance_type  = "t3.large"
instance_count = 5
environment    = "prod"
```

```hcl
# Pattern 3: Conditional resource creation per workspace
locals {
  is_prod    = terraform.workspace == "prod"
  is_dev     = terraform.workspace == "dev"
  is_staging = terraform.workspace == "staging"
}

# Only create WAF in prod and staging
resource "aws_wafv2_web_acl" "main" {
  count = local.is_dev ? 0 : 1
  name  = "web-acl-${terraform.workspace}"
  scope = "REGIONAL"
  # ...
}

# Only create detailed monitoring in prod
resource "aws_cloudwatch_dashboard" "main" {
  count          = local.is_prod ? 1 : 0
  dashboard_name = "production-overview"
  # ...
}

# Different backup schedules
resource "aws_backup_plan" "main" {
  name = "backup-${terraform.workspace}"

  rule {
    rule_name         = "daily-backup"
    target_vault_name = aws_backup_vault.main.name
    schedule          = local.is_prod ? "cron(0 1 * * ? *)" : "cron(0 3 ? * 1 *)"
    # prod: daily at 1am | others: weekly on Monday at 3am
  }
}
```

---

### Directory-Based Environment Separation (Recommended)

This is the **enterprise-grade approach** — separate directory per environment with shared modules.

```
my-infrastructure/
├── modules/                     ← Shared reusable modules
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── monitoring/
│
├── environments/
│   ├── dev/
│   │   ├── backend.tf           ← Dev-specific state backend
│   │   ├── main.tf              ← Calls modules w/ dev config
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars     ← Dev-specific values
│   │
│   ├── staging/
│   │   ├── backend.tf
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   │
│   └── prod/
│       ├── backend.tf
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
│
└── global/                      ← Cross-environment resources
    ├── iam/                     ← IAM roles, policies
    ├── dns/                     ← Route53 hosted zones
    └── state/                   ← State bucket, DynamoDB
```

```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-tf-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"

    # Dev uses dev AWS account role
    role_arn       = "arn:aws:iam::111111111111:role/TerraformDev"
  }
}
```

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "mycompany-tf-state"
    key            = "environments/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"

    # Prod uses SEPARATE prod AWS account role
    role_arn       = "arn:aws:iam::999999999999:role/TerraformProd"
  }
}
```

```hcl
# environments/dev/main.tf
locals {
  environment = "dev"
  region      = "us-east-1"

  tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Team        = "DevOps"
  }
}

# ── Networking ─────────────────────────────────────────────────────
module "networking" {
  source = "../../modules/networking"

  name        = "myapp"
  environment = local.environment
  vpc_cidr    = "10.0.0.0/16"
  az_count    = 2              # Less AZs in dev (cheaper)
  tags        = local.tags
}

# ── Application ────────────────────────────────────────────────────
module "web_app" {
  source = "../../modules/web-app"

  name               = "myapp"
  environment        = local.environment
  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.public_subnet_ids
  private_subnet_ids = module.networking.private_subnet_ids

  instance_type = "t3.micro"   # Small in dev
  min_instances = 1
  max_instances = 2
  enable_https  = false        # No HTTPS in dev
  tags          = local.tags
}
```

```hcl
# environments/prod/main.tf
locals {
  environment = "prod"
  region      = "us-east-1"

  tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Team        = "DevOps"
    CostCenter  = "Engineering"
  }
}

# ── Networking ─────────────────────────────────────────────────────
module "networking" {
  source = "../../modules/networking"

  name        = "myapp"
  environment = local.environment
  vpc_cidr    = "10.1.0.0/16"  # Different CIDR from dev!
  az_count    = 3              # More AZs in prod (HA)
  tags        = local.tags
}

# ── Application ────────────────────────────────────────────────────
module "web_app" {
  source = "../../modules/web-app"

  name               = "myapp"
  environment        = local.environment
  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.public_subnet_ids
  private_subnet_ids = module.networking.private_subnet_ids

  instance_type   = "t3.large"    # Bigger in prod
  min_instances   = 3             # HA in prod
  max_instances   = 10
  enable_https    = true          # HTTPS in prod
  certificate_arn = var.certificate_arn

  alarms = {
    cpu_threshold    = 80
    memory_threshold = 85
    sns_topic_arn    = module.monitoring.alert_topic_arn
  }

  tags = local.tags
}
```

---

### Workspaces vs Directory Separation — Decision Matrix

```
USE WORKSPACES WHEN:              USE DIRECTORIES WHEN:
──────────────────────────────    ──────────────────────────────────
Same AWS account for all envs     Different AWS accounts per env
Same IAM permissions for all      Different IAM roles per env
Environments are truly identical  Environments differ significantly
Small team (1-3 people)           Larger teams with env ownership
Learning/personal projects        Enterprise production systems
Temp feature branch infra         Long-lived stable environments
Identical provider config         Different provider configs needed

NEVER use workspaces when:
❌ Environments span multiple AWS accounts
❌ Different teams own different environments
❌ Environments need different Terraform versions
❌ You need strict change control per environment
❌ Compliance requires environment isolation
```

---

### Environment Promotion Pattern

```
CODE PROMOTION FLOW:
─────────────────────────────────────────────────────────────────

Developer pushes code
        │
        ▼
   ┌─────────┐
   │   DEV   │ ← Auto-apply on push to feature branch
   │         │   Fast feedback, experiments OK
   └────┬────┘
        │ Manual approval / PR merge
        ▼
   ┌─────────┐
   │ STAGING │ ← Auto-apply on merge to main branch
   │         │   Integration tests run here
   └────┬────┘
        │ Manual approval + change window
        ▼
   ┌─────────┐
   │  PROD   │ ← Apply ONLY after explicit approval
   │         │   Real users, handle with care
   └─────────┘


WHAT'S PROMOTED: Code (modules), NOT state
  Dev applies → creates dev resources (dev state)
  Staging applies same code → creates staging resources (staging state)
  Prod applies same code → creates prod resources (prod state)
  Each has completely independent state and resources
```

---

### Interview-Focused Points 🎯

1. **What's the main limitation of Terraform workspaces?** They share the same backend config, provider credentials, and IAM permissions. They only isolate state — not accounts, not permissions, not provider config.

2. **When would directory separation be better than workspaces?** When environments use different AWS accounts, different IAM roles, different provider configs, or are managed by different teams.

3. **What happens to cloud resources when you delete a workspace?** Nothing — cloud resources continue to exist. Only the state file is deleted. You lose Terraform's ability to manage those resources.

4. **How do you reference the current workspace in code?** `terraform.workspace` — a built-in string value always available.

5. **What is the `default` workspace?** The workspace that exists automatically in every Terraform project. You can't delete it. Best practice: don't use it for real work — create named workspaces.

---

## 🔴 ADVANCED LEVEL

### Multi-Account AWS Architecture — Production Pattern

```
ENTERPRISE MULTI-ACCOUNT SETUP:
─────────────────────────────────────────────────────────────────
AWS Organization
├── Management Account (billing, org policies)
│
├── Security Account (CloudTrail, GuardDuty, SecurityHub)
│   └── Terraform state bucket lives here
│
├── Shared Services Account (DNS, CI/CD, artifact registry)
│
├── Development Account (111111111111)
│   └── Dev environment resources
│
├── Staging Account (222222222222)
│   └── Staging environment resources
│
└── Production Account (999999999999)
    └── Production resources
    └── Stricter SCPs (Service Control Policies)
    └── Requires MFA for human access
    └── Only CI/CD role can apply Terraform
```

```hcl
# providers.tf — Multi-account provider setup
# Root/shared credentials come from CI/CD OIDC role

provider "aws" {
  alias  = "dev"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::111111111111:role/TerraformExecutionRole"
    session_name = "Terraform-Dev-${formatdate("YYYYMMDDHHmmss", timestamp())}"
  }
}

provider "aws" {
  alias  = "staging"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::222222222222:role/TerraformExecutionRole"
    session_name = "Terraform-Staging-${formatdate("YYYYMMDDHHmmss", timestamp())}"
  }
}

provider "aws" {
  alias  = "prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::999999999999:role/TerraformExecutionRole"
    session_name = "Terraform-Prod-${formatdate("YYYYMMDDHHmmss", timestamp())}"
  }
}
```

---

### Terragrunt — DRY Environment Management

Terragrunt is a thin wrapper around Terraform that solves the DRY problem for multi-environment setups. While not core Terraform, it's widely used in production.

```
WITHOUT TERRAGRUNT:
environments/dev/backend.tf     ← 90% identical
environments/staging/backend.tf ← to each other
environments/prod/backend.tf

WITH TERRAGRUNT:
environments/
├── terragrunt.hcl              ← Root config (shared)
├── dev/
│   └── terragrunt.hcl          ← Inherits + overrides
├── staging/
│   └── terragrunt.hcl
└── prod/
    └── terragrunt.hcl
```

```hcl
# environments/terragrunt.hcl (root)
locals {
  account_vars = read_terragrunt_config(
    find_in_parent_folders("account.hcl")
  )
  env_vars = read_terragrunt_config(
    find_in_parent_folders("env.hcl")
  )

  account_id  = local.account_vars.locals.aws_account_id
  environment = local.env_vars.locals.environment
}

# Generate backend config dynamically
generate "backend" {
  path      = "backend.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
terraform {
  backend "s3" {
    bucket         = "mycompany-tf-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    role_arn       = "arn:aws:iam::${local.account_id}:role/TerraformRole"
  }
}
EOF
}

# Common provider config
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<EOF
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::${local.account_id}:role/TerraformRole"
  }
  default_tags {
    tags = {
      Environment = "${local.environment}"
      ManagedBy   = "Terraform"
    }
  }
}
EOF
}
```

---

### Advanced Workspace Patterns

**Pattern: Workspace-Aware Remote State Reference**

```hcl
# When using workspaces + remote state data source
# You need to reference the right workspace's state

data "terraform_remote_state" "networking" {
  backend   = "s3"
  workspace = terraform.workspace   # ← Use current workspace's state

  config = {
    bucket = "mycompany-tf-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Now reads networking state from same workspace
# dev workspace → reads dev networking state
# prod workspace → reads prod networking state
```

**Pattern: Feature Branch Workspaces**

```bash
# Create isolated environment for each feature branch
BRANCH=$(git branch --show-current | tr '/' '-' | tr '[:upper:]' '[:lower:]')
terraform workspace new "feature-${BRANCH}" || terraform workspace select "feature-${BRANCH}"
terraform apply -var="environment=dev" -var-file="dev.tfvars"

# After PR merge, clean up
terraform destroy -var="environment=dev" -var-file="dev.tfvars"
terraform workspace select default
terraform workspace delete "feature-${BRANCH}"
```

**Pattern: Workspace Validation Guard**

```hcl
# Prevent accidental apply to prod from wrong context
# Add to main.tf

locals {
  # Guard: only allow prod workspace in CI/CD (has specific env var)
  is_ci = can(env("CI"))

  guard = (
    terraform.workspace == "prod" && !local.is_ci
      ? tobool("ERROR: Cannot apply prod workspace outside CI/CD!")
      : true
  )
}

# Precondition guard on critical resource
resource "aws_db_instance" "main" {
  # ...

  lifecycle {
    precondition {
      condition = !(
        terraform.workspace == "prod" &&
        var.db_instance_class == "db.t3.micro"
      )
      error_message = "Production database cannot use t3.micro. Use at least db.t3.large."
    }
  }
}
```

---

### Environment-Specific CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.6.0"

permissions:
  id-token: write    # For OIDC auth to AWS
  contents: read
  pull-requests: write

jobs:
  # ── DEV: Auto-apply on push to develop ──────────────────────────
  terraform-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    environment: dev
    if: github.ref == 'refs/heads/develop'

    defaults:
      run:
        working-directory: environments/dev

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Plan
        run: terraform plan -out=tfplan

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan


  # ── STAGING: Apply on merge to main ───────────────────────────
  terraform-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment: staging
    if: github.ref == 'refs/heads/main'

    defaults:
      run:
        working-directory: environments/staging

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::222222222222:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        id: plan
        run: terraform plan -out=tfplan

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Staging Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan


  # ── PROD: Manual approval required ────────────────────────────
  terraform-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment: production    # ← GitHub environment with required reviewers
    needs: [terraform-staging] # Must pass staging first
    if: github.ref == 'refs/heads/main'

    defaults:
      run:
        working-directory: environments/prod

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::999999999999:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: |
          terraform plan \
            -out=tfplan \
            -detailed-exitcode    # Exit 2 = changes detected
        continue-on-error: true   # Exit 2 isn't failure

      - name: Save Plan Summary
        run: terraform show -no-color tfplan > plan_summary.txt

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: prod-plan
          path: environments/prod/tfplan

      # ← GitHub pauses here for manual approval
      # (configured in GitHub environment settings)

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

---

### State Isolation Strategies

```
STRATEGY 1: Workspace-per-environment (simple)
─────────────────────────────────────────────────
Same backend, different state keys
Good for: small teams, same account, similar envs

s3://state-bucket/
├── env:/dev/app/terraform.tfstate
├── env:/staging/app/terraform.tfstate
└── env:/prod/app/terraform.tfstate


STRATEGY 2: Directory-per-environment (recommended)
─────────────────────────────────────────────────────
Different backend config per directory
Good for: multiple accounts, team ownership

s3://state-bucket/
├── environments/
│   ├── dev/terraform.tfstate
│   ├── staging/terraform.tfstate
│   └── prod/terraform.tfstate


STRATEGY 3: Account-per-environment (enterprise)
──────────────────────────────────────────────────
Different S3 buckets in different accounts
Good for: strict compliance, multi-account orgs

Account 111: s3://dev-state-bucket/terraform.tfstate
Account 222: s3://staging-state-bucket/terraform.tfstate
Account 999: s3://prod-state-bucket/terraform.tfstate


STRATEGY 4: Stack-based separation (complex infra)
────────────────────────────────────────────────────
Separate state per infrastructure layer
Good for: large teams, independent lifecycle per layer

s3://state-bucket/prod/
├── networking/terraform.tfstate     ← Networking team
├── databases/terraform.tfstate      ← DBA team
├── compute/terraform.tfstate        ← App team
└── monitoring/terraform.tfstate     ← SRE team
```

---

### Variable Management Per Environment

```hcl
# ── Strategy: Environment variable files ─────────────────────────

# environments/dev/terraform.tfvars
environment      = "dev"
aws_region       = "us-east-1"
vpc_cidr         = "10.0.0.0/16"
az_count         = 2
instance_type    = "t3.micro"
instance_count   = 1
db_class         = "db.t3.micro"
multi_az_db      = false
enable_waf       = false
enable_shield    = false
log_retention    = 7
alert_email      = "devops-dev@mycompany.com"
certificate_arn  = ""


# environments/prod/terraform.tfvars
environment      = "prod"
aws_region       = "us-east-1"
vpc_cidr         = "10.1.0.0/16"
az_count         = 3
instance_type    = "t3.large"
instance_count   = 5
db_class         = "db.r5.large"
multi_az_db      = true
enable_waf       = true
enable_shield    = true
log_retention    = 90
alert_email      = "oncall@mycompany.com"
certificate_arn  = "arn:aws:acm:us-east-1:999999999999:certificate/abc123"
```

```hcl
# ── Strategy: AWS SSM Parameter Store for dynamic config ──────────

# Store env-specific config in SSM (not in Git)
# /myapp/dev/db_password
# /myapp/prod/db_password

data "aws_ssm_parameter" "db_password" {
  name            = "/myapp/${var.environment}/db_password"
  with_decryption = true
}

data "aws_ssm_parameter" "app_secret_key" {
  name            = "/myapp/${var.environment}/app_secret_key"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

---

### Environment Drift Detection

```bash
# ── Scheduled drift detection (run as cron job) ───────────────────

#!/bin/bash
# drift-check.sh — Run for each environment

set -e
ENVIRONMENTS=("dev" "staging" "prod")
SLACK_WEBHOOK="https://hooks.slack.com/services/..."
DRIFT_FOUND=false

for ENV in "${ENVIRONMENTS[@]}"; do
  echo "Checking drift in $ENV..."

  cd "environments/$ENV"

  # Initialize without prompts
  terraform init -input=false > /dev/null 2>&1

  # Check for drift (exit code 2 = drift, 0 = clean, 1 = error)
  terraform plan \
    -refresh-only \
    -detailed-exitcode \
    -input=false \
    -out="/tmp/${ENV}-drift.tfplan" \
    > "/tmp/${ENV}-drift.log" 2>&1

  EXIT_CODE=$?

  if [ $EXIT_CODE -eq 2 ]; then
    echo "⚠️  DRIFT DETECTED in $ENV!"
    DRIFT_FOUND=true

    # Send Slack alert
    PLAN_SUMMARY=$(terraform show -no-color /tmp/${ENV}-drift.tfplan | head -50)
    curl -s -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      --data "{
        \"text\": \"⚠️ Infrastructure Drift Detected in *${ENV}*!\",
        \"attachments\": [{
          \"text\": \"\`\`\`${PLAN_SUMMARY}\`\`\`\",
          \"color\": \"warning\"
        }]
      }"
  elif [ $EXIT_CODE -eq 0 ]; then
    echo "✅ $ENV is clean — no drift"
  else
    echo "❌ Error checking $ENV"
  fi

  cd -
done

# Exit non-zero if any drift found (for CI alerting)
if [ "$DRIFT_FOUND" = true ]; then
  exit 1
fi
```

---

### Advanced Edge Cases

```hcl
# ── EDGE CASE 1: Workspace name in resource that requires uniqueness ─
# Problem: bucket names must be globally unique
resource "aws_s3_bucket" "app" {
  # ❌ This will conflict if workspace names are similar
  bucket = "myapp-${terraform.workspace}"

  # ✅ Add account ID to guarantee uniqueness
  bucket = "myapp-${terraform.workspace}-${data.aws_caller_identity.current.account_id}"
}


# ── EDGE CASE 2: Workspace-conditional depends_on ─────────────────
# In prod you have WAF, in dev you don't
# But your ALB listener optionally depends on WAF

resource "aws_lb_listener" "https" {
  # ...
  depends_on = compact([
    # Only include WAF dependency if it exists
    terraform.workspace == "prod" ? aws_wafv2_web_acl_association.main[0].id : ""
  ])
}


# ── EDGE CASE 3: Different regions per workspace ──────────────────
locals {
  workspace_regions = {
    dev     = "us-east-1"
    staging = "us-east-1"
    prod    = "us-west-2"   # Prod in different region for DR
  }
  region = lookup(local.workspace_regions, terraform.workspace, "us-east-1")
}

provider "aws" {
  region = local.region
}


# ── EDGE CASE 4: Reading outputs across workspace states ───────────
# Read staging state from prod workspace for comparison/migration

data "terraform_remote_state" "staging_reference" {
  backend   = "s3"
  workspace = "staging"   # Explicitly read staging state

  config = {
    bucket = "mycompany-tf-state"
    key    = "app/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use staging AMI ID as reference in prod
locals {
  # Take the same AMI that staging validated
  validated_ami = data.terraform_remote_state.staging_reference.outputs.ami_id
}
```

---

### Production Best Practices

```hcl
# ✅ 1. Never use terraform.workspace == "default" for real workloads
#       Always create named workspaces

# ✅ 2. Add workspace validation at top of main.tf
locals {
  allowed_workspaces = ["dev", "staging", "prod"]

  _workspace_check = (
    !contains(local.allowed_workspaces, terraform.workspace)
      ? tobool("Unknown workspace '${terraform.workspace}'. Allowed: ${join(", ", local.allowed_workspaces)}")
      : true
  )
}


# ✅ 3. Always output the workspace name for clarity
output "current_workspace" {
  description = "The Terraform workspace this was deployed with"
  value       = terraform.workspace
}

output "deployment_info" {
  description = "Deployment metadata"
  value = {
    workspace   = terraform.workspace
    region      = var.aws_region
    deployed_at = timestamp()
  }
}


# ✅ 4. Protect prod with prevent_destroy on all stateful resources
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = terraform.workspace == "prod" ? true : false
    # ← Dynamically set based on workspace (TF doesn't support this natively)
    # ← In practice: hardcode prevent_destroy = true in prod directory
  }
}
# Better approach with directory separation:
# environments/prod/main.tf  ← prevent_destroy = true hardcoded
# environments/dev/main.tf   ← prevent_destroy = false


# ✅ 5. Use different VPC CIDRs per environment (for VPC peering)
#       Dev:     10.0.0.0/16
#       Staging: 10.1.0.0/16
#       Prod:    10.2.0.0/16
#       Mgmt:    10.3.0.0/16
```

---

## 📁 CODE WRITING GUIDANCE

### Complete Production Structure

```
infrastructure/
│
├── _bootstrap/                  ← Run ONCE to create state infrastructure
│   ├── main.tf                  ← S3 bucket, DynamoDB, KMS
│   ├── outputs.tf
│   └── README.md
│
├── modules/                     ← Reusable modules (Topic 5)
│   ├── networking/
│   ├── compute/
│   ├── database/
│   └── monitoring/
│
├── environments/
│   │
│   ├── dev/
│   │   ├── main.tf              ← Module calls
│   │   ├── variables.tf         ← Variable declarations
│   │   ├── outputs.tf
│   │   ├── backend.tf           ← Dev state backend
│   │   ├── terraform.tfvars     ← Dev values (safe to commit)
│   │   └── secrets.tfvars       ← Dev secrets (GITIGNORED)
│   │
│   ├── staging/
│   │   └── ... (same structure)
│   │
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── backend.tf
│       ├── terraform.tfvars     ← Prod values
│       └── secrets.tfvars       ← Prod secrets (GITIGNORED)
│
├── global/
│   ├── iam/                     ← IAM roles for Terraform CI/CD
│   ├── dns/                     ← Route53 base zones
│   └── network-firewall/        ← Cross-env firewall rules
│
├── scripts/
│   ├── drift-check.sh           ← Scheduled drift detection
│   ├── apply-env.sh             ← Safe apply helper script
│   └── destroy-env.sh           ← Safe destroy helper
│
├── .github/
│   └── workflows/
│       ├── terraform-dev.yml
│       ├── terraform-staging.yml
│       └── terraform-prod.yml
│
└── .gitignore
```

```bash
# scripts/apply-env.sh — Safe apply with workspace/dir validation
#!/bin/bash
set -euo pipefail

ENV=$1
VALID_ENVS=("dev" "staging" "prod")

# Validate environment argument
if [[ ! " ${VALID_ENVS[@]} " =~ " ${ENV} " ]]; then
  echo "❌ Invalid environment: $ENV"
  echo "Usage: $0 [dev|staging|prod]"
  exit 1
fi

# Extra confirmation for prod
if [ "$ENV" == "prod" ]; then
  echo "⚠️  You are about to apply to PRODUCTION"
  read -p "Type 'yes-prod' to confirm: " CONFIRM
  if [ "$CONFIRM" != "yes-prod" ]; then
    echo "Aborted."
    exit 1
  fi
fi

cd "environments/$ENV"

echo "🔧 Initializing Terraform for $ENV..."
terraform init -input=false

echo "📋 Planning changes for $ENV..."
terraform plan \
  -var-file="terraform.tfvars" \
  -out="tfplan" \
  -input=false

echo ""
read -p "Apply these changes to $ENV? (yes/no): " APPLY_CONFIRM
if [ "$APPLY_CONFIRM" == "yes" ]; then
  terraform apply -auto-approve tfplan
  echo "✅ Apply complete for $ENV"
else
  echo "Aborted."
  rm -f tfplan
fi
```

---

## 🧪 HANDS-ON LAB

### Exercise: Multi-Environment Setup with Directory Separation

**Goal:** Build a complete multi-environment infrastructure with:
1. A shared `modules/s3-static-site` module (creates S3 bucket + website config)
2. Three environments: `dev`, `staging`, `prod`
3. Each environment has its own `backend.tf` and `terraform.tfvars`
4. Dev uses minimal config, prod uses enhanced config
5. Add a workspace validation guard to reject unknown workspaces
6. All environments share the same module, configured differently

**Expected output per environment:**
```bash
# Dev
Outputs:
environment    = "dev"
website_url    = "http://myapp-dev-123456789.s3-website-us-east-1.amazonaws.com"
bucket_name    = "myapp-dev-123456789"

# Prod
Outputs:
environment    = "prod"
website_url    = "http://myapp-prod-123456789.s3-website-us-east-1.amazonaws.com"
bucket_name    = "myapp-prod-123456789"
versioning     = "Enabled"
```

**Try it yourself first!**

---

### ✅ Solution

```hcl
# modules/s3-static-site/variables.tf
variable "name"        { type = string }
variable "environment" { type = string }

variable "versioning_enabled" {
  type    = bool
  default = false
}

variable "lifecycle_days" {
  type        = number
  description = "Days before old versions expire. 0 = disabled."
  default     = 0
}

variable "tags" {
  type    = map(string)
  default = {}
}
```

```hcl
# modules/s3-static-site/main.tf
data "aws_caller_identity" "current" {}

locals {
  bucket_name = "${var.name}-${var.environment}-${data.aws_caller_identity.current.account_id}"
  common_tags = merge({ ManagedBy = "Terraform", Environment = var.environment }, var.tags)
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name
  tags   = local.common_tags
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_website_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  index_document { suffix = "index.html" }
  error_document { key    = "error.html"  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "this" {
  bucket     = aws_s3_bucket.this.id
  depends_on = [aws_s3_bucket_public_access_block.this]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = "*"
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.this.arn}/*"
    }]
  })
}

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  count  = var.lifecycle_days > 0 ? 1 : 0
  bucket = aws_s3_bucket.this.id

  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    noncurrent_version_expiration {
      noncurrent_days = var.lifecycle_days
    }
  }
}
```

```hcl
# modules/s3-static-site/outputs.tf
output "bucket_name"    { value = aws_s3_bucket.this.id }
output "bucket_arn"     { value = aws_s3_bucket.this.arn }
output "website_url" {
  value = "http://${aws_s3_bucket_website_configuration.this.website_endpoint}"
}
output "versioning" {
  value = aws_s3_bucket_versioning.this.versioning_configuration[0].status
}
```

```hcl
# environments/dev/backend.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "mycompany-tf-state"
    key            = "environments/dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = { Environment = "dev", ManagedBy = "Terraform" }
  }
}
```

```hcl
# environments/dev/main.tf
module "website" {
  source = "../../modules/s3-static-site"

  name               = var.app_name
  environment        = var.environment
  versioning_enabled = false       # No versioning in dev
  lifecycle_days     = 0           # No lifecycle rules
  tags               = var.tags
}
```

```hcl
# environments/dev/terraform.tfvars
app_name    = "myapp"
environment = "dev"
tags = {
  Team    = "DevOps"
  Owner   = "dev-team"
}
```

```hcl
# environments/prod/main.tf
module "website" {
  source = "../../modules/s3-static-site"

  name               = var.app_name
  environment        = var.environment
  versioning_enabled = true      # Versioning in prod
  lifecycle_days     = 90        # Expire old versions after 90 days
  tags               = var.tags
}
```

```hcl
# environments/prod/terraform.tfvars
app_name    = "myapp"
environment = "prod"
tags = {
  Team       = "DevOps"
  Owner      = "sre-team"
  CostCenter = "Engineering"
}
```

```hcl
# environments/dev/outputs.tf  (same for prod)
output "environment" { value = var.environment }
output "website_url" { value = module.website.website_url }
output "bucket_name" { value = module.website.bucket_name }
output "versioning"  { value = module.website.versioning  }
```

```bash
# Apply dev
cd environments/dev
terraform init
terraform apply -var-file="terraform.tfvars"

# Apply prod
cd environments/prod
terraform init
terraform apply -var-file="terraform.tfvars"
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| Workspace | Named state isolation within same backend — same code, separate state |
| `terraform.workspace` | Built-in string with current workspace name |
| `default` workspace | Always exists, can't delete — don't use for real work |
| Directory separation | Separate dir per env — full isolation, different backends possible |
| Workspaces limit | Share same backend config and provider credentials |
| Environment promotion | Promote CODE, not state — each env applies same code independently |
| Drift detection | `terraform plan -refresh-only -detailed-exitcode` — exit 2 = drift |
| Multi-account | Use directory separation + assume_role per account |
| Workspace validation | Guard block using `tobool()` trick to error on invalid workspace |
| State isolation | Workspace < Directory < Separate Account (increasing isolation) |

---

### Quick Command Reference

```bash
# Workspace commands
terraform workspace show              # Current workspace
terraform workspace list              # All workspaces (* = current)
terraform workspace new NAME          # Create + switch
terraform workspace select NAME       # Switch to existing
terraform workspace delete NAME       # Delete (must be empty + not current)

# Apply with environment context
cd environments/prod                  # Directory-based approach
terraform init
terraform plan  -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"

# Workspace-based approach
terraform workspace select prod
terraform apply -var-file="prod.tfvars"

# Drift detection (all envs)
terraform plan -refresh-only -detailed-exitcode
# Exit 0 = clean, Exit 2 = drift, Exit 1 = error

# Safe targeted apply
terraform apply -target=module.networking -var-file="terraform.tfvars"

# Check current workspace in config
# terraform.workspace  ← built-in string value

# Force unlock stuck state
terraform force-unlock LOCK_ID
```

---

### Top 10 Interview Questions

1. **What is a Terraform workspace?** A named slot for an independent state file within the same backend configuration. Multiple workspaces share backend config but have completely separate state.

2. **What are the limitations of workspaces?** They share the same backend, provider credentials, and IAM permissions. Only state is isolated — not accounts, not regions, not team permissions.

3. **When would you choose directory separation over workspaces?** When environments use different AWS accounts, different IAM roles, need different provider configurations, or are owned by different teams with different access levels.

4. **What is the `default` workspace?** The automatically created workspace in every Terraform project. It cannot be deleted. Best practice is to never use it for actual workloads — create named workspaces instead.

5. **How do you prevent someone from running `terraform apply` on prod accidentally?** Use GitHub environment protection rules requiring manual approval, add workspace validation guards in code, require MFA for prod AWS role assumption, and restrict CI/CD pipeline triggers.

6. **What is environment drift and how do you detect it?** When real cloud infrastructure differs from what Terraform expects in state. Detect with `terraform plan -refresh-only -detailed-exitcode` — exit code 2 means drift was found.

7. **How do you promote infrastructure changes from dev to prod?** You promote code (module changes, config changes) — not state. Each environment independently applies the same updated code, creating its own version of the infrastructure.

8. **What is the recommended CIDR strategy for multiple environments?** Use non-overlapping CIDRs per environment to enable VPC peering if needed: dev=10.0.0.0/16, staging=10.1.0.0/16, prod=10.2.0.0/16.

9. **How do secrets differ between environments in Terraform?** Use environment-specific SSM Parameter Store paths, separate secret files per environment (gitignored), or separate Secrets Manager entries per account, never hardcode secrets in tfvars that get committed.

10. **How do workspaces store state in S3?** Default workspace: at the configured `key` path directly. Named workspaces: at `env:/<workspace-name>/<key>`. For example, `prod` workspace with key `app/terraform.tfstate` stores at `env:/prod/app/terraform.tfstate`.

---
