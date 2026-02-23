# 🔌 TOPIC 2: Providers & Resources — Deep Dive

---

## 🟢 BEGINNER LEVEL

### What is a Provider? (Simple Terms)

Imagine Terraform is a **universal TV remote**. By itself, it does nothing. But when you plug in a **specific module for Samsung TV**, it knows how to talk to Samsung. Plug in a **LG module**, it talks to LG.

**Providers are those modules** — they teach Terraform how to talk to specific platforms.

```
Terraform Core  +  AWS Provider   =  Can create AWS resources
Terraform Core  +  Azure Provider =  Can create Azure resources
Terraform Core  +  GitHub Provider =  Can manage GitHub repos
Terraform Core  +  Kubernetes Provider =  Can deploy K8s resources
```

Without a provider, Terraform is just an engine with no fuel.

---

### What is a Resource? (Simple Terms)

A **resource** is any piece of infrastructure you want Terraform to create and manage.

```
Provider: AWS
└── Resources:
    ├── aws_instance          (EC2 virtual machine)
    ├── aws_s3_bucket         (Storage bucket)
    ├── aws_vpc               (Virtual network)
    ├── aws_rds_instance      (Database)
    ├── aws_lambda_function   (Serverless function)
    └── aws_iam_role          (Security role)
```

Think of a Provider as a **department store** and Resources as the **individual products** you buy from it.

---

### Real-World Analogy

```
Real World                    Terraform Equivalent
─────────────────────────────────────────────────
App Store                  →  Terraform Registry (registry.terraform.io)
App (e.g., Instagram)      →  Provider (e.g., aws, azure)
Feature (e.g., Stories)    →  Resource (e.g., aws_s3_bucket)
Feature Settings           →  Resource Arguments
App Permissions            →  Provider Credentials (API keys)
App Version                →  Provider Version
```

---

### Basic Provider Syntax

```hcl
# Minimal provider block
provider "aws" {
  region = "us-east-1"
}

# Basic resource block
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-12345"
}
```

**Resource block anatomy:**
```
resource  "aws_s3_bucket"  "my_bucket"  {
   ↑            ↑               ↑
Block Type  Resource Type   Local Name
            (from provider)  (your name)
```

```
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-12345"
     ↑                ↑
  Argument Key     Argument Value
```

---

### How to Find Resources

```
1. Go to: registry.terraform.io
2. Search for your provider: "aws"
3. Click Documentation
4. Browse resource types: aws_instance, aws_s3_bucket, etc.
5. Each resource shows: required arguments, optional arguments, attributes
```

---

### Minimal Working Example

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# main.tf
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id   # References the VPC above
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name = "public-subnet"
  }
}
```

Notice: `aws_vpc.main.id` — this is how resources **reference each other**. Terraform automatically understands this creates a dependency.

---

### Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Wrong resource type name
resource "aws_s3" "bucket" { }          # Wrong — it's aws_s3_bucket
resource "aws_ec2_instance" "web" { }   # Wrong — it's aws_instance

# ❌ MISTAKE 2: Same local name for same resource type
resource "aws_instance" "web" { ... }
resource "aws_instance" "web" { ... }   # Duplicate! Will error

# ❌ MISTAKE 3: Circular references
resource "aws_security_group" "a" {
  # references b...
}
resource "aws_security_group" "b" {
  # references a... ← Terraform can't resolve this
}

# ❌ MISTAKE 4: Not declaring provider before using it
resource "aws_instance" "web" {         # Error if no provider "aws" block
  ...
}

# ❌ MISTAKE 5: Confusing resource type vs local name
resource "aws_instance" "aws_instance" { }  # Works but VERY confusing
# Use: resource "aws_instance" "web_server" { }

# ❌ MISTAKE 6: Missing required arguments
resource "aws_instance" "web" {
  # ami and instance_type are REQUIRED — this will fail plan
  tags = { Name = "web" }
}
```

---

## 🟡 INTERMEDIATE LEVEL

