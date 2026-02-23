# Terraform HCL Complete Guide: From Beginner to Production-Ready

## 🚀 TOPIC 1: Introduction to Terraform & HCL Basics

---

# 📘 BEGINNER LEVEL

## What is Terraform & HCL?

**In simple terms:** Terraform is a tool that lets you create, manage, and destroy infrastructure (servers, databases, networks) by writing code instead of clicking buttons in a web console.

**HCL (HashiCorp Configuration Language)** is the language you use to write that code. It's designed to be human-readable while also being machine-parseable.

**Why use it?**
- Instead of manually clicking through AWS/Azure/GCP consoles, you write code once and Terraform handles everything
- Your infrastructure becomes **version-controlled**, **repeatable**, and **auditable**
- A team of 50 engineers can all work with the same infrastructure safely

**Real-World Analogy:**
> Think of Terraform like a **blueprint for a house**. An architect writes a blueprint once. The construction team can build the exact same house 10 times in 10 different cities using that same blueprint. If you want to add a room, you update the blueprint and rebuild. Terraform is your infrastructure blueprint.

---

## Basic Syntax — The Building Blocks

HCL has a very clean, readable syntax. Everything is built around **blocks**.

```hcl
# This is a comment

# Basic block structure
<BLOCK_TYPE> "<BLOCK_LABEL>" "<BLOCK_NAME>" {
  # Arguments go here
  argument_name = "argument_value"
}
```

**The 5 fundamental building blocks:**

```hcl
# 1. TERRAFORM BLOCK — Configures Terraform itself
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 2. PROVIDER BLOCK — Tells Terraform WHICH cloud/platform to use
provider "aws" {
  region = "us-east-1"
}

# 3. RESOURCE BLOCK — The actual infrastructure you want to create
resource "aws_instance" "my_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# 4. VARIABLE BLOCK — Input values (makes code reusable)
variable "instance_type" {
  description = "The size of the EC2 instance"
  type        = string
  default     = "t2.micro"
}

# 5. OUTPUT BLOCK — Values to display after Terraform runs
output "server_ip" {
  description = "The public IP of the server"
  value       = aws_instance.my_server.public_ip
}
```

---

## Minimal Working Example

Let's create the **simplest possible Terraform configuration** — a local file (no cloud account needed!).

```hcl
# main.tf

terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "hello_world" {
  filename = "${path.module}/hello.txt"
  content  = "Hello, Terraform! I am learning HCL."
}

output "file_path" {
  value = local_file.hello_world.filename
}
```

**To run this:**
```bash
terraform init      # Downloads the provider
terraform plan      # Shows what WILL happen (dry run)
terraform apply     # Actually creates the file
terraform destroy   # Removes everything
```

---

## Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Forgetting to run `terraform init` before apply
# Always init first when starting or adding providers

