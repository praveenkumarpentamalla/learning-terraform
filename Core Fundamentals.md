# 🏗️ TERRAFORM CORE FUNDAMENTALS: Complete Beginner to Advanced Guide

Let's start with the foundation of everything in Terraform:

---

# 📌 TOPIC 1: What is Terraform & Infrastructure as Code (IaC)

---

## 🟢 BEGINNER LEVEL

### What is Terraform in Simple Terms?

Imagine you're building a house. Traditionally, you'd call different contractors — one for electricity, one for plumbing, one for walls — and manually coordinate everything. If the house burns down, you'd have to start over and remember every detail.

**Terraform is like having a blueprint of that house stored in a file.** If anything breaks, you just re-run the blueprint and the exact same house gets rebuilt — automatically.

In the tech world, instead of a physical house, we're building **cloud infrastructure** — servers, databases, networks, storage buckets, etc.

---

### Why is Terraform Used?

Without Terraform (manual way):
- You click through AWS/Azure/GCP console to create resources
- Hard to track what was created
- Impossible to recreate exactly
- Team members can't collaborate on infra changes

With Terraform:
- Infrastructure is defined in **code files** (`.tf` files)
- Version controlled in Git
- Reproducible anywhere, anytime
- Team can review infra changes via Pull Requests
- Destroy and recreate with one command

---

### Real-World Analogy

| Concept | Real World | Terraform |
|---|---|---|
| Blueprint | House blueprint | `.tf` config files |
| Contractor | Builder who reads blueprint | Terraform CLI |
| Material supplier | Hardware store | AWS / Azure / GCP |
| Building permit | What's allowed to build | Provider permissions |
| Finished house | Physical house | Live cloud infrastructure |

---

### Core Concepts at a Glance

```
YOU write .tf files  →  Terraform reads them  →  Cloud Provider creates resources
    (Blueprint)              (Contractor)              (The actual house)
```

---

### Basic Syntax — HCL (HashiCorp Configuration Language)

Terraform uses **HCL** — a human-readable language. It looks like this:

```hcl
# This is a comment

# Block structure: block_type "type" "name" { ... }
resource "aws_instance" "my_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**Key syntax rules:**
- Blocks use `{}` curly braces
- Arguments use `=` for assignment
- Strings use `"double quotes"`
- Comments use `#` or `//`
- No semicolons needed

---

### Minimal Working Example

Let's create your first Terraform config that creates an S3 bucket on AWS:

**File: `main.tf`**
```hcl
# Tell Terraform we're using AWS
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure AWS region
provider "aws" {
  region = "us-east-1"
}

# Create an S3 bucket
resource "aws_s3_bucket" "my_first_bucket" {
  bucket = "my-first-terraform-bucket-12345"

  tags = {
    Name        = "My First Bucket"
    Environment = "Dev"
  }
}
```

**Run these commands:**
```bash
terraform init     # Downloads AWS provider plugin
terraform plan     # Shows what will be created (dry run)
terraform apply    # Actually creates the bucket
terraform destroy  # Deletes everything when done
```

---

### Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Duplicate resource names
resource "aws_s3_bucket" "my_bucket" { ... }
resource "aws_s3_bucket" "my_bucket" { ... }  # Error! Same name

# ❌ MISTAKE 2: Forgetting terraform init before plan/apply
# Always run init first when starting or adding providers

# ❌ MISTAKE 3: Hardcoding sensitive values
resource "aws_db_instance" "db" {
  password = "mypassword123"  # NEVER do this! Use variables or secrets manager
}

# ❌ MISTAKE 4: Wrong indentation (Terraform is forgiving but be consistent)
# Use 2 spaces, not tabs

# ❌ MISTAKE 5: Forgetting that S3 bucket names are globally unique
resource "aws_s3_bucket" "test" {
  bucket = "test"  # This is likely already taken by someone else globally
}
```

---

## 🟡 INTERMEDIATE LEVEL

### How Terraform Works Internally (The 3-Phase Cycle)

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   WRITE     │───▶│    PLAN     │───▶│    APPLY    │
│  .tf files  │    │  Diff check │    │ API calls   │
│             │    │  vs state   │    │ to provider │
└─────────────┘    └─────────────┘    └─────────────┘
                          │
                   ┌──────▼──────┐
                   │  STATE FILE │
                   │terraform.   │
                   │tfstate      │
                   └─────────────┘
```