### Provider Internal Working

When you run `terraform init`, here's exactly what happens with providers:

```
terraform init
      │
      ▼
1. Reads terraform {} block for required_providers
      │
      ▼
2. Downloads provider binary from registry.terraform.io
   (stored in .terraform/providers/registry.terraform.io/hashicorp/aws/5.x.x/)
      │
      ▼
3. Verifies checksum against .terraform.lock.hcl
      │
      ▼
4. Provider binary is now available as a gRPC plugin
      │
      ▼
5. When plan/apply runs, Terraform Core communicates
   with provider via gRPC protocol
      │
      ▼
6. Provider translates HCL config → Cloud API calls
   (e.g., CreateBucket, RunInstances, CreateVpc)
```

```
┌─────────────────────────────────────────────────────────┐
│  Your .tf files                                         │
│  resource "aws_s3_bucket" "bucket" { bucket = "test" }  │
└──────────────────────┬──────────────────────────────────┘
                       │ HCL Config
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Terraform Core                                         │
│  - Parses HCL                                           │
│  - Builds dependency graph                              │
│  - Manages state                                        │
└──────────────────────┬──────────────────────────────────┘
                       │ gRPC calls
                       ▼
┌─────────────────────────────────────────────────────────┐
│  AWS Provider Plugin (terraform-provider-aws)           │
│  - Validates config                                     │
│  - Translates to AWS SDK calls                          │
│  - Returns resource attributes                          │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS API calls
                       ▼
┌─────────────────────────────────────────────────────────┐
│  AWS APIs                                               │
│  s3.amazonaws.com, ec2.amazonaws.com, etc.              │
└─────────────────────────────────────────────────────────┘
```

---

### Provider Configuration — All Options

```hcl
provider "aws" {
  # ── AUTHENTICATION (pick ONE method) ──────────────────────────────

  # Method 1: Static credentials (NEVER use in production)
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

  # Method 2: Profile from ~/.aws/credentials (good for local dev)
  profile = "myprofile"

  # Method 3: IAM Role (best for CI/CD and EC2)
  # No explicit config — uses instance profile or env vars automatically

  # Method 4: Assume role (cross-account access)
  assume_role {
    role_arn     = "arn:aws:iam::123456789012:role/TerraformRole"
    session_name = "TerraformSession"
    external_id  = "unique-external-id"  # For third-party access
  }

  # ── REGION ────────────────────────────────────────────────────────
  region = "us-east-1"

  # ── ENDPOINTS (for LocalStack or custom endpoints) ─────────────────
  endpoints {
    s3       = "http://localhost:4566"
    ec2      = "http://localhost:4566"
    dynamodb = "http://localhost:4566"
  }

  # ── ADVANCED ──────────────────────────────────────────────────────
  max_retries = 3          # API call retry count
  
  # Add default tags to ALL resources (AWS provider 3.38+)
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Owner       = "DevOps"
      CostCenter  = "Engineering"
    }
  }

  # Ignore specific tags on all resources (prevents drift)
  ignore_tags {
    keys = ["LastModifiedBy", "AutoScaling"]
  }
}
```

---

### Multiple Providers & Aliases

```hcl
# ── SCENARIO 1: Multi-region deployment ───────────────────────────

provider "aws" {
  region = "us-east-1"
  alias  = "us_east"    # Give it an alias
}

provider "aws" {
  region = "eu-west-1"
  alias  = "eu_west"
}

# Use specific provider via meta-argument
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east    # Uses us-east-1 provider
  bucket   = "my-bucket-us"
}

resource "aws_s3_bucket" "eu_bucket" {
  provider = aws.eu_west    # Uses eu-west-1 provider
  bucket   = "my-bucket-eu"
}


# ── SCENARIO 2: Multi-account deployment ──────────────────────────

provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "dev_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

resource "aws_instance" "prod_server" {
  provider      = aws.prod_account
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.large"
}

resource "aws_instance" "dev_server" {
  provider      = aws.dev_account
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}


# ── SCENARIO 3: Multiple providers (AWS + Cloudflare) ─────────────

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "cloudflare_record" "web_dns" {
  zone_id = var.cloudflare_zone_id
  name    = "www"
  type    = "A"
  value   = aws_instance.web.public_ip   # Cross-provider reference!
}
```