# ❌ MISTAKE 2: Wrong block syntax (missing quotes on labels)
resource aws_instance my_server {   # WRONG
resource "aws_instance" "my_server" {  # CORRECT

# ❌ MISTAKE 3: Using wrong value types
variable "count_value" {
  default = "3"   # ❌ This is a string, not a number
  default = 3     # ✅ This is a number
}

# ❌ MISTAKE 4: Hardcoding sensitive values
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"  # ❌ NEVER do this
  secret_key = "wJalrXUtnFEMI/K7MDENG" # ❌ Exposed in git!
}
# ✅ Use environment variables: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

# ❌ MISTAKE 5: Not reading the plan before applying
terraform apply  # ❌ Always check the plan first!
terraform plan   # ✅ Read this carefully
terraform apply  # ✅ Then apply
```

---

# 📗 INTERMEDIATE LEVEL

## How Terraform Works Internally

Understanding the **Terraform execution cycle** is critical for production work:

```
Write HCL Code
      ↓
terraform init
  → Downloads providers (plugins)
  → Sets up backend (where state is stored)
  → Creates .terraform/ directory
      ↓
terraform plan
  → Reads current state (.tfstate file)
  → Calls provider APIs to get real-world state
  → Computes DIFF (desired vs actual)
  → Produces execution plan
      ↓
terraform apply
  → Executes the plan
  → Calls provider APIs to create/update/delete resources
  → Updates .tfstate file with new reality
      ↓
terraform destroy
  → Creates a destroy plan
  → Deletes all managed resources
  → Clears state
```

**The State File — The Most Critical Concept:**

```hcl
# Terraform keeps a "terraform.tfstate" file that maps:
# Your HCL Code  →  Real-World Resource IDs

# Example state entry (simplified):
{
  "resource": "aws_instance.my_server",
  "id": "i-0a1b2c3d4e5f",        ← Real AWS Instance ID
  "public_ip": "54.23.111.45",   ← Real IP address
  "instance_type": "t2.micro"
}

# Without state, Terraform doesn't know what it has already created!
```

---

## All HCL Data Types with Examples

```hcl
# ============================================
# PRIMITIVE TYPES
# ============================================

variable "server_name" {
  type    = string
  default = "web-server-01"
}

variable "instance_count" {
  type    = number
  default = 3
}

variable "enable_monitoring" {
  type    = bool
  default = true
}

# ============================================
# COLLECTION TYPES
# ============================================

# LIST — Ordered collection of same type
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# MAP — Key-value pairs of same type
variable "tags" {
  type = map(string)
  default = {
    Environment = "production"
    Team        = "platform"
    CostCenter  = "engineering"
  }
}

# SET — Like list but unordered, no duplicates
variable "allowed_ports" {
  type    = set(number)
  default = [22, 80, 443]
}

# ============================================
# STRUCTURAL TYPES
# ============================================

# OBJECT — Like a map but each attribute can have different type
variable "database_config" {
  type = object({
    name     = string
    port     = number
    username = string
    ssl      = bool
  })
  default = {
    name     = "myapp_db"
    port     = 5432
    username = "admin"
    ssl      = true
  }
}

# TUPLE — Ordered collection where each element can have different type
variable "server_spec" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 20, true]
}
```

---

## String Operations & Interpolation (Deep Dive)

```hcl
locals {
  environment = "production"
  app_name    = "myapp"
  region      = "us-east-1"

  # String interpolation
  bucket_name = "${local.app_name}-${local.environment}-${local.region}"
  # Result: "myapp-production-us-east-1"

  # Heredoc syntax for multiline strings
  user_data_script = <<-EOT
    #!/bin/bash
    echo "Hello from ${local.app_name}"
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
  EOT

  # String functions
  upper_env    = upper(local.environment)        # "PRODUCTION"
  short_region = substr(local.region, 0, 7)      # "us-east"
  joined       = join("-", ["web", "server", "01"]) # "web-server-01"
  split_az     = split(",", "us-east-1a,us-east-1b") # ["us-east-1a","us-east-1b"]
}
```

---

## Meta-Arguments (Very Important!)

Meta-arguments are special arguments that work on **any** resource:

```hcl
# ============================================
# count — Create multiple copies of a resource
# ============================================
resource "aws_instance" "web_servers" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index}"  # web-server-0, web-server-1, web-server-2
  }
}

# Access a specific instance
output "first_server_ip" {
  value = aws_instance.web_servers[0].public_ip
}

# Access all IPs as a list
output "all_server_ips" {
  value = aws_instance.web_servers[*].public_ip
}


# ============================================
# for_each — Better than count for maps/sets
# ============================================
variable "servers" {
  default = {
    web  = "t2.micro"
    api  = "t2.small"
    db   = "t2.medium"
  }
}

resource "aws_instance" "app_servers" {
  for_each      = var.servers
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value  # t2.micro, t2.small, t2.medium

  tags = {
    Name = "server-${each.key}"  # server-web, server-api, server-db
  }
}

# Access specific server
output "web_server_ip" {
  value = aws_instance.app_servers["web"].public_ip
}


# ============================================
# depends_on — Explicit dependency declaration
# ============================================
resource "aws_s3_bucket" "log_bucket" {
  bucket = "my-app-logs"
}

resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  depends_on = [aws_s3_bucket.log_bucket]  # Wait for bucket first
}


# ============================================
# lifecycle — Control resource replacement behavior
# ============================================
resource "aws_instance" "critical_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true   # Create new before deleting old
    prevent_destroy       = true   # Never allow destroy (protects production!)
    ignore_changes        = [ami]  # Don't update if AMI changes externally
  }
}
```

---

## count vs for_each — When to Use Which

```hcl
# USE count WHEN:
# - Creating identical resources
# - The number comes from a simple integer

resource "aws_instance" "identical_workers" {
  count         = 5  # 5 identical worker nodes
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# ⚠️ count PROBLEM: If you delete item from middle, indices shift!
# Deleting index 1 from [0,1,2,3,4] causes resources 2,3,4 to be
# destroyed and recreated — dangerous in production!


# USE for_each WHEN:
# - Resources have unique configurations
# - You need stable resource identities
# - Working with maps or sets

resource "aws_iam_user" "team_members" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.value
}
# ✅ Deleting "bob" only affects bob's user, alice and charlie untouched
```

---

## Interview-Focused Points

> **Q: What's the difference between `terraform plan` and `terraform apply`?**
> Plan shows you what WILL happen (read-only). Apply actually makes the changes. Always review the plan output before applying.

> **Q: What is Terraform state and why is it important?**
> State is a JSON file that maps your HCL code to real-world resource IDs. Without it, Terraform can't know what already exists, leading to duplicate resources or failed destroys.

> **Q: When would you use `count` vs `for_each`?**
> Use `count` for identical resources where order doesn't matter. Use `for_each` for resources with unique properties — it provides stable resource addressing and safer modifications.

> **Q: What does `terraform init` do?**
> It initializes the working directory: downloads providers, configures the backend, and creates the `.terraform` lock file.

---

# 📕 ADVANCED LEVEL

## Deep Dive: How the Terraform Graph Works

Terraform builds a **Directed Acyclic Graph (DAG)** of all resources before doing anything. This is the engine behind everything.

```hcl
# Given this configuration:
resource "aws_vpc" "main" { ... }

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id   # ← This creates an implicit dependency!
}

resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id
}

resource "aws_instance" "web" {
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
}

# Terraform builds this graph:
#
#   aws_vpc.main
#        ↓           ↓
#   aws_subnet   aws_security_group
#        ↓           ↓
#       aws_instance.web
#
# Resources at the same level run IN PARALLEL (default: 10 concurrent)
# Run with more parallelism: terraform apply -parallelism=20
```

---

## Advanced: Local Values (locals) Deep Dive

```hcl
locals {
  # Computed from variables
  is_production = var.environment == "production"
  
  # Conditional logic
  instance_type = local.is_production ? "t3.large" : "t2.micro"
  
  # Complex transformations
  # Convert a list of objects to a map (for_each-ready)
  users_map = {
    for user in var.users :
    user.name => user
  }

  # Filter a list
  admin_users = [
    for user in var.users :
    user.name
    if user.role == "admin"
  ]

  # Nested for expressions
  all_subnet_cidrs = flatten([
    for vpc_name, vpc_config in var.vpcs : [
      for subnet in vpc_config.subnets :
      subnet.cidr
    ]
  ])

  # Merge maps
  common_tags = {
    ManagedBy   = "terraform"
    Environment = var.environment
    UpdatedAt   = timestamp()
  }

  resource_tags = merge(local.common_tags, var.additional_tags)
}
```

---

## Advanced: Dynamic Blocks

Dynamic blocks let you programmatically generate nested blocks within a resource:

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP traffic"
    },
    {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS traffic"
    },
    {
      port        = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
      description = "SSH from internal only"
    }
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  # ✅ Dynamic block — generates multiple ingress blocks from variable
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## Production Best Practices

```hcl
# ============================================
# 1. ALWAYS USE REMOTE STATE in production
# ============================================
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "production/us-east-1/main.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"  # Prevents concurrent applies!
  }
}


# ============================================
# 2. PIN provider versions strictly
# ============================================
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.31.0"  # Exact pin in production
    }
  }
}


