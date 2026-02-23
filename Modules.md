# 🧩 TOPIC 5: Modules — Building Production-Grade Reusable Infrastructure

---

## 🟢 BEGINNER LEVEL

### What is a Terraform Module? (Simple Terms)

Imagine you're a **chef at a restaurant chain** with 50 locations. Every location needs the same kitchen setup — same equipment, same layout, same safety systems. 

You have two choices:
- **Without modules:** Go to each location and manually set up everything from scratch, 50 times, making small mistakes each time
- **With modules:** Create one **master kitchen blueprint**, customize it per location (size, color), and stamp it out 50 times — perfectly consistent

**Terraform modules are those blueprints.** Write infrastructure once, deploy it anywhere, customize it with variables.

```
WITHOUT MODULES                    WITH MODULES
─────────────────────              ─────────────────────────────
dev/main.tf   (500 lines)          modules/web-app/main.tf (200 lines)
staging/main.tf (498 lines)              ↑ written ONCE
prod/main.tf  (503 lines)          
                                   dev/main.tf     (20 lines) ← calls module
All nearly identical               staging/main.tf (20 lines) ← calls module
Bugs fixed in 3 places             prod/main.tf    (20 lines) ← calls module
Drift inevitable                   Bug fixed ONCE, fixed everywhere
```

---

### Real-World Analogy

```
LEGO SET ANALOGY:
─────────────────────────────────────────────────────────────────
LEGO Brick        →  Individual resource (aws_instance, aws_vpc)
LEGO Kit/Set      →  Module (collection of resources with purpose)
Instructions Book →  Module variables (how to customize it)
Finished Model    →  Module output (what you get when done)
LEGO Store        →  Terraform Registry (pre-built modules)

You can:
✅ Build your own custom LEGO set (local module)
✅ Buy a pre-made LEGO set (registry module)
✅ Combine multiple sets into bigger creation (nested modules)
✅ Make 10 copies of same set with different colors (multiple instances)
```

---

### Types of Modules

```
┌─────────────────────────────────────────────────────────────────┐
│                      MODULE TYPES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ROOT MODULE     → Your main working directory                  │
│                    Every Terraform config IS a root module      │
│                    Contains: main.tf, variables.tf, outputs.tf  │
│                                                                 │
│  CHILD MODULE    → Called by root or another module             │
│                    Lives in: ./modules/  or external source     │
│                    Has its own variables, resources, outputs    │
│                                                                 │
│  LOCAL MODULE    → Stored in your own repository               │
│                    Source: "./modules/my-module"                │
│                                                                 │
│  REGISTRY MODULE → From registry.terraform.io                  │
│                    Source: "hashicorp/consul/aws"               │
│                                                                 │
│  GIT MODULE      → From a Git repository                       │
│                    Source: "git::https://github.com/org/repo"   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Basic Module Syntax

```hcl
# Calling a module (in root main.tf)
module "MODULE_LABEL" {
  source = "PATH_OR_REGISTRY"    # Required: where module lives

  # Input variables (defined in module's variables.tf)
  variable_name = value
  another_var   = value
}

# Accessing module outputs
module.MODULE_LABEL.OUTPUT_NAME
```

---

### Your First Module — Minimal Working Example

**File structure:**
```
my-project/
├── main.tf                    ← Root module (calls child modules)
├── variables.tf
├── outputs.tf
└── modules/
    └── s3-bucket/             ← Child module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

```hcl
# modules/s3-bucket/variables.tf
variable "bucket_name" {
  type        = string
  description = "Name of the S3 bucket"
}

variable "environment" {
  type        = string
  description = "Environment tag"
}

variable "versioning_enabled" {
  type        = bool
  description = "Enable versioning"
  default     = false
}
```

```hcl
# modules/s3-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```hcl
# modules/s3-bucket/outputs.tf
output "bucket_id" {
  description = "The name of the bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "The ARN of the bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "The domain name of the bucket"
  value       = aws_s3_bucket.this.bucket_domain_name
}
```

```hcl
# root main.tf — calling the module multiple times
module "app_assets" {
  source = "./modules/s3-bucket"

  bucket_name        = "myapp-dev-assets"
  environment        = "dev"
  versioning_enabled = false
}

module "app_logs" {
  source = "./modules/s3-bucket"

  bucket_name        = "myapp-dev-logs"
  environment        = "dev"
  versioning_enabled = true   # Different config, same module
}

# Using module outputs
output "assets_bucket_arn" {
  value = module.app_assets.bucket_arn
}
```

```bash
# Commands
terraform init      # Downloads/initializes modules
terraform plan
terraform apply
```

---

### Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Accessing resource directly from outside module
# Modules are BLACK BOXES — you can only use their outputs
module.my_module.aws_s3_bucket.this.id   # ❌ Can't reach inside!
module.my_module.bucket_id               # ✅ Use outputs

# ❌ MISTAKE 2: Forgetting to run terraform init after adding module
# Every new module source requires: terraform init

# ❌ MISTAKE 3: No outputs in module — makes it useless to callers
# modules/s3-bucket/main.tf  ← creates bucket
# modules/s3-bucket/outputs.tf ← EMPTY  ← caller can't reference anything!

# ❌ MISTAKE 4: Hardcoding values inside module
# modules/ec2/main.tf
resource "aws_instance" "web" {
  instance_type = "t2.micro"  # ❌ Hardcoded — not reusable!
}
# ✅ Use variables for anything that might change

# ❌ MISTAKE 5: Relative path wrong when calling module
module "app" {
  source = "modules/s3-bucket"    # ❌ Missing ./
  source = "./modules/s3-bucket"  # ✅ Correct
}

# ❌ MISTAKE 6: Putting provider config inside a module
# modules/ec2/main.tf
provider "aws" {           # ❌ NEVER put provider in child module
  region = "us-east-1"
}
# Providers belong ONLY in root module or specific cases
```