---

### Resource Meta-Arguments (Work on ANY Resource)

These are special arguments built into Terraform itself, not from the provider:

```hcl
# ── 1. depends_on — Explicit dependency ───────────────────────────
resource "aws_s3_bucket_policy" "policy" {
  bucket = aws_s3_bucket.main.id
  policy = data.aws_iam_policy_document.policy.json

  # Forces this to wait until public access block is done
  # even though there's no direct reference
  depends_on = [aws_s3_bucket_public_access_block.main]
}


# ── 2. count — Multiple identical resources ────────────────────────
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index + 1}"  # web-server-1, 2, 3
  }
}

# Reference: aws_instance.web[0], aws_instance.web[1], aws_instance.web[2]
output "web_ips" {
  value = aws_instance.web[*].public_ip   # Splat expression — all IPs
}


# ── 3. for_each — Multiple resources from map/set ─────────────────
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# Reference: aws_iam_user.team["alice"], aws_iam_user.team["bob"]

resource "aws_s3_bucket" "environments" {
  for_each = {
    dev  = "us-east-1"
    prod = "us-west-2"
  }
  bucket = "myapp-${each.key}"

  tags = {
    Environment = each.key
    Region      = each.value
  }
}


# ── 4. lifecycle — Control create/destroy behavior ────────────────
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  lifecycle {
    # Create new resource BEFORE destroying old one
    # Critical for zero-downtime updates
    create_before_destroy = true

    # Terraform will NEVER destroy this resource
    # Use for production databases
    prevent_destroy = true

    # Ignore changes to these attributes after creation
    # Useful for attributes managed externally
    ignore_changes = [
      ami,        # Ignore if someone manually updates AMI
      tags["LastModifiedBy"],  # Ignore auto-managed tags
    ]

    # Terraform 1.2+: Validate resource after creation
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP"
    }
  }
}


# ── 5. provider — Specify non-default provider ────────────────────
resource "aws_s3_bucket" "eu_logs" {
  provider = aws.eu_west   # Use aliased provider
  bucket   = "eu-logs-bucket"
}


# ── 6. timeouts — Control how long to wait ────────────────────────
resource "aws_db_instance" "main" {
  # ... db config ...

  timeouts {
    create = "60m"   # Wait up to 60 min for DB creation
    update = "80m"   
    delete = "40m"
  }
}
```

---

### Resource Attributes — Arguments vs Attributes

```hcl
resource "aws_instance" "web" {
  # ── ARGUMENTS (you set these) ─────────────────────────────────────
  ami           = "ami-0c55b159cbfafe1f0"   # Required argument
  instance_type = "t2.micro"                # Required argument
  key_name      = "my-keypair"             # Optional argument

  tags = {                                  # Block argument
    Name = "web-server"
  }
}

# ── ATTRIBUTES (Terraform sets these after creation) ───────────────
# These are ONLY available after the resource is created
output "instance_details" {
  value = {
    id              = aws_instance.web.id             # "i-1234567890abcdef0"
    public_ip       = aws_instance.web.public_ip      # "54.123.45.67"
    private_ip      = aws_instance.web.private_ip     # "10.0.1.5"
    arn             = aws_instance.web.arn             # "arn:aws:ec2:..."
    public_dns      = aws_instance.web.public_dns     # "ec2-54-123-45-67..."
    availability_zone = aws_instance.web.availability_zone
  }
}

# ── HOW TO FIND ALL ATTRIBUTES ────────────────────────────────────
# 1. registry.terraform.io → your resource → "Attributes Reference" section
# 2. terraform state show aws_instance.web   (after apply)
# 3. terraform console → type: aws_instance.web.  (tab complete)
```