**Phase 1 — Init:** Downloads provider plugins (like AWS SDK), sets up backend
**Phase 2 — Plan:** Compares your `.tf` files against the current state file, shows a diff
**Phase 3 — Apply:** Makes actual API calls to cloud provider to create/update/delete resources

---

### The State File — Terraform's Memory

The `terraform.tfstate` file is **critical**. It's a JSON file that tracks what Terraform has created.

```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_first_bucket",
      "instances": [
        {
          "attributes": {
            "bucket": "my-first-terraform-bucket-12345",
            "arn": "arn:aws:s3:::my-first-terraform-bucket-12345"
          }
        }
      ]
    }
  ]
}
```

**Why state matters:**
- Terraform compares state vs your config to know what changed
- Without state, Terraform would try to create everything fresh every time
- In teams, state must be stored remotely (S3, Terraform Cloud) — never in Git

---

### Terraform Block Types

```hcl
# 1. TERRAFORM BLOCK — global settings
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  # Remote backend (for teams)
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# 2. PROVIDER BLOCK — cloud provider config
provider "aws" {
  region  = "us-east-1"
  profile = "default"  # Uses ~/.aws/credentials
}

# 3. RESOURCE BLOCK — actual infrastructure
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# 4. DATA BLOCK — read existing resources (don't create)
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

# 5. VARIABLE BLOCK — input values
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

# 6. OUTPUT BLOCK — expose values after apply
output "server_ip" {
  value       = aws_instance.web.public_ip
  description = "The public IP of the web server"
}

# 7. LOCALS BLOCK — local computed values
locals {
  common_tags = {
    Team    = "DevOps"
    Project = "MyApp"
  }
}
```

---

### Variables — Making Config Reusable

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type    = number
  default = 1
}

variable "enable_monitoring" {
  type    = bool
  default = false
}

variable "tags" {
  type = map(string)
  default = {
    Owner = "DevOps Team"
  }
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 22]
}
```

```hcl
# main.tf — using variables
resource "aws_instance" "web" {
  instance_type = var.instance_type
  count         = var.instance_count
  
  tags = merge(var.tags, {
    Environment = var.environment
  })
}
```

```bash
# Passing variables at command line
terraform apply -var="environment=prod" -var="instance_count=3"

# Using a .tfvars file (recommended)
terraform apply -var-file="prod.tfvars"
```

```hcl
# prod.tfvars
environment      = "prod"
instance_count   = 3
enable_monitoring = true
```

---

### Best Practices at Intermediate Level

```hcl
# ✅ GOOD: Use locals to avoid repetition
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.name_prefix}-data"
  tags   = local.common_tags
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = local.common_tags
}
```

```hcl
# ✅ GOOD: Always pin provider versions
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allows 5.x but not 6.x
    }
  }
}

# ❌ BAD: No version pinning
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      # No version = could break when new major version releases
    }
  }
}
```

---

### Interview-Focused Points 🎯

1. **What is idempotency in Terraform?** Running `terraform apply` multiple times produces the same result — it won't create duplicate resources if they already exist in state.

2. **What's the difference between `terraform plan` and `terraform apply`?** Plan is a dry-run showing what *would* change. Apply actually makes those changes.

3. **What happens if you delete the state file?** Terraform loses track of what it created. Running apply would try to create everything again, causing conflicts with existing resources.

4. **What is a provider in Terraform?** A plugin that translates your HCL config into actual API calls to a cloud platform (AWS, Azure, GCP, etc.)

5. **Difference between `resource` and `data` blocks?** `resource` creates/manages infrastructure. `data` only reads existing infrastructure without managing it.

---

## 🔴 ADVANCED LEVEL

### Terraform Execution Deep Dive

When you run `terraform apply`, here's exactly what happens:

```
1. REFRESH PHASE
   └── Terraform calls cloud APIs to get real current state
   └── Updates local state with actual values

2. DIFF CALCULATION
   └── Compares: Desired config (.tf files) vs Current state
   └── Generates a dependency graph (DAG)
   └── Determines: CREATE / UPDATE / DESTROY / NO-OP per resource

3. DEPENDENCY GRAPH (DAG — Directed Acyclic Graph)
   └── Resources that reference others are dependent
   └── Independent resources are created in PARALLEL
   └── Dependent resources wait for their dependencies

4. APPLY PHASE
   └── Walks the dependency graph
   └── Makes API calls to cloud provider
   └── Updates state after each resource completes