---

## 🟡 INTERMEDIATE LEVEL

### Module Sources — All Types

```hcl
# ── LOCAL PATH ────────────────────────────────────────────────────
module "networking" {
  source = "./modules/networking"    # Relative path
}

module "compute" {
  source = "../shared/modules/ec2"   # Up a directory
}


# ── TERRAFORM REGISTRY (Public) ───────────────────────────────────
# Format: "<NAMESPACE>/<MODULE>/<PROVIDER>"
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"                  # ALWAYS pin version!
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
}


# ── GITHUB (Public Repo) ──────────────────────────────────────────
module "security" {
  source = "github.com/myorg/terraform-modules//modules/security"
  # Double slash // = subdirectory within repo
}

# With specific ref (tag, branch, commit)
module "security" {
  source = "github.com/myorg/terraform-modules//modules/security?ref=v2.1.0"
}


# ── GIT GENERIC ───────────────────────────────────────────────────
module "internal" {
  source = "git::https://gitlab.mycompany.com/infra/modules.git//networking?ref=v1.5.0"
}

# SSH authentication
module "private" {
  source = "git::ssh://git@github.com/myorg/private-modules.git//vpc?ref=main"
}


# ── S3 BUCKET (for private distribution) ─────────────────────────
module "approved" {
  source = "s3::https://s3-us-east-1.amazonaws.com/my-modules/networking.zip"
}


# ── TERRAFORM CLOUD PRIVATE REGISTRY ─────────────────────────────
module "vpc" {
  source  = "app.terraform.io/myorg/vpc/aws"
  version = "~> 2.0"
}
```

---

### Using Public Registry Modules

```hcl
# The most popular AWS modules from terraform-aws-modules
# github.com/terraform-aws-modules — battle-tested, production-ready

# ── VPC Module ────────────────────────────────────────────────────
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false   # One per AZ for HA
  enable_vpn_gateway     = false
  enable_dns_hostnames   = true

  tags = local.common_tags
}

# Access outputs
output "vpc_id" { value = module.vpc.vpc_id }
output "private_subnets" { value = module.vpc.private_subnets }
output "public_subnets" { value = module.vpc.public_subnets }


# ── EKS Module ────────────────────────────────────────────────────
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.29"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 5
      desired_size   = 3
    }
  }
}


# ── RDS Module ────────────────────────────────────────────────────
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier     = "my-db"
  engine         = "postgres"
  engine_version = "14"
  instance_class = "db.t3.large"

  allocated_storage     = 20
  max_allocated_storage = 100   # Auto-scaling storage

  db_name  = "myappdb"
  username = "dbadmin"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.db.id]
  subnet_ids             = module.vpc.private_subnets

  family               = "postgres14"
  major_engine_version = "14"

  deletion_protection = true
  skip_final_snapshot = false

  tags = local.common_tags
}
```

---

### Building a Production Module — Complete Example

```
modules/
└── web-app/
    ├── main.tf            ← All resources
    ├── variables.tf       ← Input contract
    ├── outputs.tf         ← Output contract
    ├── locals.tf          ← Internal computations
    ├── data.tf            ← Data sources
    └── README.md          ← Documentation
```

```hcl
# modules/web-app/variables.tf

# ── REQUIRED (no default) ─────────────────────────────────────────
variable "name" {
  type        = string
  description = "Name of the web application. Used as prefix for all resources."

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,20}$", var.name))
    error_message = "Name must be 2-21 chars, lowercase letters, numbers, hyphens."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment (dev/staging/prod)."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be: dev, staging, or prod."
  }
}

variable "vpc_id" {
  type        = string
  description = "ID of the VPC to deploy into."
}

variable "public_subnet_ids" {
  type        = list(string)
  description = "List of public subnet IDs for the load balancer."
}

variable "private_subnet_ids" {
  type        = list(string)
  description = "List of private subnet IDs for EC2 instances."
}

# ── OPTIONAL (with sensible defaults) ─────────────────────────────
variable "instance_type" {
  type        = string
  description = "EC2 instance type."
  default     = null   # null = use environment-based default from locals
}

variable "min_instances" {
  type        = number
  description = "Minimum number of EC2 instances in ASG."
  default     = null   # null = use environment-based default
}

variable "max_instances" {
  type        = number
  description = "Maximum number of EC2 instances in ASG."
  default     = null   # null = use environment-based default
}

variable "health_check_path" {
  type        = string
  description = "HTTP path for health check."
  default     = "/health"
}

variable "app_port" {
  type        = number
  description = "Port the application listens on."
  default     = 8080
}

variable "enable_https" {
  type        = bool
  description = "Enable HTTPS with ACM certificate."
  default     = false
}

variable "certificate_arn" {
  type        = string
  description = "ACM certificate ARN. Required if enable_https = true."
  default     = null
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources."
  default     = {}
}

# ── COMPLEX TYPE ──────────────────────────────────────────────────
variable "alarms" {
  type = object({
    cpu_threshold    = number
    memory_threshold = number
    sns_topic_arn    = string
  })
  description = "CloudWatch alarm configuration."
  default     = null   # null = no alarms created
}
```