---

### Data Sources — Reading Without Managing

```hcl
# ── USE CASE 1: Get latest Ubuntu AMI (don't hardcode!) ───────────
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical's AWS account

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id   # Always uses latest Ubuntu
  instance_type = "t2.micro"
}


# ── USE CASE 2: Reference existing VPC (not managed by this code) ──
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

resource "aws_subnet" "new_subnet" {
  vpc_id     = data.aws_vpc.existing.id
  cidr_block = "10.0.5.0/24"
}


# ── USE CASE 3: Get current AWS account ID ────────────────────────
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

# Use in resource policies
resource "aws_s3_bucket_policy" "policy" {
  bucket = aws_s3_bucket.main.id
  policy = jsonencode({
    Statement = [{
      Principal = {
        AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
      }
      Action   = "s3:*"
      Resource = aws_s3_bucket.main.arn
    }]
  })
}


# ── USE CASE 4: Read AWS availability zones ───────────────────────
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}


# ── USE CASE 5: IAM policy document ───────────────────────────────
data "aws_iam_policy_document" "assume_role" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "lambda_role" {
  name               = "lambda-execution-role"
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}
```

---

### When to Use Resource vs Data Source

```
Use RESOURCE when:                   Use DATA SOURCE when:
────────────────────────────────     ──────────────────────────────────
Creating new infrastructure          Reading existing infrastructure
Terraform should MANAGE lifecycle    Terraform should NOT manage it
You want it in state                 It's already created elsewhere
It doesn't exist yet                 Created by another team/tool/module
```

---

### Provider Authentication — Best Practices by Environment

```
┌─────────────────────────────────────────────────────────────────┐
│  ENVIRONMENT          RECOMMENDED AUTH METHOD                   │
├─────────────────────────────────────────────────────────────────┤
│  Local Dev            AWS Profile (~/.aws/credentials)          │
│  CI/CD (GitHub        OIDC (OpenID Connect) — no static keys!  │
│  Actions)                                                       │
│  EC2/EKS (in AWS)    IAM Instance Profile / IRSA               │
│  Cross-Account        IAM Role Assumption (assume_role)         │
│  Emergency/Quick      Environment Variables (temporary)         │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Environment variable authentication (temporary/CI)
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG"
export AWS_DEFAULT_REGION="us-east-1"
terraform apply
```

```hcl
# GitHub Actions OIDC (most secure for CI/CD)
# No static credentials at all!
provider "aws" {
  region = "us-east-1"
  # Credentials come from OIDC token automatically
}
```

---

### Interview-Focused Points 🎯

1. **What's the difference between `source` and `version` in `required_providers`?** `source` is the registry path (e.g., `hashicorp/aws`), `version` is the constraint. Together they ensure you get the exact right provider.

2. **What is `.terraform.lock.hcl`?** A lock file that pins exact provider versions and checksums — similar to `package-lock.json` in Node. Should be committed to Git for reproducibility.

3. **When would you use `provider aliases`?** Multi-region deployments, multi-account setups, or when you need different configurations of the same provider (e.g., different roles).

4. **What is `default_tags` in AWS provider?** A way to apply tags to ALL AWS resources managed by that provider block — reduces tag repetition across resources.

5. **What's the difference between `ignore_changes` and `prevent_destroy`?** `ignore_changes` tells Terraform to not update specific attributes even if they drift. `prevent_destroy` prevents Terraform from destroying the resource at all.

6. **Can resources from different providers reference each other?** Yes! Any resource attribute can be referenced cross-provider. This is powerful for building complete stacks (e.g., AWS instance IP → Cloudflare DNS record).

---

## 🔴 ADVANCED LEVEL

### Provider Plugin Protocol — gRPC Deep Dive