```

**Dependency Graph Example:**
```
                    ┌──────────────┐
                    │    VPC       │ (created first)
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼────┐ ┌─────▼─────┐     │
       │  Subnet A │ │  Subnet B │     │
       └──────┬────┘ └─────┬─────┘     │
              │            │      ┌────▼─────────┐
              └────────────┘      │ Internet GW  │
                    │             └──────────────┘
              ┌─────▼──────┐
              │  EC2 Instance│ (created last — needs subnet)
              └─────────────┘
```

---

### Terraform Modules — Reusable Infrastructure

Modules are like functions in programming — write once, reuse anywhere.

**Module structure:**
```
modules/
└── ec2-web-server/
    ├── main.tf         # Resources
    ├── variables.tf    # Input variables
    ├── outputs.tf      # Output values
    └── README.md       # Documentation
```

```hcl
# modules/ec2-web-server/main.tf
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id
  
  user_data = <<-EOF
    #!/bin/bash
    apt-get update
    apt-get install -y nginx
    systemctl start nginx
  EOF

  tags = {
    Name = "${var.name}-web-server"
  }
}

resource "aws_security_group" "web_sg" {
  name   = "${var.name}-web-sg"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

```hcl
# modules/ec2-web-server/variables.tf
variable "name"          { type = string }
variable "ami_id"        { type = string }
variable "instance_type" { type = string; default = "t2.micro" }
variable "subnet_id"     { type = string }
variable "vpc_id"        { type = string }
```

```hcl
# modules/ec2-web-server/outputs.tf
output "instance_id"  { value = aws_instance.web.id }
output "public_ip"    { value = aws_instance.web.public_ip }
output "sg_id"        { value = aws_security_group.web_sg.id }
```

```hcl
# root main.tf — CALLING the module
module "web_server_dev" {
  source = "./modules/ec2-web-server"

  name          = "dev"
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  vpc_id        = aws_vpc.main.id
}

module "web_server_prod" {
  source = "./modules/ec2-web-server"

  name          = "prod"
  ami_id        = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.large"  # Bigger for prod
  subnet_id     = aws_subnet.public.id
  vpc_id        = aws_vpc.main.id
}
```

---

### Remote State & State Locking (Production Critical)

```hcl
# backend.tf — Store state in S3, lock with DynamoDB
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "prod/us-east-1/web-app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                      # Encrypt state at rest
    dynamodb_table = "terraform-state-lock"    # Prevents concurrent applies
    
    # For cross-account: specify role
    role_arn       = "arn:aws:iam::123456789:role/TerraformRole"
  }
}
```

```hcl
# Create the state bucket and lock table (do this ONCE manually or with a bootstrap script)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"
}

resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"  # Keep history of all state versions
  }
}

resource "aws_dynamodb_table" "state_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

---

### Advanced Patterns

**`count` vs `for_each` — Creating Multiple Resources**

```hcl
# count — simple repetition (index-based)
resource "aws_instance" "server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  tags = {
    Name = "server-${count.index}"  # server-0, server-1, server-2
  }
}

# ⚠️ Problem with count: if you remove middle item, 
# Terraform destroys and recreates everything after it

# for_each — map-based (key-stable)
variable "servers" {
  default = {
    web  = "t2.micro"
    api  = "t2.small"
    db   = "t2.medium"
  }
}

resource "aws_instance" "server" {
  for_each      = var.servers
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value
  
  tags = {
    Name = "server-${each.key}"  # server-web, server-api, server-db
  }
}

