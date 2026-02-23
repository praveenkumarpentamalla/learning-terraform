# Terraform Meta-Arguments: Complete Guide — Beginner to Production-Ready

---

# 📘 BEGINNER LEVEL

## What Are Meta-Arguments?

**In simple terms:** Meta-arguments are special instructions you can add to ANY resource block that change *how* Terraform manages that resource — not *what* the resource is.

Regular arguments configure the resource itself (like `instance_type = "t2.micro"`). Meta-arguments configure Terraform's *behavior* toward that resource.

**Why are they used?**
- Create multiple copies of a resource without copy-pasting
- Control the order resources are created/destroyed
- Protect critical resources from accidental deletion
- Tell Terraform to ignore certain changes

**Real-World Analogy:**

> Imagine you're a **construction manager** giving instructions to your crew.
>
> - **Regular arguments** = "Build a house with 3 bedrooms, red roof, wooden floors"
> - **Meta-arguments** = "Build 5 identical copies of it", "Always build the new one before tearing down the old one", "Never demolish this building under any circumstances", "Wait for the road to be built first"
>
> Meta-arguments are **instructions about the building process itself**, not the building's design.

---

## The 5 Meta-Arguments at a Glance

| Meta-Argument | One-Line Purpose |
|---|---|
| `count` | Create N identical copies of a resource |
| `for_each` | Create one resource per item in a map/set |
| `depends_on` | Manually declare "create this AFTER that" |
| `lifecycle` | Control create/update/destroy behavior |
| `provider` | Use a non-default provider configuration |

---

## Basic Syntax

```hcl
resource "aws_instance" "example" {
  # Regular arguments (configure the resource)
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # Meta-arguments (configure Terraform's behavior)
  count      = 3
  depends_on = [aws_vpc.main]

  lifecycle {
    prevent_destroy = true
  }
}
```

---

## Minimal Working Example (No Cloud Needed)

```hcl
# main.tf — Uses local provider, runs on any machine

terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# count — Create 3 files with one resource block
resource "local_file" "notes" {
  count    = 3
  filename = "./note-${count.index}.txt"
  content  = "This is note number ${count.index}"
}

output "all_file_paths" {
  value = local_file.notes[*].filename
}
```

```bash
terraform init
terraform apply -auto-approve

# Output:
# all_file_paths = [
#   "./note-0.txt",
#   "./note-1.txt",
#   "./note-2.txt",
# ]
```

---

## Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Using count AND for_each on the same resource
resource "aws_instance" "bad" {
  count    = 3
  for_each = var.servers  # ERROR: Cannot use both!
}

# ❌ MISTAKE 2: Referencing a counted resource without an index
resource "aws_instance" "web" {
  count = 3
}
output "wrong" {
  value = aws_instance.web.public_ip   # ❌ ERROR: which one?
}
output "correct" {
  value = aws_instance.web[0].public_ip  # ✅ specify index
  # OR
  value = aws_instance.web[*].public_ip  # ✅ get all as list
}

# ❌ MISTAKE 3: depends_on with wrong syntax (missing list brackets)
resource "aws_instance" "app" {
  depends_on = aws_s3_bucket.logs   # ❌ Not a list
  depends_on = [aws_s3_bucket.logs] # ✅ Must be a list
}

# ❌ MISTAKE 4: Putting lifecycle outside the resource block
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"
}
lifecycle {           # ❌ This is outside — syntax error!
  prevent_destroy = true
}

# ✅ lifecycle must be INSIDE the resource block
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  lifecycle {         # ✅ Inside the block
    prevent_destroy = true
  }
}

# ❌ MISTAKE 5: Using count with a map variable (index instability)
variable "user_names" {
  default = ["alice", "bob", "charlie"]
}
resource "aws_iam_user" "users" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
  # ❌ Deleting "alice" shifts all indices — bob becomes [0], charlie [1]
  # Terraform will DESTROY and RECREATE bob and charlie!
}
```

---

# 📗 INTERMEDIATE LEVEL

---

# META-ARGUMENT 1: `count`

## How It Works Internally

When you set `count = N`, Terraform creates **N instances** of that resource. Each instance gets a **numeric index** starting at 0. Internally, Terraform treats them as:

```
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
```

These are stored separately in state. The `count.index` value is available inside the resource block to differentiate each instance.

## Complete Reference

```hcl
# ============================================
# count — Full Practical Reference
# ============================================

# Basic count
resource "aws_instance" "web" {
  count         = 5
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name  = "web-server-${count.index}"   # web-server-0 through web-server-4
    Index = tostring(count.index)
  }
}

# Dynamic count from a variable
variable "server_count" {
  type    = number
  default = 3
}

resource "aws_instance" "dynamic" {
  count         = var.server_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Conditional creation (count = 0 or 1)
variable "create_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count         = var.create_bastion ? 1 : 0  # Create or skip
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Access the optional resource safely
output "bastion_ip" {
  value = var.create_bastion ? aws_instance.bastion[0].public_ip : "No bastion created"
}

# Using count with a list variable
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = var.availability_zones[count.index]   # pairs index to AZ

  tags = {
    Name = "public-subnet-${var.availability_zones[count.index]}"
  }
}
```

## Referencing count Resources

```hcl
# After creating:
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Reference a SPECIFIC instance
output "first_ip"  { value = aws_instance.web[0].public_ip }
output "second_ip" { value = aws_instance.web[1].public_ip }