```
Terraform Core ←──── gRPC ────→ Provider Plugin

Provider exposes these gRPC methods:
├── GetProviderSchema()         Returns all resource types, their schemas
├── ValidateProviderConfig()    Validates your provider {} block
├── ConfigureProvider()         Authenticates with the cloud API
├── ValidateResourceConfig()    Validates your resource {} block
├── PlanResourceChange()        Calculates diff (what will change)
├── ApplyResourceChange()       Makes actual API calls (CRUD)
├── ImportResourceState()       terraform import support
├── ReadResource()              Refresh resource from real state
└── GetFunctions()              Provider-defined functions (TF 1.8+)
```

Understanding this means you know that **every `terraform plan`** actually calls your cloud API to get current resource state — that's why plan requires credentials.

---

### Resource Lifecycle — Complete Internal Flow

```
RESOURCE CREATION FLOW:
─────────────────────────────────────────────────────────────────
.tf file parsed
     │
     ▼
ValidateResourceConfig() ← Provider validates your arguments
     │
     ▼
PlanResourceChange()     ← Provider calculates what will be created
     │                      Returns: resource plan with all known values
     │                      Unknown values shown as (known after apply)
     ▼
[User confirms apply]
     │
     ▼
ApplyResourceChange()    ← Provider makes API calls
     │                      Returns: complete resource with all attributes
     ▼
State updated            ← Terraform saves ALL attributes to .tfstate


RESOURCE UPDATE FLOW:
─────────────────────────────────────────────────────────────────
You change argument in .tf
     │
     ▼
ReadResource()           ← Refresh actual cloud state
     │
     ▼
PlanResourceChange()     ← Diff: current state vs desired config
     │
     ├─── Can update in-place? ──► ApplyResourceChange() [UPDATE]
     │
     └─── Must recreate?       ──► ApplyResourceChange() [DELETE]
                                        │
                                        ▼
                                   ApplyResourceChange() [CREATE]


RESOURCE DESTROY FLOW:
─────────────────────────────────────────────────────────────────
terraform destroy or resource removed from .tf
     │
     ▼
PlanResourceChange()     ← Plan shows destruction
     │
     ▼
ApplyResourceChange()    ← DELETE API call
     │
     ▼
Removed from state
```

---

### Advanced Resource Patterns

**Pattern 1: Conditional Resource Creation**

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

# Create CloudWatch alarm only if monitoring enabled
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  count = var.enable_monitoring ? 1 : 0   # 1 = create, 0 = skip

  alarm_name          = "high-cpu-utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = 80
  dimensions = {
    InstanceId = aws_instance.web.id
  }
}

# Conditional resource reference
output "alarm_arn" {
  # one() extracts from list — returns null if empty
  value = one(aws_cloudwatch_metric_alarm.cpu_high[*].arn)
}
```

**Pattern 2: Resource with Complex Nested Blocks**

```hcl
resource "aws_security_group" "web" {
  name        = "web-security-group"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  # Multiple ingress blocks
  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description     = "SSH from bastion only"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]   # SG reference
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"         # All protocols
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags

  lifecycle {
    create_before_destroy = true   # Critical for SGs attached to instances
  }
}
```

**Pattern 3: Resource with `replace_triggered_by` (TF 1.2+)**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    # Force recreation if launch template changes
    replace_triggered_by = [
      aws_launch_template.web.latest_version
    ]
  }
}
```

**Pattern 4: Using `moved` block for refactoring (TF 1.1+)**

```hcl
# PROBLEM: You renamed a resource. Without this, Terraform would
# destroy old and create new (downtime!)

# OLD code was:  resource "aws_instance" "server" { ... }
# NEW code is:   resource "aws_instance" "web_server" { ... }

# SOLUTION: Tell Terraform about the rename
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}

# Terraform now moves state instead of destroy+create!
```

**Pattern 5: `import` block (TF 1.5+) — Declarative Import**

```hcl
# Import an existing EC2 instance into Terraform management
# WITHOUT running terraform import CLI command

import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"   # The actual AWS instance ID
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  # ... other config matching existing instance
}

# Run: terraform plan   (shows import plan)
# Run: terraform apply  (imports and brings under management)

# Even better: terraform plan -generate-config-out=generated.tf
# Terraform writes the resource config FOR YOU!
```