# ============================================
# 3. NEVER store secrets in state (use data sources)
# ============================================
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/myapp/db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # ✅ Secret fetched at runtime, not stored in state
}


# ============================================
# 4. Use prevent_destroy for critical resources
# ============================================
resource "aws_db_instance" "production_db" {
  # ... db config ...
  lifecycle {
    prevent_destroy = true  # Prevents accidental terraform destroy
  }
}


# ============================================
# 5. Use validation in variables
# ============================================
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 100
    error_message = "Instance count must be between 1 and 100."
  }
}
```

---

## Real-World Production Architecture Example

```hcl
# ============================================
# PROJECT: Three-Tier Web Application on AWS
# ============================================
# Structure:
# VPC → Public Subnets (ALB) → Private Subnets (App) → DB Subnets (RDS)

# --- variables.tf ---
variable "project_name" {
  type        = string
  description = "Project identifier used in all resource names"
}

variable "environment" {
  type    = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}


# --- locals.tf ---
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  public_subnet_cidrs  = [for i, az in var.availability_zones : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i, az in var.availability_zones : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  db_subnet_cidrs      = [for i, az in var.availability_zones : cidrsubnet(var.vpc_cidr, 8, i + 20)]

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Team        = "platform-engineering"
  }
}


# --- networking.tf ---
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags                 = merge(local.common_tags, { Name = "${local.name_prefix}-vpc" })
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  })
}


# --- compute.tf ---
resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app.id]
  }

  user_data = base64encode(templatefile("${path.module}/scripts/user_data.sh", {
    environment  = var.environment
    project_name = var.project_name
  }))

  lifecycle {
    create_before_destroy = true
  }

  tags = local.common_tags
}

resource "aws_autoscaling_group" "app" {
  name                = "${local.name_prefix}-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"

  min_size         = var.environment == "prod" ? 3 : 1
  max_size         = var.environment == "prod" ? 10 : 3
  desired_capacity = var.environment == "prod" ? 3 : 1

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  dynamic "tag" {
    for_each = local.common_tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}
```

---

## Debugging Strategies

```bash
# 1. Verbose logging — see exactly what Terraform + provider are doing
export TF_LOG=DEBUG
export TF_LOG_PATH=./terraform-debug.log
terraform apply

# Log levels: TRACE, DEBUG, INFO, WARN, ERROR

# 2. Inspect state to understand what Terraform knows
terraform state list                           # List all resources in state
terraform state show aws_instance.web_servers  # Full details of one resource

# 3. Refresh state to sync with real world
terraform refresh   # Updates state to match real infrastructure

# 4. Target specific resources (careful in production!)
terraform plan -target=aws_instance.web_servers
terraform apply -target=aws_security_group.web

# 5. Import existing resources into state
terraform import aws_instance.my_server i-0a1b2c3d4e5f6789

# 6. Validate configuration syntax
terraform validate

# 7. Format code consistently
terraform fmt -recursive

# 8. Console for testing expressions interactively
terraform console
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
> length(["a","b","c"])
3
```

---

# 💻 CODE WRITING GUIDANCE

## Professional File Structure

```
my-terraform-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf          ← Root config for dev
│   │   ├── variables.tf     ← Dev-specific variable declarations
│   │   ├── terraform.tfvars ← Dev variable values (never commit secrets!)
│   │   └── outputs.tf       ← Dev outputs
│   ├── staging/
│   └── production/
│
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md        ← Always document modules!
│   ├── compute/
│   └── database/
│
├── .terraform.lock.hcl      ← Commit this! (provider version lock)
├── .gitignore               ← Ignore *.tfvars, .terraform/, *.tfstate
└── README.md
```

**.gitignore for Terraform:**
```
# .gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars          # May contain secrets!
*.tfvars.json
.terraform.tfstate
crash.log
override.tf
```

---

## Naming Conventions

```hcl
# Resource names: use underscores, be descriptive
resource "aws_instance" "web_server_primary" {}    # ✅
resource "aws_instance" "webServerPrimary" {}       # ❌ (camelCase)
resource "aws_instance" "ws1" {}                    # ❌ (not descriptive)

# Variable names: snake_case, noun-based
variable "instance_count" {}    # ✅
variable "InstanceCount" {}     # ❌
variable "ic" {}                # ❌

# Output names: descriptive, include resource type
output "vpc_id" {}                    # ✅
output "web_server_public_ip" {}      # ✅
output "ip" {}                        # ❌ (too vague)

# Local values: snake_case, computed/derived values
locals {
  name_prefix  = "..."     # ✅
  is_prod      = "..."     # ✅
  namePrefix   = "..."     # ❌
}

# Module names: lowercase, hyphenated in directory, underscored in HCL
module "vpc_networking" {         # ✅ underscore in HCL reference
  source = "./modules/networking" # ✅ hyphen in folder OK
}
```

---

## How Professionals Write It: Step-by-Step

```hcl
# STEP 1: Always start with the terraform block
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  # Add backend from day 1!
  backend "s3" {
    bucket = "my-tfstate-bucket"
    key    = "project/environment/terraform.tfstate"
    region = "us-east-1"
  }
}