# Reference ALL instances as a list (splat expression)
output "all_ips"   { value = aws_instance.web[*].public_ip }
# Result: ["1.2.3.4", "5.6.7.8", "9.10.11.12"]

# Use in another resource
resource "aws_eip" "web_ips" {
  count    = 3
  instance = aws_instance.web[count.index].id  # Pair EIP to instance by index
}
```

---

# META-ARGUMENT 2: `for_each`

## How It Works Internally

`for_each` iterates over a **map** or **set** and creates one resource instance per item. Each instance is identified by a **string key** (not a number like count). Internally stored as:

```
aws_instance.web["web"]
aws_instance.web["api"]
aws_instance.web["worker"]
```

The two special values inside the block:
- `each.key` → the map key or set value
- `each.value` → the map value (for maps), same as key (for sets)

## Complete Reference

```hcl
# ============================================
# for_each — Full Practical Reference
# ============================================

# --- With a SET (list of strings, deduplicated) ---
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.value  # "alice", "bob", "charlie"
}

# --- With a MAP (key → value pairs) ---
variable "server_configs" {
  default = {
    web    = "t2.micro"
    api    = "t2.small"
    worker = "t2.medium"
  }
}

resource "aws_instance" "servers" {
  for_each      = var.server_configs
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value   # t2.micro, t2.small, t2.medium

  tags = {
    Name = "server-${each.key}"  # server-web, server-api, server-worker
    Role = each.key
  }
}

# --- With a MAP of OBJECTs (most powerful pattern) ---
variable "services" {
  type = map(object({
    instance_type = string
    port          = number
    replicas      = number
  }))
  default = {
    auth = {
      instance_type = "t2.micro"
      port          = 8080
      replicas      = 2
    }
    payment = {
      instance_type = "t2.large"
      port          = 9090
      replicas      = 5
    }
    notification = {
      instance_type = "t2.small"
      port          = 7070
      replicas      = 1
    }
  }
}

resource "aws_instance" "microservices" {
  for_each      = var.services
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.instance_type

  tags = {
    Name     = "svc-${each.key}"
    Port     = tostring(each.value.port)
    Replicas = tostring(each.value.replicas)
  }
}

# Access a specific instance
output "auth_server_ip" {
  value = aws_instance.microservices["auth"].public_ip
}

# Get all IPs as a map
output "all_service_ips" {
  value = { for k, v in aws_instance.microservices : k => v.public_ip }
}
# Result: { "auth" = "1.2.3.4", "payment" = "5.6.7.8", ... }
```

## for_each with Locals (Real Pattern)

```hcl
# Often you compute for_each values in locals
locals {
  environments = ["dev", "staging", "production"]
  regions      = ["us-east-1", "eu-west-1"]

  # Create all environment-region combinations
  deployments = {
    for pair in setproduct(local.environments, local.regions) :
    "${pair[0]}-${pair[1]}" => {
      environment = pair[0]
      region      = pair[1]
    }
  }
  # Result keys: "dev-us-east-1", "dev-eu-west-1", "staging-us-east-1", ...
}

resource "aws_s3_bucket" "deployments" {
  for_each = local.deployments
  bucket   = "myapp-${each.key}-state"

  tags = {
    Environment = each.value.environment
    Region      = each.value.region
  }
}
```

---

# count vs for_each — The Definitive Comparison

```
┌─────────────────────────────────────────────────────────────┐
│                  count vs for_each                           │
├──────────────────┬──────────────────┬───────────────────────┤
│ Feature          │ count            │ for_each              │
├──────────────────┼──────────────────┼───────────────────────┤
│ Index type       │ Integer (0,1,2)  │ String key            │
│ Input type       │ Number           │ Map or Set            │
│ Stability        │ ⚠️ Fragile       │ ✅ Stable             │
│ Best for         │ Identical copies │ Unique configurations │
│ Deletion safety  │ ❌ Shifts indices│ ✅ Only affects key   │
│ Reference syntax │ resource[0]      │ resource["key"]       │
└──────────────────┴──────────────────┴───────────────────────┘
```

**The Critical Stability Problem with count:**

```hcl
# Initial state: 3 users
variable "users" {
  default = ["alice", "bob", "charlie"]
}
resource "aws_iam_user" "users" {
  count = length(var.users)
  name  = var.users[count.index]
}
# State: users[0]=alice, users[1]=bob, users[2]=charlie

# Now delete "alice":
variable "users" {
  default = ["bob", "charlie"]  # removed alice
}
# State becomes: users[0]=bob, users[1]=charlie
# Terraform sees:
#   users[0]: alice → bob   (UPDATE — rename alice to bob!)
#   users[1]: bob   → charlie (UPDATE — rename bob to charlie!)
#   users[2]: charlie → (delete)
# ❌ DISASTER: 3 users modified/deleted instead of 1!