---

### Edge Cases & Gotchas

```hcl
# ── GOTCHA 1: Provider default_tags conflict ──────────────────────
provider "aws" {
  default_tags {
    tags = { Environment = "prod" }
  }
}

resource "aws_instance" "web" {
  tags = {
    Environment = "dev"   # This OVERRIDES the provider default tag
    Name        = "web"   # This MERGES with provider default tags
  }
}
# Final tags: { Environment = "dev", Name = "web", ManagedBy = "Terraform" }


# ── GOTCHA 2: destroy before create on some resources ─────────────
# Some AWS resources can't have two with same name simultaneously
# Without create_before_destroy, update fails because:
# Terraform tries to create new SG with same name → ERROR (already exists)
resource "aws_security_group" "web" {
  name = "web-sg"   # Name must be unique in VPC
  lifecycle {
    create_before_destroy = true   # MUST have this
  }
}


# ── GOTCHA 3: Sensitive attributes in state ───────────────────────
resource "aws_db_instance" "main" {
  password = var.db_password   # This is stored in PLAIN TEXT in state!
}
# Mitigation: Use sensitive = true on variable, encrypt state


# ── GOTCHA 4: for_each with computed values ───────────────────────
# This FAILS — for_each key can't be unknown until apply
resource "aws_instance" "web" {}

resource "aws_route53_record" "web" {
  for_each = toset(aws_instance.web[*].id)   # ERROR: unknown at plan time
}

# SOLUTION: Use count or a known set
resource "aws_route53_record" "web" {
  for_each = toset(["server-1", "server-2"])
}


# ── GOTCHA 5: depends_on disables automatic dependency detection ──
# Use depends_on ONLY when there's a hidden dependency
# Over-using it creates unnecessary serialization (slower!)
resource "aws_instance" "web" {
  depends_on = [aws_vpc.main]   # ❌ Unnecessary if you reference vpc_id
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

---

### Production Best Practices for Resources

```hcl
# ✅ BEST PRACTICE 1: Always add prevent_destroy to stateful resources
resource "aws_rds_cluster" "main" {
  cluster_identifier = "prod-db"
  lifecycle {
    prevent_destroy = true   # Prevents accidental terraform destroy
  }
}


# ✅ BEST PRACTICE 2: Use data source for AMI, never hardcode
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}


# ✅ BEST PRACTICE 3: Always tag everything (use locals for DRY)
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.team
    CostCenter  = var.cost_center
  }
}


# ✅ BEST PRACTICE 4: Use aws_iam_policy_document data source
# instead of inline JSON strings
data "aws_iam_policy_document" "s3_access" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject"]
    resources = ["${aws_s3_bucket.main.arn}/*"]
  }
}


# ✅ BEST PRACTICE 5: Postconditions for validation
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance failed to start"
    }
  }
}


# ✅ BEST PRACTICE 6: Use preconditions on variables
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod"
  }
}
```

---

### Debugging Provider Issues

```bash
# ─── Debug provider API calls ─────────────────────────────────────
export TF_LOG=DEBUG
terraform apply 2>&1 | grep -E "(Request|Response|Error)"

# ─── Check provider version in use ───────────────────────────────
terraform version
terraform providers

# ─── Inspect provider schema ─────────────────────────────────────
terraform providers schema -json | jq '.provider_schemas."registry.terraform.io/hashicorp/aws".resource_schemas | keys'

# ─── Test provider auth ───────────────────────────────────────────
aws sts get-caller-identity   # Verify AWS credentials work

# ─── Upgrade provider ────────────────────────────────────────────
terraform init -upgrade

# ─── Remove provider lock (re-resolve) ───────────────────────────
rm .terraform.lock.hcl
terraform init

# ─── See what a resource looks like in state ─────────────────────
terraform state show aws_instance.web