```hcl
# modules/web-app/locals.tf
locals {
  # ── Environment-based defaults ────────────────────────────────
  env_defaults = {
    dev = {
      instance_type = "t3.micro"
      min_instances = 1
      max_instances = 2
    }
    staging = {
      instance_type = "t3.small"
      min_instances = 1
      max_instances = 3
    }
    prod = {
      instance_type = "t3.medium"
      min_instances = 2
      max_instances = 10
    }
  }

  # Resolve: use provided value OR fall back to env default
  instance_type = coalesce(var.instance_type,
                           local.env_defaults[var.environment].instance_type)
  min_instances = coalesce(var.min_instances,
                           local.env_defaults[var.environment].min_instances)
  max_instances = coalesce(var.max_instances,
                           local.env_defaults[var.environment].max_instances)

  # ── Naming ────────────────────────────────────────────────────
  prefix = "${var.name}-${var.environment}"

  # ── Tags ──────────────────────────────────────────────────────
  common_tags = merge(
    {
      Name        = local.prefix
      Module      = "web-app"
      Environment = var.environment
      ManagedBy   = "Terraform"
    },
    var.tags
  )

  # ── Conditional flags ─────────────────────────────────────────
  create_https_listener = var.enable_https && var.certificate_arn != null
  create_alarms         = var.alarms != null
}
```

```hcl
# modules/web-app/data.tf
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

```hcl
# modules/web-app/main.tf

# ── Security Group ────────────────────────────────────────────────
resource "aws_security_group" "alb" {
  name        = "${local.prefix}-alb-sg"
  description = "Security group for ALB"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  dynamic "ingress" {
    for_each = local.create_https_listener ? [1] : []
    content {
      description = "HTTPS"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, { Name = "${local.prefix}-alb-sg" })

  lifecycle { create_before_destroy = true }
}

resource "aws_security_group" "app" {
  name        = "${local.prefix}-app-sg"
  description = "Security group for application instances"
  vpc_id      = var.vpc_id

  ingress {
    description     = "App port from ALB only"
    from_port       = var.app_port
    to_port         = var.app_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, { Name = "${local.prefix}-app-sg" })

  lifecycle { create_before_destroy = true }
}

# ── Application Load Balancer ────────────────────────────────────
resource "aws_lb" "this" {
  name               = "${local.prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = var.environment == "prod"

  tags = local.common_tags
}

resource "aws_lb_target_group" "this" {
  name        = "${local.prefix}-tg"
  port        = var.app_port
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "instance"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    path                = var.health_check_path
    matcher             = "200"
  }

  lifecycle { create_before_destroy = true }

  tags = local.common_tags
}

# HTTP Listener (always created)
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.this.arn
  port              = 80
  protocol          = "HTTP"

  # If HTTPS enabled: redirect HTTP to HTTPS
  # If no HTTPS: forward to target group
  default_action {
    type = local.create_https_listener ? "redirect" : "forward"

    dynamic "redirect" {
      for_each = local.create_https_listener ? [1] : []
      content {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }

    dynamic "forward" {
      for_each = local.create_https_listener ? [] : [1]
      content {
        target_group {
          arn = aws_lb_target_group.this.arn
        }
      }
    }
  }
}

# HTTPS Listener (conditionally created)
resource "aws_lb_listener" "https" {
  count = local.create_https_listener ? 1 : 0

  load_balancer_arn = aws_lb.this.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.this.arn
  }
}

# ── Launch Template ───────────────────────────────────────────────
resource "aws_launch_template" "this" {
  name_prefix   = "${local.prefix}-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = local.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]

  user_data = base64encode(templatefile(
    "${path.module}/templates/user_data.sh.tpl",
    {
      app_port  = var.app_port
      env       = var.environment
    }
  ))

  monitoring { enabled = var.environment == "prod" }

  tag_specifications {
    resource_type = "instance"
    tags          = merge(local.common_tags, { Name = "${local.prefix}-instance" })
  }

  lifecycle { create_before_destroy = true }
}

# ── Auto Scaling Group ────────────────────────────────────────────
resource "aws_autoscaling_group" "this" {
  name                = "${local.prefix}-asg"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.this.arn]
  health_check_type   = "ELB"

  min_size         = local.min_instances
  max_size         = local.max_instances
  desired_capacity = local.min_instances

  launch_template {
    id      = aws_launch_template.this.id
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

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [desired_capacity]   # Allow external scaling
  }
}

# ── CloudWatch Alarms (optional) ──────────────────────────────────
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  count = local.create_alarms ? 1 : 0

  alarm_name          = "${local.prefix}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = var.alarms.cpu_threshold
  alarm_description   = "CPU utilization too high"
  alarm_actions       = [var.alarms.sns_topic_arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.this.name
  }

  tags = local.common_tags
}
```

```hcl
# modules/web-app/outputs.tf

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.this.dns_name
}

output "alb_arn" {
  description = "ARN of the Application Load Balancer"
  value       = aws_lb.this.arn
}

output "target_group_arn" {
  description = "ARN of the ALB Target Group"
  value       = aws_lb_target_group.this.arn
}