# ✅ FIX: Use for_each
resource "aws_iam_user" "users" {
  for_each = toset(["bob", "charlie"])  # removed alice
  name     = each.value
}
# Terraform sees:
#   users["alice"]: DELETE ← only this one!
#   users["bob"]:   no change
#   users["charlie"]: no change
# ✅ SAFE: Only alice is deleted
```

---

# META-ARGUMENT 3: `depends_on`

## How It Works Internally

Terraform automatically detects dependencies when you **reference** one resource's attributes in another. `depends_on` is for cases where there's a **hidden dependency** that Terraform can't see through attribute references.

```hcl
# ============================================
# depends_on — Full Practical Reference
# ============================================

# Terraform AUTOMATICALLY handles this (no depends_on needed):
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # ← Reference creates implicit dependency
  cidr_block = "10.0.1.0/24"
}
# Terraform knows: create VPC first, then subnet


# depends_on is needed for HIDDEN dependencies:

# Example 1: IAM propagation delay
resource "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_policy" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_lambda_function" "my_function" {
  function_name = "my-function"
  role          = aws_iam_role.lambda_role.arn
  # Even though role ARN is referenced, the POLICY ATTACHMENT is a hidden dep
  # Lambda might deploy before policy is attached!
  depends_on = [aws_iam_role_policy_attachment.lambda_policy]  # ✅ Explicit
}


# Example 2: S3 bucket policy must exist before objects are written
resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs-bucket"
}

resource "aws_s3_bucket_policy" "logs_policy" {
  bucket = aws_s3_bucket.logs.id
  policy = jsonencode({ ... })
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  # App writes to S3 at startup — bucket policy must be ready first!
  depends_on = [aws_s3_bucket_policy.logs_policy]
}


# Example 3: Module-level depends_on
module "database" {
  source = "./modules/rds"
}

module "application" {
  source     = "./modules/ec2"
  depends_on = [module.database]  # Entire module dependency
}
```

## When NOT to Use depends_on

```hcl
# ❌ DON'T use depends_on when reference already creates dependency
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id   # This IS the dependency signal
  cidr_block = "10.0.1.0/24"
  depends_on = [aws_vpc.main]    # ❌ Redundant! Reference already handles it
}

# ❌ DON'T overuse depends_on — it forces SEQUENTIAL execution
# Terraform's parallel execution is a performance feature
# Unnecessary depends_on eliminates that parallelism
```

---

# META-ARGUMENT 4: `lifecycle`

## How It Works Internally

`lifecycle` rules intercept Terraform's normal create/update/destroy workflow and modify the behavior. It has 5 settings, each solving a different problem.

## Complete Reference — All 5 lifecycle Settings

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    # Setting 1: create_before_destroy
    # Default behavior: destroy old → create new (causes downtime!)
    # With this: create new → destroy old (zero downtime replacement)
    create_before_destroy = true

    # Setting 2: prevent_destroy
    # Raises an error if anyone tries to terraform destroy this resource
    # Critical for: production databases, S3 buckets with data, etc.
    prevent_destroy = true

    # Setting 3: ignore_changes
    # Tell Terraform to ignore external changes to specific attributes
    # Useful when: auto-scaling changes instance count, humans change tags, etc.
    ignore_changes = [
      tags,           # Ignore ALL tag changes
      ami,            # Ignore AMI updates (managed outside Terraform)
      instance_type,  # Ignore manual resizing
    ]
    # OR ignore everything (use sparingly!):
    # ignore_changes = all

    # Setting 4: replace_triggered_by
    # Force replacement of this resource when another resource changes
    replace_triggered_by = [
      aws_security_group.web.id  # Replace instance if security group changes
    ]

    # Setting 5: precondition / postcondition (Terraform 1.2+)
    precondition {
      condition     = var.instance_type != "t1.micro"
      error_message = "t1.micro is deprecated and not allowed."
    }
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
}
```

## Deep Dive: create_before_destroy

```hcl
# ============================================
# The Problem: Downtime During Replacement
# ============================================

# Scenario: You update the AMI on a web server
# WITHOUT create_before_destroy:
#   1. Terraform destroys old instance  ← DOWNTIME STARTS
#   2. Terraform creates new instance   ← DOWNTIME ENDS
#   Total downtime: instance creation time (2-5 minutes)

# WITH create_before_destroy:
#   1. Terraform creates new instance  ← No downtime
#   2. Terraform destroys old instance ← New one is already serving
#   Total downtime: 0

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  lifecycle {
    create_before_destroy = true
  }
}

# ⚠️ IMPORTANT: create_before_destroy cascades!
# If resource A has it, and resource B depends on A,
# resource B is also implicitly create_before_destroy

# ⚠️ WATCH OUT: Unique name constraints
# If your resource has a unique name, create_before_destroy
# will FAIL because you can't have two resources with same name!
resource "aws_iam_role" "bad_example" {
  name = "my-fixed-role-name"   # ❌ Two roles can't have same name!
  lifecycle {
    create_before_destroy = true  # ❌ Will fail — name conflict
  }
}

resource "aws_iam_role" "good_example" {
  name_prefix = "my-role-"   # ✅ Generates unique suffix: my-role-abc123
  lifecycle {
    create_before_destroy = true  # ✅ Works because names are unique
  }
}
```