# ─── Validate resource config ────────────────────────────────────
terraform validate

# ─── Interactive console to test expressions ──────────────────────
terraform console
> aws_instance.web.public_ip
> data.aws_ami.ubuntu.id
> local.common_tags
```

---

## 📁 CODE WRITING GUIDANCE

### Professional File Structure for Provider-Heavy Projects

```
project/
├── providers.tf          # ALL provider configurations
├── versions.tf           # terraform {} block with required_providers  
├── main.tf               # Primary resources
├── data.tf               # ALL data sources
├── variables.tf          # Variable declarations
├── locals.tf             # Local values
├── outputs.tf            # Output values
├── terraform.tfvars      # Variable values (gitignored if sensitive)
└── modules/
    └── web-tier/
        ├── providers.tf  # Module provider requirements (if needed)
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

---

### How Professionals Write Provider Config

```hcl
# versions.tf — separate file for version constraints
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.31"   # Specific minor, allows patches
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
  }

  backend "s3" {
    bucket         = "mycompany-tf-state"
    key            = "services/web-app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tf-state-lock"
  }
}
```

```hcl
# providers.tf — separate file for provider configuration
provider "aws" {
  region = var.aws_region

  # All resources get these tags automatically
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Repository  = "github.com/myorg/infra"
      Team        = var.team_name
      Environment = var.environment
    }
  }

  # Ignore auto-managed tags that change externally
  ignore_tags {
    key_prefixes = ["kubernetes.io/", "k8s.io/"]
  }
}

# Second region for DR
provider "aws" {
  alias  = "dr"
  region = "us-west-2"

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Region      = "dr"
    }
  }
}
```

---

## 🧪 HANDS-ON LAB

### Exercise: Multi-Resource AWS Infrastructure

**Goal:** Build a complete, working VPC setup with:
1. A VPC
2. 2 Public subnets in different AZs
3. 1 Internet Gateway
4. 1 Route table with association
5. 1 Security group (allows HTTP + SSH)
6. 1 EC2 instance in the first public subnet
7. Outputs for VPC ID, subnet IDs, and instance public IP

**Constraints:**
- Use `data "aws_availability_zones"` for AZ names (don't hardcode)
- Use `data "aws_ami"` for Ubuntu (don't hardcode AMI)
- Use `locals` for common tags
- All resources must have Name tags
- Use variables for: `environment`, `project_name`, `instance_type`

**Expected Output:**
```
Apply complete! Resources: 8 added, 0 changed, 0 destroyed.

Outputs:

instance_public_ip = "54.123.45.67"
subnet_ids = [
  "subnet-0abc1234",
  "subnet-0def5678",
]
vpc_id = "vpc-0a1b2c3d4e5f"
website_url = "http://54.123.45.67"
```

**Try it yourself first!**

---

### ✅ Solution

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

```hcl
# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = var.project_name
      Environment = var.environment
    }
  }
}
```

```hcl
# variables.tf
variable "aws_region" {
  type        = string
  description = "AWS region to deploy resources"
  default     = "us-east-1"
}

variable "project_name" {
  type        = string
  description = "Name of the project"
  default     = "terraform-lab"
}

variable "environment" {
  type        = string
  description = "Environment name"
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}
```

```hcl
# locals.tf
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  public_subnet_cidrs = [
    cidrsubnet(var.vpc_cidr, 8, 1),   # 10.0.1.0/24
    cidrsubnet(var.vpc_cidr, 8, 2),   # 10.0.2.0/24
  ]
}
```

```hcl
# data.tf
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

```hcl
# main.tf

# ── VPC ───────────────────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

# ── SUBNETS ───────────────────────────────────────────────────────
resource "aws_subnet" "public" {
  count = 2

  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.name_prefix}-public-subnet-${count.index + 1}"
    Type = "Public"
  }
}

# ── INTERNET GATEWAY ──────────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