output "asg_name" {
  description = "Name of the Auto Scaling Group"
  value       = aws_autoscaling_group.this.name
}

output "app_security_group_id" {
  description = "ID of the application security group"
  value       = aws_security_group.app.id
}

output "alb_security_group_id" {
  description = "ID of the ALB security group"
  value       = aws_security_group.alb.id
}

output "app_url" {
  description = "URL to access the application"
  value       = local.create_https_listener ? "https://${aws_lb.this.dns_name}" : "http://${aws_lb.this.dns_name}"
}
```

---

### Module Composition — Calling Multiple Modules

```hcl
# root/main.tf — Orchestrating modules together

# ── Layer 1: Networking ───────────────────────────────────────────
module "networking" {
  source = "./modules/networking"

  environment         = var.environment
  vpc_cidr            = var.vpc_cidr
  availability_zones  = var.availability_zones
}

# ── Layer 2: Database ─────────────────────────────────────────────
module "database" {
  source = "./modules/database"

  environment        = var.environment
  name               = var.app_name
  vpc_id             = module.networking.vpc_id          # ← from networking
  subnet_ids         = module.networking.private_subnet_ids  # ← from networking
  db_password        = var.db_password

  depends_on = [module.networking]   # Explicit just for clarity
}

# ── Layer 3: Application ──────────────────────────────────────────
module "web_app" {
  source = "./modules/web-app"

  name               = var.app_name
  environment        = var.environment
  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.public_subnet_ids
  private_subnet_ids = module.networking.private_subnet_ids

  enable_https    = var.environment == "prod"
  certificate_arn = var.certificate_arn

  alarms = var.environment == "prod" ? {
    cpu_threshold    = 80
    memory_threshold = 85
    sns_topic_arn    = module.monitoring.alert_topic_arn
  } : null
}

# ── Layer 4: Monitoring ───────────────────────────────────────────
module "monitoring" {
  source = "./modules/monitoring"

  environment    = var.environment
  app_name       = var.app_name
  asg_name       = module.web_app.asg_name     # ← from web_app
  alb_arn        = module.web_app.alb_arn      # ← from web_app
  alert_email    = var.alert_email
}

# ── Root Outputs ──────────────────────────────────────────────────
output "application_url" {
  value = module.web_app.app_url
}

output "database_endpoint" {
  value = module.database.endpoint
}
```

---

### Module Versioning Strategy

```hcl
# ── VERSION PINNING OPTIONS ───────────────────────────────────────

# Exact version (most strict — safest for prod)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"         # Exactly this version
}

# Pessimistic constraint (allows patches only: 5.1.x)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"        # 5.1.0, 5.1.1, 5.1.2 ... but NOT 5.2.0
}

# Minor updates allowed (5.x.x)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"        # Any 5.x.x
}

# Range constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0, < 6.0"
}

# Git tag (for private modules)
module "internal_vpc" {
  source = "git::https://github.com/myorg/tf-modules.git//vpc?ref=v2.3.1"
}

# Git SHA (most pinned — for critical modules)
module "internal_vpc" {
  source = "git::https://github.com/myorg/tf-modules.git//vpc?ref=abc1234def5678"
}
```

---

### Interview-Focused Points 🎯

1. **What is the difference between root and child modules?** The root module is where you run Terraform commands from — your working directory. Child modules are called from the root or other modules via a `module` block.

2. **Can a module call another module?** Yes — these are called nested modules. However, deep nesting (3+ levels) makes debugging hard. Keep nesting shallow.

3. **How do you pass provider configuration to a module?** Via the `providers` meta-argument in the module block. Modules inherit the default provider by default but can be given aliased providers explicitly.

4. **Why should you NEVER put provider blocks inside child modules?** It creates tight coupling between module and environment, prevents multi-provider use, and causes unexpected behavior. Providers belong in root modules only.

5. **What is module encapsulation?** Modules are black boxes. Callers interact only through variables (inputs) and outputs. Internal resources are not directly accessible from outside.

6. **How do you update a module version?** Change the `version` constraint, run `terraform init -upgrade`, then `terraform plan` to review changes.

---

## 🔴 ADVANCED LEVEL

### How Modules Work Internally

```
terraform init
      │
      ├── Reads all module sources
      ├── Downloads registry modules to: .terraform/modules/
      ├── Creates: .terraform/modules/modules.json (manifest)
      └── Local modules: referenced by path (no download)

terraform plan/apply
      │
      ├── Builds unified resource graph across ALL modules
      ├── Module boundaries are TRANSPARENT to graph builder
      ├── Resources from modules are addressed as:
      │   module.NAME.resource_type.resource_name
      │   module.outer.module.inner.resource_type.name  (nested)
      │
      └── State stores all module resources with full address:
          "module.networking.aws_vpc.main"
          "module.web_app.aws_lb.this"
```

```bash
# .terraform/modules/modules.json after init
{
  "Modules": [
    {
      "Key": "networking",
      "Source": "registry.terraform.io/terraform-aws-modules/vpc/aws",
      "Version": "5.1.2",
      "Dir": ".terraform/modules/networking"
    },
    {
      "Key": "web_app",
      "Source": "./modules/web-app",
      "Dir": "modules/web-app"
    }
  ]
}
```

---

### Advanced Module Patterns

**Pattern 1: Module with `for_each` — Deploy Multiple Instances**

```hcl
# Deploy the same module multiple times from a map