## Deep Dive: prevent_destroy

```hcl
# ============================================
# prevent_destroy — Production Safety Net
# ============================================

resource "aws_db_instance" "production" {
  identifier        = "prod-postgres"
  engine            = "postgres"
  instance_class    = "db.t3.large"
  allocated_storage = 100

  lifecycle {
    prevent_destroy = true
  }
}

# What happens when you try to destroy?
# $ terraform destroy
# Error: Instance cannot be destroyed
#
# on database.tf line 12, in resource "aws_db_instance" "production":
# Resource aws_db_instance.production has lifecycle.prevent_destroy set,
# but the plan calls for this resource to be destroyed.

# ⚠️ prevent_destroy does NOT prevent:
# - Manual deletion in AWS console (Terraform just loses track of it)
# - terraform state rm (removes from state without deleting)
# - Removing the lifecycle block and then destroying

# ✅ Best practice: Combine with deletion_protection on the resource itself
resource "aws_db_instance" "production" {
  identifier         = "prod-postgres"
  deletion_protection = true   # ← AWS-level protection

  lifecycle {
    prevent_destroy = true     # ← Terraform-level protection
  }
}
```

## Deep Dive: ignore_changes

```hcl
# ============================================
# ignore_changes — Real-World Scenarios
# ============================================

# SCENARIO 1: Auto Scaling changes desired_count
# If you set desired_capacity=3 but Auto Scaling changes it to 5,
# Terraform would try to "fix" it back to 3 on next apply.
# You don't want that!
resource "aws_autoscaling_group" "app" {
  min_size         = 2
  max_size         = 10
  desired_capacity = 3   # Initial value

  lifecycle {
    ignore_changes = [desired_capacity]  # ✅ Let Auto Scaling manage this
  }
}

# SCENARIO 2: External team manages tags
resource "aws_instance" "shared" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name        = "shared-server"
    Environment = "production"
    # Finance team adds CostCenter tags manually in console
    # Security team adds Compliance tags
    # If we don't ignore, Terraform removes their tags on next apply!
  }

  lifecycle {
    ignore_changes = [tags]  # ✅ Don't touch tags that others manage
  }
}

# SCENARIO 3: Ignore AMI changes (custom AMIs managed by separate pipeline)
resource "aws_instance" "app" {
  ami           = data.aws_ami.latest.id  # Initial AMI
  instance_type = "t2.micro"

  lifecycle {
    ignore_changes = [ami]  # AMI updates managed by separate AMI pipeline
  }
}

# ⚠️ WARNING: ignore_changes = all is a code smell
resource "aws_instance" "bad" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  lifecycle {
    ignore_changes = all  # ❌ Terraform becomes blind to all changes
    # This defeats the purpose of Terraform!
    # Only acceptable for very specific bootstrapping scenarios
  }
}
```

## Preconditions & Postconditions (Terraform 1.2+)

```hcl
# ============================================
# precondition / postcondition — Contract Checks
# ============================================

variable "environment" {
  type = string
}

variable "instance_type" {
  type = string
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  lifecycle {
    # Checked BEFORE creating/modifying the resource
    precondition {
      condition = (
        var.environment == "production" &&
        contains(["t3.large", "t3.xlarge", "m5.large"], var.instance_type)
      ) || var.environment != "production"
      error_message = "Production environment requires t3.large, t3.xlarge, or m5.large."
    }

    # Checked AFTER the resource is created
    postcondition {
      condition     = self.public_ip != null
      error_message = "Instance was created but has no public IP. Check subnet config."
    }
  }
}

# Preconditions in data sources — validate before using external data
data "aws_ami" "app_ami" {
  most_recent = true
  owners      = ["self"]

  lifecycle {
    postcondition {
      condition     = self.tags["Validated"] == "true"
      error_message = "The AMI must be tagged with Validated=true before use in prod."
    }
  }
}
```

---

# META-ARGUMENT 5: `provider`

## How It Works Internally

By default, each resource uses the single provider configuration for its type. The `provider` meta-argument lets you specify **which provider alias** to use — critical for multi-region and multi-account deployments.

```hcl
# ============================================
# provider meta-argument — Multi-Region/Account
# ============================================

# Define multiple provider configurations with aliases
provider "aws" {
  region = "us-east-1"
  # This is the DEFAULT provider (no alias)
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast"
  region = "ap-southeast-1"
}

# Multi-account setup
provider "aws" {
  alias = "production_account"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformDeployRole"
  }
  region = "us-east-1"
}

provider "aws" {
  alias = "staging_account"
  assume_role {
    role_arn = "arn:aws:iam::STAGING_ACCOUNT_ID:role/TerraformDeployRole"
  }
  region = "us-east-1"
}

# Resources using specific providers
resource "aws_s3_bucket" "us_bucket" {
  bucket = "my-app-us-east-data"
  # Uses default provider (us-east-1) — no meta-argument needed
}

resource "aws_s3_bucket" "eu_bucket" {
  bucket   = "my-app-eu-west-data"
  provider = aws.eu_west  # ← provider meta-argument!
}

resource "aws_s3_bucket" "ap_bucket" {
  bucket   = "my-app-ap-southeast-data"
  provider = aws.ap_southeast
}

resource "aws_instance" "prod_server" {
  provider      = aws.production_account
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "aws_instance" "staging_server" {
  provider      = aws.staging_account
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Pass provider to modules
module "us_networking" {
  source = "./modules/networking"
  providers = {
    aws = aws          # Default provider
  }
}

module "eu_networking" {
  source = "./modules/networking"
  providers = {
    aws = aws.eu_west  # EU provider
  }
}
```