# ── ROUTE TABLE ───────────────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${local.name_prefix}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count = 2

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ── SECURITY GROUP ────────────────────────────────────────────────
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Allow HTTP and SSH traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH from anywhere (use bastion in prod!)"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${local.name_prefix}-web-sg"
  }

  lifecycle {
    create_before_destroy = true
  }
}

# ── EC2 INSTANCE ──────────────────────────────────────────────────
resource "aws_instance" "web" {
  ami                    = data.aws_ami.ubuntu.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    echo "<h1>Deployed by Terraform | ${local.name_prefix}</h1>" \
      > /var/www/html/index.html
  EOF

  tags = {
    Name = "${local.name_prefix}-web-server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

```hcl
# outputs.tf
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "instance_public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "website_url" {
  description = "URL to access the website"
  value       = "http://${aws_instance.web.public_ip}"
}

output "ami_used" {
  description = "AMI ID used for the instance"
  value       = data.aws_ami.ubuntu.id
}
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| Provider | Plugin that talks to cloud API. Always pin version. |
| Provider alias | Multiple instances of same provider (multi-region/account) |
| Resource | Infrastructure to create. Has arguments (you set) and attributes (provider sets) |
| Data source | Read-only. Reads existing infra. Doesn't create anything. |
| `depends_on` | Explicit dependency. Use only when reference isn't possible. |
| `lifecycle` | Controls create/destroy behavior. `prevent_destroy` for prod. |
| `count` | Simple repetition. Fragile when list changes. |
| `for_each` | Map/set based. Stable identity. Preferred over count. |
| `moved` block | Rename resources without destroy+recreate. |
| `import` block | Bring existing resources under Terraform control. |

---

### Quick Syntax Reference

```hcl
# Provider with alias
provider "aws" { alias = "name"; region = "..." }

# Use aliased provider
resource "..." "..." { provider = aws.name }

# Data source
data "aws_ami" "name" { owners = [...]; filter { ... } }
# Reference: data.aws_ami.name.id

# Resource attribute reference
resource_type.resource_name.attribute

# Meta-arguments
resource "..." "..." {
  count      = number
  for_each   = map_or_set
  depends_on = [resource.name]
  provider   = provider.alias
  
  lifecycle {
    create_before_destroy = bool
    prevent_destroy       = bool
    ignore_changes        = [attribute]
  }
}

# count references
count.index                    # Current index
resource.name[count.index]     # Specific instance
resource.name[*].attribute     # All instances (splat)

# for_each references
each.key                       # Current key
each.value                     # Current value
resource.name["key"]           # Specific instance
```

---

### Top 8 Interview Questions

1. **How does Terraform know which provider to use for a resource?** The resource type prefix (e.g., `aws_`) maps to the provider. For ambiguous cases or aliases, use the `provider` meta-argument.

2. **What is `.terraform.lock.hcl` and should it be committed?** It pins exact provider versions and checksums. **Yes, commit it** — ensures your whole team uses the same provider version.

3. **How do you manage infrastructure across multiple AWS accounts?** Use provider aliases with `assume_role` to switch between accounts in the same Terraform config.

4. **What's the difference between `ignore_changes` and importing?** `ignore_changes` tells Terraform to not manage specific attributes after creation. `import` brings externally created resources fully under Terraform management.

5. **When would `create_before_destroy = true` cause issues?** When the new resource can't be created while the old one exists (e.g., unique name constraints). Solution: use random suffix or different naming.

6. **How do you reference a resource's output from a data source?** `data.TYPE.NAME.ATTRIBUTE` — e.g., `data.aws_ami.ubuntu.id`

7. **Can you use Terraform to manage resources created outside Terraform?** Yes, using `terraform import` (CLI) or `import {}` block (TF 1.5+). You bring them into state and then manage them going forward.

8. **What happens to resources if you run `terraform apply` with no changes in your config?** Terraform refreshes state (reads from cloud), compares, shows "No changes. Infrastructure is up-to-date." Nothing is modified.

---