# STEP 2: Provider configuration (use variables, not hardcoded values)
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.common_tags  # Apply tags to ALL resources automatically
  }
}

# STEP 3: Data sources before resources (read existing infra)
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# STEP 4: Locals for computed values
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# STEP 5: Resources in dependency order (though Terraform handles this)
resource "aws_vpc" "main" { ... }
resource "aws_subnet" "public" { ... }  # depends on vpc
resource "aws_instance" "web" { ... }   # depends on subnet
```

---

## Anti-Patterns to Avoid

```hcl
# ❌ ANTI-PATTERN 1: Everything in one file
# main.tf with 2000 lines is unmaintainable
# ✅ Split into: networking.tf, compute.tf, database.tf, iam.tf


# ❌ ANTI-PATTERN 2: Hardcoded values everywhere
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # ❌ What region? When valid?
  instance_type = "t2.micro"               # ❌ What if needs change?
}
# ✅ Use data sources and variables


# ❌ ANTI-PATTERN 3: No variable validation
variable "environment" {
  type = string  # ❌ Accepts anything, "prod", "prd", "production"...
}
# ✅ Add validation blocks


# ❌ ANTI-PATTERN 4: Using count where for_each is better
resource "aws_iam_user" "users" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
}
# ✅ Use for_each with toset()


# ❌ ANTI-PATTERN 5: Local state in team environments
# Never use local terraform.tfstate in a team
# ✅ Always use S3 + DynamoDB backend (or Terraform Cloud)


# ❌ ANTI-PATTERN 6: Not using modules for repeated patterns
# Copy-pasting the same VPC config for dev, staging, prod
# ✅ Create a VPC module, call it 3 times with different variables


# ❌ ANTI-PATTERN 7: Ignoring the .terraform.lock.hcl file
# .gitignore includes .terraform.lock.hcl  ← WRONG!
# ✅ Commit the lock file to ensure team uses same provider versions
```

---

# 🧪 HANDS-ON LAB

## Lab Exercise: Multi-Environment Local File Generator

**Objective:** Practice variables, locals, for_each, outputs, and validation without needing any cloud account.

**Task:** Create a Terraform configuration that generates config files for multiple microservices, with environment-specific settings.

**Requirements:**
1. Accept a list of microservice names as a variable
2. Accept environment (dev/staging/prod) as a variable with validation
3. Generate one config file per microservice
4. Each file should contain the service name, environment, and a computed "replica count" (prod=3, staging=2, dev=1)
5. Output the path to all generated files

**Expected Output:**
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

service_configs = {
  "auth-service"  = "./output/auth-service-production.conf"
  "api-gateway"   = "./output/api-gateway-production.conf"
  "user-service"  = "./output/user-service-production.conf"
}
```

**Contents of each file should look like:**
```
Service: auth-service
Environment: production
Replicas: 3
Managed By: Terraform
```

---

**Try it yourself before looking at the solution!**