---

# 📕 ADVANCED LEVEL

## Deep Dive: How Meta-Arguments Affect the DAG

```
# Configuration:
resource "aws_s3_bucket" "logs" { }
resource "aws_s3_bucket_policy" "policy" { bucket = aws_s3_bucket.logs.id }
resource "aws_instance" "app" { depends_on = [aws_s3_bucket_policy.policy] }
resource "aws_instance" "web" { count = 3 }

# Terraform DAG:
#
#  aws_s3_bucket.logs
#          ↓
#  aws_s3_bucket_policy.policy
#          ↓
#  aws_instance.app
#
#  aws_instance.web[0] ──┐
#  aws_instance.web[1] ──┤  (parallel, no dependencies between them)
#  aws_instance.web[2] ──┘
#
# app and web[0,1,2] run CONCURRENTLY (different branches of DAG)
# Terraform default parallelism: 10 concurrent operations
```

## Advanced: for_each with Complex Transformations

```hcl
# ============================================
# PATTERN: Normalizing data for for_each
# ============================================

# Input: flat list of objects
variable "security_group_rules" {
  default = [
    { name = "http",  port = 80,  cidr = "0.0.0.0/0"   },
    { name = "https", port = 443, cidr = "0.0.0.0/0"   },
    { name = "ssh",   port = 22,  cidr = "10.0.0.0/8"  },
  ]
}

# Transform list to map for for_each (using name as key)
locals {
  sg_rules_map = { for rule in var.security_group_rules : rule.name => rule }
  # Result:
  # {
  #   "http"  = { name = "http",  port = 80,  cidr = "0.0.0.0/0" }
  #   "https" = { name = "https", port = 443, cidr = "0.0.0.0/0" }
  #   "ssh"   = { name = "ssh",   port = 22,  cidr = "10.0.0.0/8" }
  # }
}

resource "aws_security_group_rule" "ingress" {
  for_each          = local.sg_rules_map
  type              = "ingress"
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = "tcp"
  cidr_blocks       = [each.value.cidr]
  security_group_id = aws_security_group.app.id
  description       = "Allow ${each.key} traffic"
}


# ============================================
# PATTERN: Cartesian product for multi-region multi-env
# ============================================
locals {
  regions      = ["us-east-1", "eu-west-1"]
  environments = ["staging", "production"]

  deployments = {
    for combo in setproduct(local.regions, local.environments) :
    "${combo[1]}-${combo[0]}" => {
      region      = combo[0]
      environment = combo[1]
    }
  }
  # Keys: "staging-us-east-1", "production-us-east-1",
  #       "staging-eu-west-1", "production-eu-west-1"
}


# ============================================
# PATTERN: Flatten nested structures for for_each
# ============================================
variable "teams" {
  default = {
    backend  = ["alice", "bob"]
    frontend = ["charlie", "diana"]
    devops   = ["eve"]
  }
}

locals {
  # Create a flat map of user → team for IAM user creation
  team_members = merge([
    for team, members in var.teams : {
      for member in members :
      member => team
    }
  ]...)
  # Result: { "alice"="backend", "bob"="backend", "charlie"="frontend", ... }
}

resource "aws_iam_user" "members" {
  for_each = local.team_members
  name     = each.key

  tags = {
    Team = each.value
  }
}
```

## Advanced: lifecycle Edge Cases

```hcl
# ============================================
# EDGE CASE 1: replace_triggered_by
# ============================================
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    # Force instance replacement when security group changes
    # Even if no instance attributes changed
    replace_triggered_by = [
      aws_security_group.app  # Replace when SG is replaced
    ]
  }
}


# ============================================
# EDGE CASE 2: prevent_destroy doesn't survive lifecycle removal
# ============================================
# If someone removes the lifecycle block and applies,
# Terraform CAN destroy the resource!
# Solution: Use AWS deletion protection + Terraform prevent_destroy

resource "aws_rds_cluster" "production" {
  cluster_identifier  = "prod-aurora"
  deletion_protection = true   # AWS-level: prevents deletion via API

  lifecycle {
    prevent_destroy = true     # Terraform-level: prevents plan with destroy
  }
}


# ============================================
# EDGE CASE 3: ignore_changes with computed values
# ============================================
resource "aws_instance" "app" {
  ami           = data.aws_ami.latest.id
  instance_type = "t2.micro"

  # You CANNOT ignore computed/read-only attributes this way:
  lifecycle {
    # ignore_changes = [id]        # ❌ id is always computed
    # ignore_changes = [public_ip] # ❌ public_ip is always computed

    # Only ignore CONFIGURABLE attributes:
    ignore_changes = [ami, tags, user_data]  # ✅
  }
}


# ============================================
# EDGE CASE 4: count = 0 cleanup behavior
# ============================================
# If you change count from 3 to 0, ALL 3 resources are destroyed
# If you change count from 3 to 1, resources [1] and [2] are destroyed
# with for_each, removing keys is always targeted and safe

variable "enable_monitoring" {
  type    = bool
  default = true
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0  # Common on/off pattern

  alarm_name  = "high-cpu-usage"
  # ... alarm config ...
}
```