variable "environments" {
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = number
  }))
  default = {
    dev = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 1
      max_size      = 3
    }
    prod = {
      instance_type = "t3.medium"
      min_size      = 3
      max_size      = 10
    }
  }
}

module "web_app" {
  source   = "./modules/web-app"
  for_each = var.environments

  name          = "myapp"
  environment   = each.key                    # "dev", "staging", "prod"
  instance_type = each.value.instance_type
  min_instances = each.value.min_size
  max_instances = each.value.max_size

  vpc_id             = module.networking[each.key].vpc_id
  public_subnet_ids  = module.networking[each.key].public_subnet_ids
  private_subnet_ids = module.networking[each.key].private_subnet_ids
}

# Access specific environment's outputs
output "prod_url" {
  value = module.web_app["prod"].app_url
}

# Access all environment URLs
output "all_urls" {
  value = { for env, mod in module.web_app : env => mod.app_url }
}
```

---

**Pattern 2: Passing Providers to Modules (Multi-Region)**

```hcl
# root/providers.tf
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# root/main.tf
module "app_us" {
  source = "./modules/web-app"

  providers = {
    aws = aws.us_east   # Pass specific provider to module
  }

  name        = "myapp"
  environment = "prod"
}

module "app_eu" {
  source = "./modules/web-app"

  providers = {
    aws = aws.eu_west   # Same module, different region
  }

  name        = "myapp"
  environment = "prod"
}
```

```hcl
# modules/web-app/providers.tf
# Declare that this module requires an aws provider
# (but doesn't configure it — root does that)
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

**Pattern 3: Optional Module Features with `count`**

```hcl
# modules/web-app/main.tf

# Feature: WAF protection (only in prod)
resource "aws_wafv2_web_acl_association" "this" {
  count = var.enable_waf ? 1 : 0

  resource_arn = aws_lb.this.arn
  web_acl_arn  = var.waf_acl_arn
}

# Feature: Custom domain (only when provided)
resource "aws_route53_record" "this" {
  count = var.domain_name != null ? 1 : 0

  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.this.dns_name
    zone_id                = aws_lb.this.zone_id
    evaluate_target_health = true
  }
}

# Outputs handle optional resources safely
output "domain_url" {
  description = "Custom domain URL if configured"
  value       = var.domain_name != null ? "https://${var.domain_name}" : null
}
```

---

**Pattern 4: Module Abstraction Layer — Wrapping Public Modules**

```hcl
# modules/networking/main.tf
# Internal wrapper around public VPC module
# Enforces company standards, hides complexity

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "${var.name}-${var.environment}"
  cidr = var.vpc_cidr

  azs = slice(
    data.aws_availability_zones.available.names,
    0,
    var.az_count
  )

  private_subnets = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 4, i)
  ]

  public_subnets = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 4, i + 10)
  ]

  # Company standards — always enforced, not exposed as variables
  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"  # HA in prod
  enable_dns_hostnames   = true
  enable_dns_support     = true

  # VPC Flow Logs — always on (compliance requirement)
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
  flow_log_max_aggregation_interval    = 60

  tags = var.tags
}
```

---

**Pattern 5: Module Output Aggregation**

```hcl
# When module uses for_each, aggregate outputs elegantly

module "microservices" {
  source   = "./modules/ecs-service"
  for_each = var.services   # map of service configs

  name        = each.key
  environment = var.environment
  # ...
}

# Aggregate all service URLs into a single map
output "service_urls" {
  value = {
    for name, svc in module.microservices :
    name => svc.service_url
  }
}

# Aggregate all task definition ARNs
output "task_definitions" {
  value = {
    for name, svc in module.microservices :
    name => svc.task_definition_arn
  }
}

# Find services that failed health check (advanced)
output "unhealthy_services" {
  value = [
    for name, svc in module.microservices :
    name if !svc.is_healthy
  ]
}
```

---

### Building an Enterprise Module Library

```
terraform-modules/               ← Separate Git repository
├── README.md
├── CHANGELOG.md
├── .github/
│   └── workflows/
│       ├── validate.yml         ← CI: fmt + validate + tflint
│       └── release.yml          ← CD: tag + release on merge
│
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── README.md            ← Auto-generated by terraform-docs
│   │   ├── examples/
│   │   │   ├── basic/
│   │   │   │   ├── main.tf
│   │   │   │   └── README.md
│   │   │   └── complete/
│   │   │       ├── main.tf
│   │   │       └── README.md
│   │   └── tests/
│   │       └── networking_test.go  ← Terratest
│   │
│   ├── compute/
│   ├── database/
│   ├── monitoring/
│   └── security/
│
└── _templates/                  ← Module scaffolding template
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

```yaml
# .github/workflows/validate.yml
name: Validate Terraform Modules