# ✅ for_each is safer — removing "api" only destroys that one instance
```

**Dynamic Blocks**

```hcl
variable "ingress_rules" {
  default = [
    { port = 80,  protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 443, protocol = "tcp", cidr = "0.0.0.0/0" },
    { port = 22,  protocol = "tcp", cidr = "10.0.0.0/8" },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

---

### Production Best Practices

```
✅ Always use remote state with locking
✅ Never commit .tfstate files to Git
✅ Use workspaces or separate state files per environment
✅ Pin ALL provider and module versions
✅ Use terraform fmt in CI/CD pipelines
✅ Use terraform validate before plan
✅ Implement OPA or Sentinel for policy-as-code
✅ Use -target sparingly (dangerous in production)
✅ Always review plan output before applying
✅ Use terraform import for existing resources before managing them
```

---

### Debugging Strategies

```bash
# Enable detailed logging
export TF_LOG=DEBUG      # DEBUG, INFO, WARN, ERROR, TRACE
export TF_LOG_PATH=./terraform.log
terraform apply

# Check state
terraform state list                          # List all resources in state
terraform state show aws_instance.web         # Show specific resource details
terraform state pull                          # Download current remote state

# Validate and format
terraform validate    # Check syntax
terraform fmt -check  # Check formatting (CI-friendly)
terraform fmt -recursive  # Fix all formatting

# Plan with detailed output
terraform plan -out=tfplan          # Save plan to file
terraform show tfplan               # Inspect saved plan
terraform apply tfplan              # Apply saved plan (no re-confirmation)

# Refresh state (sync with real cloud)
terraform refresh                   # Update state from real infrastructure

# Force unlock if state is stuck
terraform force-unlock LOCK_ID
```

---

## 📁 CODE WRITING GUIDANCE

### Professional File Structure

```
my-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf          # Module calls
│   │   ├── variables.tf     # Variable declarations
│   │   ├── terraform.tfvars # Actual values for dev
│   │   ├── backend.tf       # State backend config
│   │   └── outputs.tf       # Exposed outputs
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       ├── backend.tf
│       └── outputs.tf
│
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── .gitignore               # Must include *.tfstate, .terraform/
```

---

### Naming Conventions

```hcl
# Resource naming: snake_case, descriptive
resource "aws_instance" "web_server" { }          # ✅
resource "aws_instance" "WebServer" { }           # ❌ No PascalCase
resource "aws_instance" "ws" { }                  # ❌ Too abbreviated

# Variable naming: snake_case, descriptive
variable "instance_type" { }                      # ✅
variable "instanceType" { }                       # ❌ No camelCase

# Output naming: what it IS, not where it came from
output "web_server_public_ip" { }                 # ✅
output "aws_instance_web_public_ip" { }           # ❌ Too verbose

# Module naming: functional purpose
module "web_tier" { }                             # ✅
module "aws_ec2_module" { }                       # ❌ Don't include provider name

# File naming convention
# main.tf       — primary resources
# variables.tf  — all variable declarations
# outputs.tf    — all output declarations
# locals.tf     — complex local expressions
# data.tf       — all data sources
# backend.tf    — backend configuration
# providers.tf  — provider configurations
```

---

### Anti-Patterns to Avoid

```hcl
# ❌ ANTI-PATTERN 1: Giant monolithic main.tf with 2000 lines
# Split into logical files and modules

# ❌ ANTI-PATTERN 2: Hardcoded account IDs and regions
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
# Instead: use data sources or variables

# ❌ ANTI-PATTERN 3: Using terraform.workspace for env separation
# Workspaces share backend config, causing state management issues
# Instead: use separate directories per environment

# ❌ ANTI-PATTERN 4: resource "aws_instance" "this" everywhere
# "this" is meaningless; use descriptive names

# ❌ ANTI-PATTERN 5: No outputs from modules
# Always expose at minimum the ID and ARN of created resources

# ❌ ANTI-PATTERN 6: Sensitive data in outputs
output "db_password" {
  value = aws_db_instance.main.password
  # sensitive = true  ← Add this if you must output sensitive values
}

# ❌ ANTI-PATTERN 7: Using count for diverse resources (use for_each)
resource "aws_s3_bucket" "buckets" {
  count  = 3
  bucket = "bucket-${count.index}"  # If you delete bucket-1, bucket-2 becomes bucket-1
}
```

---

## 🧪 HANDS-ON LAB

### Exercise: Build a Complete Static Website Hosting Infrastructure

**Goal:** Create an S3-based static website with proper configuration using Terraform.

**Requirements:**
1. Create an S3 bucket with a unique name
2. Enable static website hosting
3. Set bucket policy for public read
4. Output the website URL

**Expected Output after `terraform apply`:**
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

website_url = "http://my-terraform-website-12345.s3-website-us-east-1.amazonaws.com"
```

**Try it yourself first, then see the solution below.**

---

### ✅ Solution

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# S3 Bucket
resource "aws_s3_bucket" "website" {
  bucket = "${var.project_name}-${var.environment}-${random_id.suffix.hex}"
  tags   = local.common_tags
}

# Random suffix to ensure unique bucket name
resource "random_id" "suffix" {
  byte_length = 4
}

# Enable static website hosting
resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

# Disable block public access (needed for static website)
resource "aws_s3_bucket_public_access_block" "website" {
  bucket                  = aws_s3_bucket.website.id
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

# Bucket policy for public read
resource "aws_s3_bucket_policy" "website" {
  bucket     = aws_s3_bucket.website.id
  depends_on = [aws_s3_bucket_public_access_block.website]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "PublicReadGetObject"
        Effect    = "Allow"
        Principal = "*"
        Action    = "s3:GetObject"
        Resource  = "${aws_s3_bucket.website.arn}/*"
      }
    ]
  })
}
```

```hcl
# variables.tf
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project_name" {
  type    = string
  default = "my-terraform-website"
}

variable "environment" {
  type    = string
  default = "dev"
}
```

```hcl
# locals.tf
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

```hcl
# outputs.tf
output "website_url" {
  description = "The URL of the static website"
  value       = "http://${aws_s3_bucket_website_configuration.website.website_endpoint}"
}

output "bucket_name" {
  description = "The name of the S3 bucket"
  value       = aws_s3_bucket.website.id
}
```

```hcl
# terraform.tf (providers and required_providers)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Summary |
|---|---|
| HCL | HashiCorp Configuration Language — human-readable config syntax |
| Provider | Plugin that talks to cloud APIs (AWS, Azure, GCP) |
| Resource | Actual infrastructure to create/manage |
| Data Source | Read-only reference to existing infrastructure |
| State File | Terraform's memory — tracks what it has created |
| Module | Reusable group of resources — like a function |
| Backend | Where state is stored (local or remote S3/TFC) |

---

### Quick Syntax Reference

```hcl
# Provider
provider "aws" { region = "us-east-1" }

# Resource
resource "TYPE" "NAME" { argument = value }

# Data source
data "TYPE" "NAME" { filter { } }

# Variable
variable "NAME" { type = string; default = "value" }

# Use variable
var.NAME

# Local
locals { key = "value" }

# Use local
local.key

# Output
output "NAME" { value = resource.type.name.attribute }

# Reference resource attribute
resource_type.resource_name.attribute

# String interpolation
"${var.name}-suffix"

# Module call
module "NAME" { source = "./path"; var1 = value }

# Use module output
module.NAME.output_name
```

---

### Essential Commands

```bash
terraform init          # Initialize, download providers
terraform validate      # Check syntax
terraform fmt           # Format code
terraform plan          # Preview changes
terraform apply         # Create/update infrastructure
terraform destroy       # Delete all resources
terraform state list    # List managed resources
terraform state show    # Show resource details
terraform output        # Show output values
terraform import        # Import existing resource into state
terraform taint         # Mark resource for recreation (deprecated: use -replace)
terraform apply -replace="aws_instance.web"  # Force recreation
```

---

### Top 10 Interview Questions

1. **What is IaC and why use Terraform over others?** Terraform is cloud-agnostic, uses declarative HCL, and has a massive ecosystem. Compared to CloudFormation (AWS-only) or Pulumi (imperative code), Terraform balances portability with simplicity.

2. **Explain the Terraform lifecycle.** Write → Init → Plan → Apply → (Destroy). State is maintained throughout.

3. **What is the state file and why is remote state important?** It's Terraform's source of truth. Remote state enables team collaboration and enables state locking to prevent concurrent applies.

4. **Difference between `count` and `for_each`?** `count` uses an index (fragile when items removed), `for_each` uses a map key (stable identity per resource).

5. **What is a Terraform module?** A reusable collection of resources that accepts input variables and produces outputs — like a function.

6. **How do you handle secrets in Terraform?** Use `sensitive = true` on variables, use AWS Secrets Manager or HashiCorp Vault, never hardcode in `.tf` files, never commit `.tfvars` files with secrets.

7. **What is `terraform import`?** Brings an existing cloud resource under Terraform management without recreating it.

8. **How do you prevent two engineers from applying at the same time?** State locking via DynamoDB (for S3 backend) or Terraform Cloud.

9. **What happens when a resource is deleted from your `.tf` file?** On next `apply`, Terraform will **destroy** that resource in the cloud — it detects the drift between config and state.

10. **What is a data source?** A `data` block reads information from the cloud without creating or managing it — used to reference externally created resources.

---

### .gitignore for Terraform Projects

```gitignore
# State files — NEVER commit these
*.tfstate
*.tfstate.*
*.tfstate.backup

# Terraform directory (downloaded providers)
.terraform/
.terraform.lock.hcl  # Debated — many teams DO commit this for reproducibility

# Variable files with secrets
*.tfvars
!example.tfvars      # Commit example without real values

# Plan files
*.tfplan

# Crash logs
crash.log
crash.*.log

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```