## Performance Considerations

```hcl
# ============================================
# PERFORMANCE: Parallelism Tuning
# ============================================

# Default: Terraform runs 10 operations concurrently
# For large infrastructures, tune this:

# Increase for faster applies (more API calls in parallel)
# terraform apply -parallelism=20

# Decrease for rate-limited APIs or flaky providers
# terraform apply -parallelism=5

# ⚠️ Unnecessary depends_on destroys parallelism
# This forces sequential execution:
resource "aws_instance" "worker_1" { }
resource "aws_instance" "worker_2" {
  depends_on = [aws_instance.worker_1]  # ❌ Forces sequential if not needed
}

# ✅ Without depends_on, both run in parallel


# ============================================
# PERFORMANCE: for_each at scale
# ============================================
# Creating 1000 resources with for_each is fine
# But planning becomes slow because Terraform must:
# 1. Read all 1000 items from state
# 2. Compare with 1000 real resources via API
# 3. Generate diff

# For very large for_each sets, consider:
# - Splitting into multiple state files (workspaces or separate backends)
# - Using -target to apply subsets
# - Using Terraform Cloud with remote runs for large state files
```

---

# 🏗️ REAL-WORLD PRODUCTION ARCHITECTURE

## Complete Example: Multi-Region, Multi-Environment Deployment

```hcl
# ============================================
# PRODUCTION PATTERN: Multi-Region Active-Active
# ============================================

# providers.tf
provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
}

# variables.tf
variable "primary_region"   { default = "us-east-1" }
variable "secondary_region" { default = "eu-west-1" }
variable "environment"      { type = string }
variable "services" {
  type = map(object({
    instance_type = string
    min_instances = number
    max_instances = number
    ports         = list(number)
  }))
}

# locals.tf
locals {
  name_prefix = "${var.environment}"

  # Cross-region service deployment map
  regional_services = merge(
    { for name, config in var.services :
      "${name}-primary" => merge(config, { region = var.primary_region, provider = "primary" })
    },
    { for name, config in var.services :
      "${name}-secondary" => merge(config, { region = var.secondary_region, provider = "secondary" })
    }
  )

  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Repository  = "github.com/myorg/infrastructure"
  }
}

# networking.tf — Primary Region
resource "aws_vpc" "primary" {
  provider   = aws.primary
  cidr_block = "10.0.0.0/16"

  tags = merge(local.common_tags, {
    Name   = "${local.name_prefix}-vpc-primary"
    Region = var.primary_region
  })
}

resource "aws_vpc" "secondary" {
  provider   = aws.secondary
  cidr_block = "10.1.0.0/16"

  tags = merge(local.common_tags, {
    Name   = "${local.name_prefix}-vpc-secondary"
    Region = var.secondary_region
  })
}

# Security Groups — dynamic ingress rules
variable "allowed_ingress_rules" {
  type = list(object({
    name        = string
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { name = "http",  port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { name = "https", port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
  ]
}

locals {
  ingress_rules_map = { for rule in var.allowed_ingress_rules : rule.name => rule }
}

resource "aws_security_group" "app_primary" {
  provider = aws.primary
  name     = "${local.name_prefix}-app-sg"
  vpc_id   = aws_vpc.primary.id

  dynamic "ingress" {
    for_each = local.ingress_rules_map
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = "Allow ${ingress.key}"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  lifecycle {
    create_before_destroy = true  # Zero downtime SG replacement
  }

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-app-sg-primary" })
}

# compute.tf — Services with for_each
resource "aws_autoscaling_group" "services_primary" {
  for_each = var.services

  provider            = aws.primary
  name                = "${local.name_prefix}-${each.key}-primary"
  vpc_zone_identifier = aws_subnet.private_primary[*].id

  min_size         = each.value.min_instances
  max_size         = each.value.max_instances
  desired_capacity = each.value.min_instances

  launch_template {
    id      = aws_launch_template.services_primary[each.key].id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes        = [desired_capacity]   # Let auto-scaling manage this
    create_before_destroy = true
  }

  dynamic "tag" {
    for_each = merge(local.common_tags, {
      Service = each.key
      Region  = var.primary_region
    })
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}

# Database — Protected with multiple lifecycle rules
resource "aws_db_cluster" "primary" {
  cluster_identifier  = "${local.name_prefix}-aurora-primary"
  engine              = "aurora-postgresql"
  engine_version      = "14.6"
  deletion_protection = true

  lifecycle {
    prevent_destroy       = true
    ignore_changes        = [master_password]  # Managed by secrets rotation
    create_before_destroy = false              # Can't have two primaries!
  }

  depends_on = [
    aws_db_subnet_group.primary,
    aws_security_group.database_primary,
  ]

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-aurora-primary"
    Tier = "database"
  })
}
```