---

## Lab Solution

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

# ---- variables.tf ----
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}

variable "services" {
  type        = list(string)
  description = "List of microservice names to configure"
  default     = ["auth-service", "api-gateway", "user-service"]
}

# ---- locals.tf ----
locals {
  replica_counts = {
    dev        = 1
    staging    = 2
    production = 3
  }

  replica_count = local.replica_counts[var.environment]

  # Build a map for for_each: service-name => file content
  service_configs = {
    for service in var.services :
    service => {
      filename = "./output/${service}-${var.environment}.conf"
      content  = <<-EOT
        Service: ${service}
        Environment: ${var.environment}
        Replicas: ${local.replica_count}
        Managed By: Terraform
      EOT
    }
  }
}

# ---- main.tf (resources) ----
resource "local_file" "service_config" {
  for_each = local.service_configs
  filename = each.value.filename
  content  = each.value.content
}

# ---- outputs.tf ----
output "service_configs" {
  description = "Map of service names to their config file paths"
  value = {
    for service_name, file in local_file.service_config :
    service_name => file.filename
  }
}

output "replica_count" {
  description = "Number of replicas configured for this environment"
  value       = local.replica_count
}
```

**Run it:**
```bash
mkdir output  # Create output directory

terraform init
terraform plan -var="environment=production"
terraform apply -var="environment=production" -auto-approve

# Try different environments
terraform apply -var="environment=dev" -auto-approve
terraform apply -var="environment=invalid"  # ← See the validation error!
```

---

# 📋 SUMMARY CHEAT SHEET

## Key Concepts

| Concept | What It Is | Why It Matters |
|---|---|---|
| State File | JSON mapping HCL → real resources | Without it, Terraform is blind |
| Provider | Plugin for each platform (AWS, GCP) | Translates HCL to API calls |
| Resource | Infrastructure to create/manage | The core building block |
| Data Source | Read existing infrastructure | Don't recreate what exists |
| Module | Reusable group of resources | DRY principle for infra |
| Backend | Where state is stored | Local=dev, Remote=production |

---

## Quick Syntax Reference

```hcl
# Block types
terraform {}          # Configure Terraform itself
provider "aws" {}     # Configure provider
resource "type" "name" {}   # Create infrastructure
data "type" "name" {}       # Read existing infrastructure
variable "name" {}    # Input variable
output "name" {}      # Output value
locals {}             # Local computed values
module "name" {}      # Call a module

# References
var.variable_name                    # Reference a variable
local.local_name                     # Reference a local
resource_type.resource_name.attribute # Reference resource attribute
module.module_name.output_name       # Reference module output

# Meta-arguments
count = 3                            # Create 3 copies
for_each = var.map_or_set           # Create one per item
depends_on = [resource.name]        # Explicit dependency
lifecycle { prevent_destroy = true } # Protect resource

# Common functions
length(list)                         # Count items
toset(list)                          # Convert list to set
merge(map1, map2)                    # Combine maps
join("-", list)                      # Join list to string
split(",", string)                   # Split string to list
cidrsubnet(cidr, newbits, netnum)   # Calculate subnet CIDR
templatefile(path, vars)             # Render a template file
format("prefix-%s", var.name)       # String formatting
```

---

## Top Interview Questions

**Beginner:**
1. What is Terraform and what problem does it solve?
2. What is the difference between `terraform plan` and `terraform apply`?
3. What is a provider in Terraform?
4. What is HCL?

**Intermediate:**
5. What is Terraform state and where should it be stored in production?
6. What is the difference between `count` and `for_each`?
7. How do you handle secrets in Terraform?
8. What are lifecycle rules and when would you use `prevent_destroy`?
9. What is a data source and when would you use one?
10. How does Terraform handle resource dependencies?

**Advanced:**
11. How does the Terraform DAG (dependency graph) work?
12. What is state locking and why is it critical in team environments?
13. How do you import existing infrastructure into Terraform state?
14. What is the difference between `terraform refresh` and `terraform plan`?
15. How would you structure Terraform code for a large multi-team organization?

---