on:
  pull_request:
    paths: ['modules/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Validate each module
        run: |
          for dir in modules/*/; do
            echo "Validating $dir"
            cd "$dir"
            terraform init -backend=false
            terraform validate
            cd -
          done

      - name: Run tflint
        uses: terraform-linters/setup-tflint@v4
        # ...

      - name: Generate docs
        uses: terraform-docs/gh-actions@v1
        with:
          working-dir: modules/
          recursive: true
```

---

### Module Testing with Terratest

```go
// modules/networking/tests/networking_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
  "github.com/stretchr/testify/assert"
)

func TestNetworkingModule(t *testing.T) {
  t.Parallel()

  terraformOptions := &terraform.Options{
    TerraformDir: "../examples/basic",
    Vars: map[string]interface{}{
      "environment": "test",
      "vpc_cidr":    "10.99.0.0/16",
    },
  }

  // Destroy at end of test
  defer terraform.Destroy(t, terraformOptions)

  // Deploy
  terraform.InitAndApply(t, terraformOptions)

  // Get outputs
  vpcID := terraform.Output(t, terraformOptions, "vpc_id")

  // Assert VPC exists in AWS
  vpc := aws.GetVpcById(t, vpcID, "us-east-1")
  assert.Equal(t, "10.99.0.0/16", aws.GetVpcCidr(t, vpcID, "us-east-1"))
  assert.Equal(t, true, vpc.EnableDnsHostnames)
}
```

---

### Edge Cases & Gotchas

```hcl
# ── GOTCHA 1: Module outputs can't use sensitive from resource ────
# If resource has sensitive attribute, output must be marked sensitive

# modules/database/outputs.tf
output "master_password" {
  value     = aws_db_instance.this.password
  sensitive = true   # REQUIRED — password is sensitive attribute
}


# ── GOTCHA 2: Module with for_each — keys must be known at plan ───
# ❌ This fails because count.index isn't known until apply
module "services" {
  for_each = toset(aws_instance.web[*].id)  # Unknown at plan time!
  source   = "./modules/service"
}

# ✅ Use known values
module "services" {
  for_each = toset(["web", "api", "worker"])
  source   = "./modules/service"
  name     = each.key
}


# ── GOTCHA 3: Provider inheritance in nested modules ──────────────
# Child modules inherit the DEFAULT provider (no alias)
# If you need an aliased provider in a child module,
# you MUST explicitly pass it via providers = {}

# Root uses aws.us_east for module but module doesn't know that
module "app" {
  source = "./modules/app"
  # Module silently uses DEFAULT aws provider, not aws.us_east!
}

# ✅ Explicit:
module "app" {
  source = "./modules/app"
  providers = {
    aws = aws.us_east   # Now module uses this provider
  }
}


# ── GOTCHA 4: Circular module dependencies ────────────────────────
# module A outputs something module B needs
# module B outputs something module A needs
# → Terraform detects this and errors
# Solution: Extract shared resource to a third module


# ── GOTCHA 5: Module source changes require terraform init ────────
# If you change source path/version, ALWAYS run terraform init first
# Otherwise plan uses old cached module code


# ── GOTCHA 6: -target with modules ───────────────────────────────
# -target requires full resource address including module path
terraform apply -target=module.networking.aws_vpc.main
terraform apply -target=module.web_app    # Targets entire module
```

---

### Production Best Practices

```hcl
# ✅ 1. ALWAYS document modules with terraform-docs compatible comments
# Run: terraform-docs markdown . > README.md

# ✅ 2. ALWAYS include examples/ directory in module
# Shows how to use the module correctly
# Examples are also used for testing

# ✅ 3. ALWAYS version-pin external modules
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"   # Exact pin for production
}

# ✅ 4. Use 'this' as resource name when module creates ONE of something
resource "aws_lb" "this" { }    # Inside a load-balancer module

# ✅ 5. Expose ENOUGH outputs (IDs, ARNs, names at minimum)
# Callers shouldn't need to recreate values you already have

# ✅ 6. Use optional() for truly optional object attributes (TF 1.3+)
variable "scaling_config" {
  type = object({
    min_size     = number
    max_size     = number
    desired_size = optional(number)    # Optional — defaults to null
    target_cpu   = optional(number, 70) # Optional with default
  })
}

# ✅ 7. Use preconditions for cross-variable validation
resource "aws_lb_listener" "https" {
  count = var.enable_https ? 1 : 0

  lifecycle {
    precondition {
      condition     = var.certificate_arn != null
      error_message = "certificate_arn must be provided when enable_https = true"
    }
  }
}

# ✅ 8. Never use 'terraform.workspace' inside a module
# Modules should be workspace-agnostic — pass environment as a variable

# ✅ 9. Pin Terraform and provider versions in modules
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"   # Lower bound for modules (caller can choose exact)
    }
  }
}
```

---

### Debugging Modules

```bash
# ── List all module resources in state ────────────────────────────
terraform state list | grep "^module\."

# ── Show specific module resource ─────────────────────────────────
terraform state show 'module.web_app.aws_lb.this'
terraform state show 'module.web_app["prod"].aws_lb.this'  # for_each

# ── Target plan/apply to single module ───────────────────────────
terraform plan -target=module.networking
terraform apply -target=module.web_app

# ── Force module re-download ──────────────────────────────────────
rm -rf .terraform/modules
terraform init

# ── Upgrade module version ───────────────────────────────────────
terraform init -upgrade

# ── View module contents after download ───────────────────────────
cat .terraform/modules/modules.json

# ── Debug module variable passing ────────────────────────────────
# Use terraform console to inspect:
terraform console
> var.environment
> module.web_app   # Shows available outputs (after apply)

# ── Graph module dependencies ─────────────────────────────────────
terraform graph | dot -Tsvg > graph.svg   # Requires graphviz
```

---

## 📁 CODE WRITING GUIDANCE

### Professional Module Project Structure

```
my-infrastructure/
│
├── environments/
│   ├── dev/
│   │   ├── backend.tf
│   │   ├── main.tf         ← Calls modules with dev config
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
│
├── modules/
│   ├── networking/          ← VPC, subnets, NAT, IGW
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── locals.tf
│   │   └── README.md
│   │
│   ├── web-app/             ← ALB, ASG, Launch Template
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── locals.tf
│   │   ├── data.tf
│   │   ├── templates/
│   │   │   └── user_data.sh.tpl
│   │   └── README.md
│   │
│   ├── database/            ← RDS, Parameter Group, Subnet Group
│   │   └── ...
│   │
│   └── monitoring/          ← CloudWatch, Alarms, Dashboards
│       └── ...
│
└── .github/
    └── workflows/
        └── terraform.yml    ← CI/CD pipeline
```

---

### Naming Conventions for Modules

```hcl
# ── MODULE LABEL (how you call it) ───────────────────────────────
module "networking" { }      # ✅ Functional purpose
module "web_tier" { }        # ✅ Architectural layer
module "myapp_vpc" { }       # ✅ App + component

module "aws_vpc_module" { }  # ❌ Don't include provider name
module "m1" { }              # ❌ Too abbreviated
module "the_networking_module_for_production" { }  # ❌ Too verbose

# ── RESOURCE NAMES INSIDE MODULES ────────────────────────────────
# When a module creates ONE primary resource of a type:
resource "aws_lb" "this" { }             # ✅ Use "this"
resource "aws_security_group" "this" { } # ✅ Use "this"

# When a module creates MULTIPLE of same type:
resource "aws_security_group" "alb" { }  # ✅ Descriptive role
resource "aws_security_group" "app" { }  # ✅ Descriptive role

# ── VARIABLE NAMING INSIDE MODULES ───────────────────────────────
variable "name" { }           # ✅ Simple, used as prefix
variable "environment" { }    # ✅ Standard
variable "vpc_id" { }         # ✅ What it IS (not where from)
variable "subnet_ids" { }     # ✅ Plural for list

variable "the_vpc_id" { }     # ❌ Unnecessary article
variable "input_vpc_id" { }   # ❌ "input_" prefix redundant
```

---

## 🧪 HANDS-ON LAB

### Exercise: Build a Complete Reusable VPC Module

**Goal:** Create a `networking` module that:
1. Creates a VPC with configurable CIDR
2. Creates public and private subnets across N availability zones (dynamic, using data source)
3. Creates Internet Gateway for public subnets
4. Creates NAT Gateway (single or per-AZ based on environment)
5. Creates appropriate route tables and associations
6. Accepts: `name`, `environment`, `vpc_cidr`, `az_count`
7. Outputs: `vpc_id`, `public_subnet_ids`, `private_subnet_ids`, `vpc_cidr_block`

**Then call the module twice:**
- Once for `dev` (single NAT, 2 AZs)
- Once for `prod` (NAT per AZ, 3 AZs)

**Expected outputs:**
```bash
dev_vpc_id   = "vpc-0dev..."
prod_vpc_id  = "vpc-0prod..."
dev_public_subnets  = ["subnet-0a...", "subnet-0b..."]
prod_public_subnets = ["subnet-0x...", "subnet-0y...", "subnet-0z..."]
```

**Try it yourself first!**

---

### ✅ Solution

```hcl
# modules/networking/variables.tf
variable "name" {
  type        = string
  description = "Name prefix for all resources"
}

variable "environment" {
  type        = string
  description = "Environment name"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid CIDR block."
  }
}

variable "az_count" {
  type        = number
  description = "Number of availability zones to use"
  default     = 2
  validation {
    condition     = var.az_count >= 1 && var.az_count <= 4
    error_message = "az_count must be between 1 and 4."
  }
}

variable "tags" {
  type        = map(string)
  description = "Additional tags"
  default     = {}
}
```

```hcl
# modules/networking/locals.tf
locals {
  prefix         = "${var.name}-${var.environment}"
  single_nat     = var.environment != "prod"   # HA NAT in prod only

  public_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  private_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]

  common_tags = merge(
    {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Module      = "networking"
    },
    var.tags
  )
}
```

```hcl
# modules/networking/data.tf
data "aws_availability_zones" "available" {
  state = "available"
}
```

```hcl
# modules/networking/main.tf

# ── VPC ───────────────────────────────────────────────────────────
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${local.prefix}-vpc" })
}

# ── SUBNETS ───────────────────────────────────────────────────────
resource "aws_subnet" "public" {
  count = var.az_count

  vpc_id                  = aws_vpc.this.id
  cidr_block              = local.public_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-public-${count.index + 1}"
    Tier = "Public"
  })
}

resource "aws_subnet" "private" {
  count = var.az_count

  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-private-${count.index + 1}"
    Tier = "Private"
  })
}

# ── INTERNET GATEWAY ──────────────────────────────────────────────
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(local.common_tags, { Name = "${local.prefix}-igw" })
}

# ── ELASTIC IPs FOR NAT ───────────────────────────────────────────
resource "aws_eip" "nat" {
  # Single NAT in dev, one per AZ in prod
  count  = local.single_nat ? 1 : var.az_count
  domain = "vpc"
  tags   = merge(local.common_tags, { Name = "${local.prefix}-eip-${count.index + 1}" })

  depends_on = [aws_internet_gateway.this]
}

# ── NAT GATEWAY ───────────────────────────────────────────────────
resource "aws_nat_gateway" "this" {
  count = local.single_nat ? 1 : var.az_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(local.common_tags, { Name = "${local.prefix}-nat-${count.index + 1}" })

  depends_on = [aws_internet_gateway.this]
}

# ── ROUTE TABLES ──────────────────────────────────────────────────
# Public route table (shared)
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(local.common_tags, { Name = "${local.prefix}-public-rt" })
}

resource "aws_route_table_association" "public" {
  count          = var.az_count
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route tables (one per AZ in prod, one shared in dev)
resource "aws_route_table" "private" {
  count  = local.single_nat ? 1 : var.az_count
  vpc_id = aws_vpc.this.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.this[count.index].id
  }

  tags = merge(local.common_tags, { Name = "${local.prefix}-private-rt-${count.index + 1}" })
}

resource "aws_route_table_association" "private" {
  count          = var.az_count
  subnet_id      = aws_subnet.private[count.index].id
  # In dev: all point to same RT (index 0). In prod: each has own RT
  route_table_id = aws_route_table.private[local.single_nat ? 0 : count.index].id
}
```

```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ips" {
  description = "Public IPs of NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.this.id
}
```

```hcl
# root/main.tf — Using the module

provider "aws" { region = "us-east-1" }

module "networking_dev" {
  source = "./modules/networking"

  name        = "myapp"
  environment = "dev"
  vpc_cidr    = "10.0.0.0/16"
  az_count    = 2
}

module "networking_prod" {
  source = "./modules/networking"

  name        = "myapp"
  environment = "prod"
  vpc_cidr    = "10.1.0.0/16"
  az_count    = 3
}

output "dev_vpc_id"          { value = module.networking_dev.vpc_id }
output "dev_public_subnets"  { value = module.networking_dev.public_subnet_ids }
output "prod_vpc_id"         { value = module.networking_prod.vpc_id }
output "prod_public_subnets" { value = module.networking_prod.public_subnet_ids }
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| Module | Reusable container of resources — write once, deploy many |
| Root module | Your working directory — where you run terraform commands |
| Child module | Called via `module` block — has own variables, resources, outputs |
| Module source | Local path, registry, Git, S3 — always pin external versions |
| Module outputs | Only way to expose values — modules are black boxes |
| `providers =` | Explicitly pass aliased providers to child modules |
| `for_each` on module | Deploy same module multiple times from a map |
| `moved` block | Rename module resources without destroy/recreate |
| `terraform init` | Required after adding/changing any module source |
| `this` naming | Use as resource name when module creates one of that type |

---

### Quick Syntax Reference

```hcl
# Call a module
module "LABEL" {
  source  = "./path" | "registry/module/provider" | "git::url"
  version = "~> 5.0"       # Registry and Git only

  # Input variables
  var_name = value

  # Pass specific providers
  providers = { aws = aws.alias }

  # Meta-arguments
  count      = number
  for_each   = map_or_set
  depends_on = [resource.name]
}

# Use module outputs
module.LABEL.output_name
module.LABEL["key"].output_name   # for_each module

# Module path variables (inside module)
path.module   # Directory of this module's .tf files
path.root     # Root module directory
path.cwd      # Current working directory

# Require providers inside module
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"   # Use >= for modules, exact pin in root
    }
  }
}

# Optional variable attributes (TF 1.3+)
variable "config" {
  type = object({
    required_field = string
    optional_field = optional(string)
    with_default   = optional(number, 42)
  })
}

# Moved block (refactoring without destroy)
moved {
  from = aws_instance.old
  to   = module.compute.aws_instance.web
}
```

---

### Top 10 Interview Questions

1. **What is the difference between a root and child module?** Root module is your working directory where you run Terraform. Child modules are called via `module {}` blocks and can be local, registry, or Git-sourced.

2. **Why should provider blocks never go in child modules?** It breaks reusability — the same module can't be used in different regions/accounts if it hardcodes provider config. Providers belong exclusively in root modules, passed down via `providers = {}` if needed.

3. **How do you pass an aliased provider to a module?** Via the `providers` meta-argument: `providers = { aws = aws.us_east }`. The module must also declare the provider requirement in its `terraform {}` block.

4. **What does `path.module` refer to inside a module?** The filesystem path to the directory containing the module's `.tf` files. Used to reference templates or scripts relative to the module: `"${path.module}/templates/script.sh"`.

5. **How do you deploy the same module multiple times?** Use `for_each` on the module block with a map. Each key creates an independent instance with its own state. Access outputs via `module.name["key"].output`.

6. **How do you handle a module rename/refactor without destroying resources?** Use a `moved {}` block declaring `from` and `to` addresses. Terraform moves the state entries instead of destroying and recreating.

7. **What's the difference between `version = "~> 5.1"` and `version = "~> 5.0"`?** `~> 5.1` allows 5.1.x only. `~> 5.0` allows any 5.x.x. Use `~> 5.1` for tighter control in production.

8. **How do modules communicate with each other?** Only through root module orchestration — Module A outputs a value, root passes it as an input to Module B. Modules cannot directly reference each other.

9. **What is the purpose of an `examples/` directory in a module?** Shows working usage of the module, serves as integration tests (with Terratest), and helps users understand inputs/outputs without reading all the code.

10. **What is `optional()` in object variable types?** A function (TF 1.3+) that marks object attributes as optional with an optional default value — `optional(string)` or `optional(number, 42)`. Callers don't have to provide those keys.

---