---

## Debugging Meta-Argument Issues

```bash
# ============================================
# DEBUGGING TOOLKIT
# ============================================

# 1. See all resource addresses (including count/for_each indices)
terraform state list
# Output:
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]
# aws_instance.servers["api"]
# aws_instance.servers["web"]

# 2. See what Terraform plans to do with specific resources
terraform plan -target='aws_instance.web[0]'
terraform plan -target='aws_instance.servers["api"]'

# 3. Check full state of a specific resource
terraform state show 'aws_instance.servers["api"]'

# 4. Debug dependency issues — visualize the graph
terraform graph | dot -Tsvg > graph.svg
# Requires graphviz: brew install graphviz

# 5. If for_each key changes (rename a resource without destroy/create):
terraform state mv \
  'aws_instance.servers["old-name"]' \
  'aws_instance.servers["new-name"]'

# 6. Remove prevent_destroy temporarily (edit tf file, then re-add after)
# Or use: terraform state rm aws_db_instance.production
# (removes from state without deleting — then you can import it back)

# 7. Debug lifecycle ignore_changes issues
# Temporarily comment out ignore_changes and run terraform plan
# to see what changes are being suppressed

# 8. Force replacement (when you WANT to replace despite no changes)
terraform apply -replace='aws_instance.web[0]'
# This respects create_before_destroy lifecycle rules!

# 9. Validate that your for_each keys are what you expect
terraform console
> var.services
> { for k, v in var.services : k => v.instance_type }
```

---

# 🧪 HANDS-ON LAB

## Lab: Multi-Service Configuration Manager

**Objective:** Build a configuration system for multiple microservices across multiple environments using all 5 meta-arguments.

**Requirements:**
1. Use **`for_each`** to create a config file per service
2. Use **`count`** to generate N replica config files per service (based on environment)
3. Use **`depends_on`** to ensure a "registry" file is created before service files
4. Use **`lifecycle`** with `prevent_destroy` on the registry and `ignore_changes` on timestamps
5. Use a secondary **`provider`** alias for a "backup" directory
6. Validate that environment is one of: dev, staging, production

**Expected output:**
```
Apply complete! Resources: 9 added, 0 changed, 0 destroyed.

Outputs:

registry_path    = "./output/registry.json"
service_configs  = {
  "auth"     = "./output/auth.conf"
  "payments" = "./output/payments.conf"
  "users"    = "./output/users.conf"
}
replica_count = 3
```

**Try it yourself first!**

---

## Lab Solution

```hcl
# ============================================
# FILE STRUCTURE:
# .
# ├── main.tf
# ├── variables.tf
# ├── locals.tf
# ├── outputs.tf
# └── output/        ← created by Terraform
# └── output/backup/ ← created by Terraform
# ============================================

# main.tf
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.4"
    }
  }
}

# Primary provider — writes to ./output/
provider "local" {}

# Secondary provider alias — writes to ./output/backup/
provider "local" {
  alias = "backup"
}

# variables.tf
variable "environment" {
  type        = string
  description = "Target environment"
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Must be dev, staging, or production."
  }
}

variable "services" {
  type    = list(string)
  default = ["auth", "payments", "users"]
}

# locals.tf
locals {
  replica_map = {
    dev        = 1
    staging    = 2
    production = 3
  }
  replica_count = local.replica_map[var.environment]
  services_set  = toset(var.services)

  registry_content = jsonencode({
    environment = var.environment
    services    = var.services
    replicas    = local.replica_count
    managed_by  = "terraform"
  })
}

# RESOURCE 1: Registry file (must exist first)
resource "local_file" "registry" {
  filename = "./output/registry.json"
  content  = local.registry_content

  lifecycle {
    prevent_destroy = true          # Registry must never be accidentally deleted
    ignore_changes  = [content]     # Don't re-write if content drifts externally
  }
}

# RESOURCE 2: Service config files (one per service, depends on registry)
resource "local_file" "service_config" {
  for_each = local.services_set

  filename = "./output/${each.value}.conf"
  content  = <<-EOT
    service=${each.value}
    environment=${var.environment}
    replicas=${local.replica_count}
    registry=./output/registry.json
  EOT

  depends_on = [local_file.registry]  # Registry must exist first
}

# RESOURCE 3: Replica placeholder files (count pattern)
resource "local_file" "replica_markers" {
  count    = local.replica_count
  filename = "./output/replica-${count.index}.marker"
  content  = "replica-${count.index} ready for ${var.environment}"

  depends_on = [local_file.service_config]
}

# RESOURCE 4: Backup copies (using provider alias)
resource "local_file" "backup_registry" {
  provider = local.backup
  filename = "./output/backup/registry.json"
  content  = local.registry_content
}

resource "local_file" "backup_service_configs" {
  provider  = local.backup
  for_each  = local.services_set
  filename  = "./output/backup/${each.value}.conf"
  content   = local_file.service_config[each.value].content
  depends_on = [local_file.service_config]
}

# outputs.tf
output "registry_path" {
  value = local_file.registry.filename
}

output "service_configs" {
  value = { for k, v in local_file.service_config : k => v.filename }
}

output "replica_count" {
  value = local.replica_count
}

output "backup_paths" {
  value = { for k, v in local_file.backup_service_configs : k => v.filename }
}
```

```bash
# Run it:
mkdir -p output/backup

terraform init
terraform apply -var="environment=production" -auto-approve

# Test prevent_destroy:
terraform destroy -var="environment=production"
# Should error on registry file!

# Test validation:
terraform apply -var="environment=prod"
# Should error: Must be dev, staging, or production.

# Test count stability vs for_each stability:
terraform apply -var="environment=staging" -auto-approve
# Watch how replica_count changes from 3→2, removing files [2]
# But service files are stable (for_each keys unchanged)
```

---

# 📋 SUMMARY CHEAT SHEET

## Quick Syntax Reference

```hcl
# count
resource "type" "name" {
  count = 3                          # or: var.count, condition ? 1 : 0
  name  = "item-${count.index}"      # index: 0, 1, 2
}
resource.name[0]                     # reference specific
resource.name[*].attribute           # splat: all as list

# for_each
resource "type" "name" {
  for_each = var.map_or_set          # toset([...]) for lists
  name     = each.key                # map key or set value
  value    = each.value              # map value (same as key for sets)
}
resource.name["key"].attribute       # reference specific
{ for k, v in resource.name : k => v.attribute }  # all as map

# depends_on
resource "type" "name" {
  depends_on = [other_resource.name, module.name]
}

# lifecycle
resource "type" "name" {
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    ignore_changes        = [attribute1, attribute2]
    replace_triggered_by  = [other_resource.name]
    precondition {
      condition     = <bool expression>
      error_message = "Human-readable error"
    }
    postcondition {
      condition     = self.attribute != null
      error_message = "Human-readable error"
    }
  }
}

# provider
resource "type" "name" {
  provider = aws.alias_name
}
module "name" {
  providers = { aws = aws.alias_name }
}
```

---

## Key Decision Framework

```
Need multiple copies of a resource?
├── Are they IDENTICAL (same config, just more of them)?
│   └── Use count
├── Do they have UNIQUE configs or need stable identity?
│   └── Use for_each with a map
└── Is it ON/OFF optional?
    └── Use count = condition ? 1 : 0

Need to control creation ORDER?
├── Is there an attribute reference between resources?
│   └── Terraform handles it automatically — no action needed
└── Is it a HIDDEN dependency (IAM propagation, startup scripts)?
    └── Use depends_on

Need to control DESTROY/REPLACE behavior?
├── Don't want downtime during replacement?
│   └── lifecycle { create_before_destroy = true }
├── Protecting critical production resource?
│   └── lifecycle { prevent_destroy = true }
├── External system modifying certain attributes?
│   └── lifecycle { ignore_changes = [attr] }
└── Resource must be replaced when dependency changes?
    └── lifecycle { replace_triggered_by = [resource] }

Need to deploy to multiple regions/accounts?
└── Use provider aliases + provider meta-argument
```

---

## Top Interview Questions on Meta-Arguments

**Q1: What's the difference between count and for_each? When do you use each?**
> `count` uses integer indices — safe for identical resources. `for_each` uses string keys — safe for unique resources and deletions. Use `for_each` whenever resources have meaningful identities, because deleting a `count` item shifts all indices causing unnecessary destroy/recreate.

**Q2: If resource B uses the output of resource A, do you still need depends_on?**
> No. Terraform automatically infers the dependency from the attribute reference. `depends_on` is only for **hidden dependencies** — where B depends on A's side effects but doesn't reference A's attributes (e.g., IAM policy attached before Lambda uses the role).

**Q3: What does `create_before_destroy = true` actually do and what's the catch?**
> It reverses the default order: create replacement first, then destroy original — achieving zero-downtime replacement. The catch is **unique name constraints**: if the resource requires a unique name (like IAM roles), you must use `name_prefix` instead of `name` so both can coexist briefly during replacement.

**Q4: Can prevent_destroy save you if someone removes the lifecycle block?**
> No. `prevent_destroy = true` only protects against destroying while the block exists. If someone removes the lifecycle block and runs apply, the protection is gone. Always combine with AWS-level deletion protection (`deletion_protection = true` on RDS, `prevent_destroy` on buckets, etc.).

**Q5: What happens when you change a for_each key name?**
> Terraform sees it as delete old key + create new key. To rename without destroying, use `terraform state mv 'resource["old"]' 'resource["new"]'` before applying.

**Q6: How do you deploy the same module to multiple AWS regions?**
> Define multiple provider aliases with different regions, then call the module multiple times passing `providers = { aws = aws.region_alias }` to each invocation.

**Q7: What is replace_triggered_by and when would you use it?**
> It forces a resource to be replaced (destroyed and recreated) when a referenced resource changes — even if none of the resource's own arguments changed. Use case: EC2 instances that must be recycled when their security group is replaced, or blue/green deployments triggered by a new launch template.

---

