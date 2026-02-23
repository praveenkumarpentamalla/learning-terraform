# 📦 TOPIC 3: Variables, Locals & Outputs — Deep Dive

---

## 🟢 BEGINNER LEVEL

### What Are Variables, Locals & Outputs? (Simple Terms)

Think of building a **pizza ordering system**:

```
Variables  = The ORDER FORM  → Customer fills in: size, toppings, crust
Locals     = KITCHEN NOTES   → Chef calculates: total price, prep time
Outputs    = RECEIPT         → Customer gets back: order #, pickup time
```

In Terraform:
- **Variables** = Inputs you pass INTO your config (makes it reusable)
- **Locals** = Internal computed values (used INSIDE your config only)
- **Outputs** = Values you expose OUT of your config (for humans or other modules)

---

### Real-World Analogy

```
┌─────────────────────────────────────────────────────────────────┐
│                    TERRAFORM CONFIG                             │
│                                                                 │
│  INPUT                                                          │
│  ┌──────────────┐                                               │
│  │  Variables   │──────────────────────────┐                   │
│  │  (Order Form)│                          │                   │
│  └──────────────┘                          ▼                   │
│                                   ┌─────────────────┐          │
│                                   │  Locals         │          │
│                                   │  (Kitchen Math) │          │
│                                   └────────┬────────┘          │
│                                            │                   │
│                                            ▼                   │
│                                   ┌─────────────────┐          │
│                                   │  Resources      │          │
│                                   │  (The Pizza)    │          │
│                                   └────────┬────────┘          │
│                                            │                   │
│  OUTPUT                                    ▼                   │
│  ┌──────────────┐              ┌─────────────────────┐         │
│  │  Outputs     │◄─────────────│  Result Values      │         │
│  │  (Receipt)   │              └─────────────────────┘         │
│  └──────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

### Basic Syntax

```hcl
# ── VARIABLE ──────────────────────────────────────────────────────
variable "environment" {
  type    = string
  default = "dev"
}

# Use it:
# var.environment


# ── LOCAL ─────────────────────────────────────────────────────────
locals {
  app_name = "myapp-${var.environment}"
}

# Use it:
# local.app_name


# ── OUTPUT ────────────────────────────────────────────────────────
output "bucket_name" {
  value = aws_s3_bucket.main.id
}

# See it after apply:
# terraform output bucket_name
```

---

### Minimal Working Example

```hcl
# variables.tf
variable "project_name" {
  type    = string
  default = "my-app"
}

variable "environment" {
  type    = string
  default = "dev"
}

# locals.tf
locals {
  bucket_name = "${var.project_name}-${var.environment}-assets"
}

# main.tf
resource "aws_s3_bucket" "assets" {
  bucket = local.bucket_name

  tags = {
    Name        = local.bucket_name
    Environment = var.environment
  }
}

# outputs.tf
output "bucket_name" {
  description = "Name of the assets bucket"
  value       = aws_s3_bucket.assets.id
}

output "bucket_arn" {
  description = "ARN of the assets bucket"
  value       = aws_s3_bucket.assets.arn
}
```

```bash
# Run with defaults
terraform apply

# Override a variable
terraform apply -var="environment=prod"

# See outputs
terraform output
terraform output bucket_name
```

---

### Ways to Pass Variables (Priority Order)

```
HIGHEST PRIORITY
      │
      │  1. Command line:  -var="key=value"
      │  2. .tfvars file:  -var-file="prod.tfvars"
      │  3. Auto-loaded:   terraform.tfvars  OR  *.auto.tfvars
      │  4. Env variable:  TF_VAR_key=value
      │  5. Default value: default = "..."
      │
LOWEST PRIORITY
```

```bash
# 1. Command line flag
terraform apply -var="environment=prod"

# 2. Specific var file
terraform apply -var-file="prod.tfvars"

# 3. Auto-loaded (Terraform finds this automatically)
# File: terraform.tfvars or production.auto.tfvars

# 4. Environment variable
export TF_VAR_environment="prod"
terraform apply

# 5. Default in variable block (fallback)
variable "environment" {
  default = "dev"   # Used if nothing else provides a value
}
```

---

### Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Using var inside locals at wrong scope
locals {
  name = var.project  # ✅ This is fine — var is accessible in locals
}

resource "aws_s3_bucket" "b" {
  locals { }   # ❌ locals block can't go INSIDE resource blocks
}

# ❌ MISTAKE 2: Referencing outputs like variables
output "bucket_id" {
  value = aws_s3_bucket.main.id
}

resource "aws_instance" "web" {
  # ❌ Can't use output.bucket_id here — outputs are for external consumption
  tags = { BucketId = output.bucket_id }  # Error!
  # ✅ Use the resource directly
  tags = { BucketId = aws_s3_bucket.main.id }
}

# ❌ MISTAKE 3: Forgetting that outputs are PUBLIC (sensitive data leak)
output "db_password" {
  value = aws_db_instance.main.password   # Shows in CLI, logs, state!
}
# ✅ Fix:
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true   # Hides from CLI output
}

# ❌ MISTAKE 4: Circular local references
locals {
  a = local.b   # ❌ Circular!
  b = local.a
}

# ❌ MISTAKE 5: Wrong type in variable
variable "count" {
  type    = string
  default = "3"   # It's a string "3", not number 3!
}
resource "aws_instance" "web" {
  count = var.count   # Type mismatch error
}
# ✅ Fix:
variable "count" {
  type    = number
  default = 3
}
```

---

## 🟡 INTERMEDIATE LEVEL

### Variable Types — Complete Reference

```hcl
# ── PRIMITIVE TYPES ───────────────────────────────────────────────

variable "app_name" {
  type        = string
  description = "Application name"
  default     = "myapp"
}

variable "instance_count" {
  type        = number
  description = "Number of instances"
  default     = 2
}

variable "enable_https" {
  type        = bool
  description = "Enable HTTPS"
  default     = true
}


# ── COLLECTION TYPES ──────────────────────────────────────────────

variable "allowed_ips" {
  type        = list(string)
  description = "List of allowed IP CIDR ranges"
  default     = ["10.0.0.0/8", "172.16.0.0/12"]
}

variable "instance_tags" {
  type        = map(string)
  description = "Tags to apply to instances"
  default = {
    Team    = "DevOps"
    Project = "MyApp"
  }
}

variable "allowed_ports" {
  type        = set(number)    # set = list with NO duplicates, NO order
  description = "Set of allowed ports"
  default     = [80, 443, 8080]
}


# ── STRUCTURAL TYPES ──────────────────────────────────────────────

# object — fixed structure, each attribute has its own type
variable "database_config" {
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    engine_version    = string
  })
  default = {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
    engine_version    = "14.9"
  }
}

# tuple — fixed-length list where each position has a type
variable "server_config" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 2, true]
  # Position 0 = instance_type (string)
  # Position 1 = count (number)
  # Position 2 = monitoring (bool)
}

# list of objects — most common for complex resources
variable "subnets" {
  type = list(object({
    cidr_block        = string
    availability_zone = string
    is_public         = bool
  }))
  default = [
    { cidr_block = "10.0.1.0/24", availability_zone = "us-east-1a", is_public = true  },
    { cidr_block = "10.0.2.0/24", availability_zone = "us-east-1b", is_public = true  },
    { cidr_block = "10.0.3.0/24", availability_zone = "us-east-1c", is_public = false },
  ]
}

# map of objects — for keyed complex configs
variable "environments" {
  type = map(object({
    instance_type  = string
    instance_count = number
    multi_az       = bool
  }))
  default = {
    dev = {
      instance_type  = "t2.micro"
      instance_count = 1
      multi_az       = false
    }
    prod = {
      instance_type  = "t2.large"
      instance_count = 3
      multi_az       = true
    }
  }
}

# any — escape hatch (use sparingly)
variable "extra_config" {
  type    = any
  default = {}
}
```

---

### Variable Validation — Production-Grade

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}


variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "VPC CIDR must be a valid CIDR notation (e.g., 10.0.0.0/16)."
  }
}


variable "instance_type" {
  type        = string
  description = "EC2 instance type"

  validation {
    condition     = can(regex("^(t2|t3|t3a|m5|c5)\\.", var.instance_type))
    error_message = "Instance type must be from t2, t3, t3a, m5, or c5 families."
  }
}


variable "port" {
  type        = number
  description = "Application port"

  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}


variable "bucket_name" {
  type        = string
  description = "S3 bucket name"

  # Multiple validations
  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be between 3 and 63 characters."
  }

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9\\-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must start/end with lowercase letter or number, contain only lowercase letters, numbers, and hyphens."
  }

  validation {
    condition     = !can(regex("\\.\\.+", var.bucket_name))
    error_message = "Bucket name cannot contain consecutive dots."
  }
}


variable "tags" {
  type        = map(string)
  description = "Resource tags"

  validation {
    condition     = contains(keys(var.tags), "Owner")
    error_message = "Tags must include an 'Owner' key."
  }

  validation {
    condition     = length(var.tags) <= 50
    error_message = "AWS allows maximum 50 tags per resource."
  }
}
```

---

### Sensitive Variables — Handling Secrets

```hcl
# Marking a variable as sensitive
variable "db_password" {
  type        = string
  description = "Database master password"
  sensitive   = true   # Value never shown in CLI output or logs
}

variable "api_key" {
  type      = string
  sensitive = true
}

# Sensitive variables propagate sensitivity
locals {
  connection_string = "postgres://admin:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  # ^ This local is now ALSO sensitive because it contains var.db_password
}

# Outputs with sensitive values
output "db_endpoint" {
  value       = aws_db_instance.main.endpoint
  sensitive   = false   # Not sensitive — just a hostname
}

output "connection_string" {
  value     = local.connection_string
  sensitive = true   # Must mark sensitive because local is sensitive
}
```

```bash
# Passing sensitive vars — NEVER in command line (shows in shell history!)
# ❌ terraform apply -var="db_password=mysecret"

# ✅ Method 1: Environment variable
export TF_VAR_db_password="mysecret"
terraform apply

# ✅ Method 2: Sensitive .tfvars file (gitignored)
# secrets.tfvars:
# db_password = "mysecret"
terraform apply -var-file="secrets.tfvars"

# ✅ Method 3: AWS Secrets Manager + data source (BEST for prod)
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/myapp/db-credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)
}

resource "aws_db_instance" "main" {
  password = local.db_creds["password"]
}
```

---

### Locals — Advanced Usage

```hcl
# locals.tf

locals {
  # ── SIMPLE COMPUTATION ───────────────────────────────────────────
  name_prefix    = "${var.project}-${var.environment}"
  is_production  = var.environment == "prod"
  is_development = var.environment == "dev"


  # ── CONDITIONAL VALUES ───────────────────────────────────────────
  instance_type = local.is_production ? "t2.large" : "t2.micro"
  instance_count = local.is_production ? 3 : 1
  multi_az      = local.is_production ? true : false

  # Conditional with complex default
  monitoring_config = local.is_production ? {
    interval            = 60
    enabled             = true
    retention_in_days   = 90
  } : {
    interval            = 300
    enabled             = false
    retention_in_days   = 7
  }


  # ── TAG MANAGEMENT (most common use case) ────────────────────────
  base_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.owner
    CostCenter  = var.cost_center
  }

  # Merge base tags with resource-specific tags
  web_tags = merge(local.base_tags, {
    Role       = "WebServer"
    Tier       = "Frontend"
  })

  db_tags = merge(local.base_tags, {
    Role       = "Database"
    Tier       = "Data"
    Backup     = "Required"
  })


  # ── STRING MANIPULATION ──────────────────────────────────────────
  # Convert to uppercase
  env_upper = upper(var.environment)    # "PROD"

  # Replace dashes with underscores (for some naming requirements)
  safe_name = replace(local.name_prefix, "-", "_")

  # Format with padding
  padded_index = format("%03d", 1)       # "001"


  # ── LIST/MAP TRANSFORMATIONS ─────────────────────────────────────
  # Convert list to set (remove duplicates)
  unique_ports = toset(var.allowed_ports)

  # Filter list
  private_subnets = [
    for s in var.subnets : s
    if !s.is_public
  ]

  public_subnets = [
    for s in var.subnets : s
    if s.is_public
  ]

  # Transform list into map (for for_each)
  subnet_map = {
    for i, s in var.subnets :
    "subnet-${i}" => s
  }

  # Extract specific attribute from list of objects
  subnet_cidrs = [for s in var.subnets : s.cidr_block]


  # ── COMPLEX EXPRESSION ───────────────────────────────────────────
  # Availability zones (up to max_az, or however many are available)
  az_count = min(var.max_availability_zones,
                 length(data.aws_availability_zones.available.names))

  # CIDR calculations
  public_subnet_cidrs = [
    for i in range(local.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]

  private_subnet_cidrs = [
    for i in range(local.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]


  # ── ENVIRONMENT-SPECIFIC CONFIG ──────────────────────────────────
  # Single source of truth for environment configs
  env_config = {
    dev = {
      instance_type   = "t2.micro"
      min_size        = 1
      max_size        = 2
      retention_days  = 7
      log_level       = "DEBUG"
    }
    staging = {
      instance_type   = "t2.small"
      min_size        = 1
      max_size        = 3
      retention_days  = 14
      log_level       = "INFO"
    }
    prod = {
      instance_type   = "t2.large"
      min_size        = 3
      max_size        = 10
      retention_days  = 90
      log_level       = "WARN"
    }
  }

  # Current environment config
  current_env = local.env_config[var.environment]
}

# Using the computed config
resource "aws_autoscaling_group" "web" {
  min_size          = local.current_env.min_size
  max_size          = local.current_env.max_size
  # ...
}
```

---

### Outputs — Complete Reference

```hcl
# outputs.tf

# ── SIMPLE OUTPUTS ────────────────────────────────────────────────
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Public IP address of the web server"
  value       = aws_instance.web.public_ip
}


# ── SENSITIVE OUTPUT ──────────────────────────────────────────────
output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive   = true   # Hidden from CLI, still in state file
}


# ── COMPLEX OUTPUTS ───────────────────────────────────────────────
# Output a map
output "subnet_details" {
  description = "Details of all subnets"
  value = {
    for subnet in aws_subnet.public :
    subnet.availability_zone => {
      id         = subnet.id
      cidr_block = subnet.cidr_block
    }
  }
}

# Output a list
output "instance_ids" {
  description = "IDs of all web instances"
  value       = aws_instance.web[*].id   # Splat expression
}

# Output a computed string
output "load_balancer_url" {
  description = "URL to access the application"
  value       = "https://${aws_lb.main.dns_name}"
}


# ── OUTPUT WITH PRECONDITION (TF 1.2+) ───────────────────────────
output "api_endpoint" {
  description = "API Gateway endpoint URL"
  value       = aws_api_gateway_stage.main.invoke_url

  precondition {
    condition     = aws_api_gateway_stage.main.invoke_url != ""
    error_message = "API Gateway endpoint URL is empty — deployment may have failed."
  }
}


# ── MODULE OUTPUTS (passing to parent) ────────────────────────────
# Inside a module, outputs are how you expose values to the caller

# module/networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

# Caller uses:
# module.networking.vpc_id
# module.networking.public_subnet_ids
```

---

### Best Practices — Variables, Locals, Outputs

```hcl
# ✅ BEST PRACTICE 1: Always add description to every variable and output
variable "instance_type" {
  type        = string
  description = "EC2 instance type (e.g., t2.micro, t2.large)"  # ✅
  default     = "t2.micro"
}

# ✅ BEST PRACTICE 2: Use locals for any value used more than once
# ❌ Repetition
resource "aws_s3_bucket" "data"  { bucket = "${var.project}-${var.env}-data"  }
resource "aws_s3_bucket" "logs"  { bucket = "${var.project}-${var.env}-logs"  }
resource "aws_s3_bucket" "state" { bucket = "${var.project}-${var.env}-state" }

# ✅ Use locals
locals {
  prefix = "${var.project}-${var.env}"
}
resource "aws_s3_bucket" "data"  { bucket = "${local.prefix}-data"  }
resource "aws_s3_bucket" "logs"  { bucket = "${local.prefix}-logs"  }
resource "aws_s3_bucket" "state" { bucket = "${local.prefix}-state" }

# ✅ BEST PRACTICE 3: Separate variables by concern with comments
# variables.tf
# ── PROJECT ──────────────────────────────────────────────────────
variable "project_name"  { ... }
variable "environment"   { ... }
variable "aws_region"    { ... }

# ── NETWORKING ────────────────────────────────────────────────────
variable "vpc_cidr"        { ... }
variable "az_count"        { ... }

# ── COMPUTE ───────────────────────────────────────────────────────
variable "instance_type"   { ... }
variable "instance_count"  { ... }

# ── DATABASE ──────────────────────────────────────────────────────
variable "db_instance_class" { ... }
variable "db_password"       { ... sensitive = true }

# ✅ BEST PRACTICE 4: All outputs should have description
# ✅ BEST PRACTICE 5: Group related outputs together with comments
# ✅ BEST PRACTICE 6: Use locals for env-specific config (single source of truth)
# ✅ BEST PRACTICE 7: Don't use locals for things that should be variables
#    (locals can't be overridden by users, variables can)
```

---

### Interview-Focused Points 🎯

1. **What's the difference between `variable` and `local`?** Variables are inputs — users can override them. Locals are internal computations — only the module itself can set them. Use variables for values that change per deployment, locals for derived/computed values.

2. **How do you handle sensitive variables?** Mark with `sensitive = true`, pass via `TF_VAR_` env vars or `-var-file` with gitignored files, use Secrets Manager for production.

3. **Can locals reference other locals?** Yes — but not circularly. Terraform resolves them in dependency order automatically.

4. **What's the output precedence order for variables?** CLI `-var` > `-var-file` > `terraform.tfvars` / `*.auto.tfvars` > `TF_VAR_` env vars > default value.

5. **Can you use outputs from one module in another module?** Yes — `module.module_name.output_name` — this is the primary way modules communicate.

6. **What is `sensitive = true` on an output?** It hides the value from CLI display and plan output, but the value still exists in plaintext in the state file. It's not true encryption.

7. **What happens if a required variable has no value?** Terraform prompts interactively at runtime. In CI/CD (non-interactive), it fails with an error.

---

## 🔴 ADVANCED LEVEL

### How Terraform Resolves Variables Internally

```
terraform apply -var="env=prod"
        │
        ▼
1. PARSE all .tf files
   └── Collect all `variable` blocks
   └── Note: type, default, sensitive, validation

2. BUILD value map (priority order):
   └── Start with defaults
   └── Override with TF_VAR_ env vars
   └── Override with terraform.tfvars (auto-loaded)
   └── Override with *.auto.tfvars (auto-loaded, alphabetical)
   └── Override with -var-file flags (in order specified)
   └── Override with -var flags (in order specified)

3. TYPE CONVERSION:
   └── Attempt to convert to declared type
   └── "3" → 3 if type = number
   └── Fail if conversion not possible

4. VALIDATION:
   └── Run all validation blocks
   └── Fail if any condition is false

5. SENSITIVITY PROPAGATION:
   └── Any value derived from sensitive variable
       is automatically marked sensitive

6. VALUES AVAILABLE as var.NAME throughout config
```

---

### Advanced Expression Patterns

```hcl
# ── FOR EXPRESSIONS ──────────────────────────────────────────────

# List transformation — add suffix to all items
locals {
  bucket_names = [for env in ["dev", "staging", "prod"] : "myapp-${env}-bucket"]
  # Result: ["myapp-dev-bucket", "myapp-staging-bucket", "myapp-prod-bucket"]
}

# List filtering — only get prod-like environments
locals {
  prod_envs = [for env in var.environments : env if env.is_production]
}

# Map transformation — transform all values
variable "instance_counts" {
  type    = map(number)
  default = { web = 2, api = 3, worker = 1 }
}

locals {
  # Double all instance counts for prod
  scaled_counts = {
    for tier, count in var.instance_counts :
    tier => local.is_production ? count * 2 : count
  }
  # Result: { web = 4, api = 6, worker = 2 }

  # Filter map to only included tiers
  included_tiers = {
    for tier, count in var.instance_counts :
    tier => count
    if count > 1  # Only tiers with more than 1 instance
  }
  # Result: { web = 2, api = 3 }
}

# Flatten — convert nested lists to flat list
variable "sg_rules" {
  default = {
    web = [80, 443]
    api = [8080, 8443]
  }
}

locals {
  # Create flat list of {service, port} objects
  all_rules = flatten([
    for service, ports in var.sg_rules : [
      for port in ports : {
        service = service
        port    = port
      }
    ]
  ])
  # Result: [
  #   { service = "web", port = 80  },
  #   { service = "web", port = 443 },
  #   { service = "api", port = 8080},
  #   { service = "api", port = 8443},
  # ]
}

# Use flattened result with for_each
resource "aws_security_group_rule" "ingress" {
  for_each = {
    for rule in local.all_rules :
    "${rule.service}-${rule.port}" => rule
  }

  type        = "ingress"
  from_port   = each.value.port
  to_port     = each.value.port
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  security_group_id = aws_security_group.main.id
}
```

---

### Built-in Functions — Production Toolkit

```hcl
locals {
  # ── STRING FUNCTIONS ─────────────────────────────────────────────
  upper_env   = upper(var.environment)           # "DEV"
  lower_env   = lower(var.environment)           # "dev"
  trimmed     = trimspace("  hello  ")           # "hello"
  replaced    = replace("hello-world", "-", "_") # "hello_world"
  starts_with = startswith(var.name, "prod-")    # true/false
  ends_with   = endswith(var.name, "-v2")        # true/false

  # Format (printf-style)
  padded_num  = format("%04d", 7)                # "0007"
  formatted   = format("%s-%s", var.project, var.env)

  # Join/Split
  csv_string  = join(", ", ["a", "b", "c"])      # "a, b, c"
  parts       = split("-", "prod-us-east-1")      # ["prod", "us", "east", "1"]

  # String check
  contains_str = strcontains("production", "prod")  # true

  # ── NUMBER FUNCTIONS ─────────────────────────────────────────────
  ceiling  = ceil(2.3)     # 3
  flooring = floor(2.9)    # 2
  min_val  = min(3, 1, 4)  # 1
  max_val  = max(3, 1, 4)  # 4
  abs_val  = abs(-5)       # 5

  # ── COLLECTION FUNCTIONS ─────────────────────────────────────────
  list_length  = length(var.allowed_ips)             # Count items
  map_keys     = keys(var.instance_tags)             # ["Key1", "Key2"]
  map_values   = values(var.instance_tags)           # ["Val1", "Val2"]
  has_key      = contains(keys(var.tags), "Owner")   # true/false
  list_sorted  = sort(["c", "a", "b"])               # ["a", "b", "c"]
  list_reverse = reverse([1, 2, 3])                  # [3, 2, 1]
  first_item   = element(var.allowed_ips, 0)         # First element
  list_unique  = distinct(["a", "b", "a", "c"])      # ["a", "b", "c"]
  flattened    = flatten([[1, 2], [3, 4]])            # [1, 2, 3, 4]

  # Merge maps (rightmost wins on conflict)
  merged = merge(
    { a = 1, b = 2 },
    { b = 3, c = 4 }
  )
  # Result: { a = 1, b = 3, c = 4 }

  # Convert between types
  as_set   = toset(["a", "b", "a"])    # {"a", "b"} (deduplicated)
  as_list  = tolist(toset(["a", "b"])) # ["a", "b"]
  as_map   = tomap({ key = "value" })

  # Zipmap — create map from two lists
  env_map = zipmap(
    ["dev", "staging", "prod"],
    ["t2.micro", "t2.small", "t2.large"]
  )
  # Result: { dev = "t2.micro", staging = "t2.small", prod = "t2.large" }


  # ── ENCODING FUNCTIONS ───────────────────────────────────────────
  json_str  = jsonencode({ key = "value", count = 3 })
  json_obj  = jsondecode("{\"key\":\"value\"}")
  base64    = base64encode("hello world")
  decoded   = base64decode("aGVsbG8gd29ybGQ=")
  yaml_str  = yamlencode({ key = "value" })


  # ── IP/NETWORK FUNCTIONS ─────────────────────────────────────────
  # Calculate subnet CIDRs from a base CIDR
  subnet_1  = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
  subnet_2  = cidrsubnet("10.0.0.0/16", 8, 2)  # "10.0.2.0/24"
  host_ip   = cidrhost("10.0.1.0/24", 5)        # "10.0.1.5"
  netmask   = cidrnetmask("10.0.0.0/16")         # "255.255.0.0"
  prefix    = cidrnetmask("10.0.0.0/24")         # "255.255.255.0"


  # ── TYPE/EXISTENCE FUNCTIONS ─────────────────────────────────────
  # can() — returns true if expression doesn't error
  is_valid_cidr = can(cidrnetmask(var.vpc_cidr))

  # try() — return first non-error value
  safe_lookup = try(var.config["optional_key"], "default_value")

  # coalesce — first non-null, non-empty value
  chosen_name = coalesce(var.custom_name, local.generated_name, "fallback")

  # one() — from a list of 0 or 1 items, extract the value or null
  maybe_alarm_arn = one(aws_cloudwatch_metric_alarm.cpu[*].arn)


  # ── FILESYSTEM FUNCTIONS (for loading external files) ────────────
  user_data_script  = file("${path.module}/scripts/user_data.sh")
  policy_json       = file("${path.module}/policies/s3_policy.json")
  policy_obj        = jsondecode(file("${path.module}/policies/policy.json"))

  # filebase64 — encode file content as base64
  cert_data = filebase64("${path.module}/certs/server.crt")

  # templatefile — render a template with variables
  rendered_config = templatefile("${path.module}/templates/app.conf.tpl", {
    db_host    = aws_db_instance.main.endpoint
    app_port   = var.app_port
    log_level  = local.current_env.log_level
  })
}
```

---

### `templatefile` — Advanced Config Generation

```bash
# templates/nginx.conf.tpl
server {
    listen ${port};
    server_name ${server_name};

    location / {
        proxy_pass http://${backend_host}:${backend_port};
        proxy_set_header Host $host;
    }

    %{ for domain in extra_domains ~}
    server_name ${domain};
    %{ endfor ~}

    %{ if enable_ssl }
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/server.crt;
    %{ endif }
}
```

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  user_data = templatefile("${path.module}/templates/nginx.conf.tpl", {
    port          = 80
    server_name   = "api.myapp.com"
    backend_host  = aws_instance.api.private_ip
    backend_port  = 8080
    extra_domains = ["www.myapp.com", "myapp.com"]
    enable_ssl    = var.environment == "prod"
  })
}
```

---

### Output Chaining Between Modules (Production Pattern)

```hcl
# ── ROOT MAIN.TF ──────────────────────────────────────────────────

# Module 1: Networking
module "networking" {
  source      = "./modules/networking"
  environment = var.environment
  vpc_cidr    = var.vpc_cidr
}

# Module 2: Database — uses outputs from networking
module "database" {
  source             = "./modules/database"
  environment        = var.environment
  vpc_id             = module.networking.vpc_id          # ← networking output
  private_subnet_ids = module.networking.private_subnet_ids  # ← networking output
  db_password        = var.db_password
}

# Module 3: Compute — uses outputs from both networking and database
module "compute" {
  source             = "./modules/compute"
  environment        = var.environment
  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.public_subnet_ids
  db_endpoint        = module.database.endpoint    # ← database output
  db_password        = var.db_password
}

# Root-level outputs — expose to operators
output "application_url" {
  value = "https://${module.compute.load_balancer_dns}"
}

output "database_endpoint" {
  value     = module.database.endpoint
  sensitive = false   # Just a hostname, not sensitive
}
```

---

### Edge Cases & Gotchas

```hcl
# ── GOTCHA 1: Sensitive propagation is automatic but must be acknowledged ──
variable "password" {
  sensitive = true
}

locals {
  conn = "user:${var.password}@host"  # Automatically sensitive
}

output "conn" {
  value     = local.conn
  sensitive = true   # MUST declare or Terraform errors
  # Error without this: "Output refers to sensitive value"
}


# ── GOTCHA 2: try() vs can() ──────────────────────────────────────
# can() returns bool — use in conditions
variable "config" {
  type    = map(string)
  default = {}
}

locals {
  # can() — just checks if it works
  has_timeout = can(var.config["timeout"])   # true or false

  # try() — evaluates and returns fallback on error
  timeout = try(var.config["timeout"], "30s")  # Value or fallback
}


# ── GOTCHA 3: Variable with no default and no value = interactive prompt ──
variable "db_password" {
  type = string
  # No default! If no value provided, Terraform asks interactively
  # In CI/CD this causes: "No value for required variable"
}


# ── GOTCHA 4: locals can't reference resources from other modules ──
# Only root module or the same module's resources are accessible in locals
# Use outputs to pass values between modules


# ── GOTCHA 5: Output values are computed AFTER apply ─────────────
# During plan, some outputs show as "(known after apply)"
# This is expected for attributes that AWS assigns (IP, ID, ARN)


# ── GOTCHA 6: list vs set in for_each ────────────────────────────
variable "names" {
  type    = list(string)
  default = ["alice", "bob"]
}

resource "aws_iam_user" "users" {
  # ❌ for_each needs set or map, not list
  # for_each = var.names  # Error!

  # ✅ Convert to set
  for_each = toset(var.names)
  name     = each.key
}
```

---

### Production Best Practices

```hcl
# ✅ Use object type for grouped config (instead of many separate vars)
variable "app_config" {
  type = object({
    name          = string
    version       = string
    port          = number
    health_path   = string
    enable_https  = bool
  })
  description = "Application configuration"
  default = {
    name         = "myapp"
    version      = "1.0.0"
    port         = 8080
    health_path  = "/health"
    enable_https = true
  }
}


# ✅ Use locals for environment lookup table (avoid duplication)
locals {
  env_settings = {
    dev     = { size = "small",  count = 1, multi_az = false }
    staging = { size = "medium", count = 2, multi_az = false }
    prod    = { size = "large",  count = 3, multi_az = true  }
  }
  current = local.env_settings[var.environment]
}


# ✅ Use path variables for portability
locals {
  scripts_dir   = "${path.module}/scripts"
  templates_dir = "${path.module}/templates"
  policies_dir  = "${path.module}/policies"
}
# path.module  = directory of current .tf file
# path.root    = directory where terraform was run
# path.cwd     = current working directory


# ✅ Validate critical variables with clear error messages
variable "aws_region" {
  type = string
  validation {
    condition = contains([
      "us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"
    ], var.aws_region)
    error_message = "Only approved regions allowed: us-east-1, us-west-2, eu-west-1, ap-southeast-1"
  }
}
```

---

### Debugging Variables & Locals

```bash
# ── See all output values after apply ─────────────────────────────
terraform output
terraform output -json   # Machine-readable JSON

# ── Get specific output value ──────────────────────────────────────
terraform output vpc_id
terraform output -raw vpc_id   # No quotes (good for shell scripts)

# ── Interactive console — test expressions LIVE ───────────────────
terraform console

# Inside console:
> var.environment
"prod"
> local.name_prefix
"myapp-prod"
> local.env_config["prod"]
{ instance_type = "t2.large", min_size = 3, max_size = 10 }
> cidrsubnet("10.0.0.0/16", 8, 3)
"10.0.3.0/24"
> merge({ a = 1 }, { b = 2 })
{ a = 1, b = 2 }
> [for i in range(3) : "subnet-${i}"]
["subnet-0", "subnet-1", "subnet-2"]

# ── Check what variables Terraform sees ───────────────────────────
terraform plan -var="environment=prod" 2>&1 | head -50

# ── See all variables consumed (plan with full trace) ─────────────
TF_LOG=TRACE terraform plan 2>&1 | grep "variable"
```

---

## 📁 CODE WRITING GUIDANCE

### Professional Variables File Structure

```hcl
# variables.tf — Professional structure

# ═══════════════════════════════════════════════════════════════════
# SECTION 1: PROJECT IDENTITY
# ═══════════════════════════════════════════════════════════════════

variable "project_name" {
  type        = string
  description = "Name of the project. Used as prefix for all resources."

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,20}$", var.project_name))
    error_message = "Project name must be 3-20 chars, start with letter, lowercase letters/numbers/hyphens only."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment. Controls sizing and redundancy."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be: dev, staging, or prod."
  }
}

variable "aws_region" {
  type        = string
  description = "AWS region for resource deployment."
  default     = "us-east-1"
}

variable "owner" {
  type        = string
  description = "Team or person responsible for these resources (for tagging/billing)."
}

# ═══════════════════════════════════════════════════════════════════
# SECTION 2: NETWORKING
# ═══════════════════════════════════════════════════════════════════

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC."
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid CIDR block."
  }
}

variable "max_azs" {
  type        = number
  description = "Maximum number of availability zones to use."
  default     = 3

  validation {
    condition     = var.max_azs >= 1 && var.max_azs <= 6
    error_message = "max_azs must be between 1 and 6."
  }
}

# ═══════════════════════════════════════════════════════════════════
# SECTION 3: COMPUTE
# ═══════════════════════════════════════════════════════════════════

variable "instance_type" {
  type        = string
  description = "EC2 instance type. Defaults are environment-appropriate."
  default     = null   # null = use environment default from locals
}

# ═══════════════════════════════════════════════════════════════════
# SECTION 4: DATABASE
# ═══════════════════════════════════════════════════════════════════

variable "db_instance_class" {
  type        = string
  description = "RDS instance class."
  default     = null   # null = use environment default from locals
}

variable "db_username" {
  type        = string
  description = "Database master username."
  default     = "dbadmin"
}

variable "db_password" {
  type        = string
  description = "Database master password. Must be at least 16 characters."
  sensitive   = true

  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Database password must be at least 16 characters."
  }
}

# ═══════════════════════════════════════════════════════════════════
# SECTION 5: FEATURE FLAGS
# ═══════════════════════════════════════════════════════════════════

variable "enable_monitoring" {
  type        = bool
  description = "Enable detailed CloudWatch monitoring."
  default     = false
}

variable "enable_backups" {
  type        = bool
  description = "Enable automated backups."
  default     = true
}
```

---

## 🧪 HANDS-ON LAB

### Exercise: Dynamic Multi-Environment Configuration

**Goal:** Build a configuration that:
1. Accepts `environment` variable (dev/staging/prod)
2. Uses a locals lookup table to set ALL sizing/config per environment
3. Creates an RDS-style config (no actual RDS needed — just local + output demo)
4. Validates that `db_password` is at least 12 characters
5. Has a `sensitive` output for connection string
6. Has a non-sensitive output showing: `{ environment, instance_type, db_class, multi_az }`
7. In `terraform console`, verify expressions work correctly

**Expected Output:**
```bash
$ terraform apply -var="environment=prod" -var="db_password=supersecret999"

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

connection_string = <sensitive>
environment_summary = {
  "db_class"      = "db.t3.large"
  "environment"   = "prod"
  "instance_type" = "t2.large"
  "multi_az"      = true
}
```

**Try it yourself first!**

---

### ✅ Solution

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Target environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be: dev, staging, or prod."
  }
}

variable "project_name" {
  type        = string
  description = "Project name used in resource naming"
  default     = "myapp"
}

variable "db_username" {
  type        = string
  description = "Database master username"
  default     = "dbadmin"
}

variable "db_password" {
  type        = string
  description = "Database master password (minimum 12 characters)"
  sensitive   = true

  validation {
    condition     = length(var.db_password) >= 12
    error_message = "Database password must be at least 12 characters long."
  }
}
```

```hcl
# locals.tf
locals {
  # ── Environment Configuration Lookup Table ──────────────────────
  env_config = {
    dev = {
      instance_type  = "t2.micro"
      db_class       = "db.t3.micro"
      multi_az       = false
      backup_days    = 1
      log_retention  = 7
    }
    staging = {
      instance_type  = "t2.small"
      db_class       = "db.t3.small"
      multi_az       = false
      backup_days    = 7
      log_retention  = 14
    }
    prod = {
      instance_type  = "t2.large"
      db_class       = "db.t3.large"
      multi_az       = true
      backup_days    = 30
      log_retention  = 90
    }
  }

  # Current environment settings
  config = local.env_config[var.environment]

  # ── Derived Values ───────────────────────────────────────────────
  name_prefix = "${var.project_name}-${var.environment}"

  # Sensitive: connection string derived from sensitive password
  db_connection_string = sensitive(
    "postgresql://${var.db_username}:${var.db_password}@${local.name_prefix}-db.example.com:5432/appdb"
  )
}
```

```hcl
# outputs.tf
output "environment_summary" {
  description = "Current environment configuration summary"
  value = {
    environment   = var.environment
    instance_type = local.config.instance_type
    db_class      = local.config.db_class
    multi_az      = local.config.multi_az
  }
}

output "connection_string" {
  description = "Database connection string (sensitive)"
  value       = local.db_connection_string
  sensitive   = true
}

output "name_prefix" {
  description = "Name prefix used for all resources"
  value       = local.name_prefix
}

output "backup_retention_days" {
  description = "Number of days to retain backups"
  value       = local.config.backup_days
}
```

```bash
# Test it:
terraform init
terraform apply -var="environment=prod" -var="db_password=supersecret999abc"

# View specific output
terraform output environment_summary

# View sensitive output (requires explicit request)
terraform output -raw connection_string

# Test in console
terraform console
> local.config
> local.name_prefix
> local.env_config["staging"]
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| `variable` | Input to your module. Can be overridden by users. |
| `local` | Internal computed value. Cannot be overridden externally. |
| `output` | Expose values to users or parent modules. |
| `sensitive = true` | Hides from CLI output. Still in state file plaintext. |
| `validation` | Validate values before any resource is created. |
| `try(expr, default)` | Safe evaluation — returns default if expr errors. |
| `can(expr)` | Returns bool — true if expr evaluates without error. |
| `for` expression | Transform/filter lists and maps. |
| `merge()` | Combine maps — rightmost wins on key conflict. |
| `flatten()` | Convert nested lists to single-level list. |
| `templatefile()` | Render external file templates with variable substitution. |

---

### Quick Syntax Reference

```hcl
# Variable
variable "NAME" {
  type        = string | number | bool | list() | map() | object() | set()
  description = "..."
  default     = VALUE
  sensitive   = true
  validation  { condition = EXPR; error_message = "..." }
}

# Use variable
var.NAME

# Local
locals {
  NAME = EXPRESSION
}

# Use local
local.NAME

# Output
output "NAME" {
  description = "..."
  value       = EXPRESSION
  sensitive   = true
  precondition { condition = EXPR; error_message = "..." }
}

# For expression — list
[for item in LIST : TRANSFORM if CONDITION]

# For expression — map
{ for key, value in MAP : NEW_KEY => NEW_VALUE if CONDITION }

# Conditional
CONDITION ? true_value : false_value

# String interpolation
"${var.project}-${var.environment}"

# Heredoc
<<-EOF
  multi
  line
  string
EOF

# Try / Can
try(expression, fallback_value)
can(expression)     # returns bool

# Coalesce
coalesce(val1, val2, val3)   # first non-null/empty

# Common functions
merge(map1, map2)
flatten([list1, list2])
toset(list)
tolist(set)
zipmap(keys_list, values_list)
jsonencode(object)
jsondecode(string)
templatefile(path, vars)
file(path)
format("pattern", args...)
cidrsubnet(cidr, newbits, netnum)
```

---

### Top 10 Interview Questions

1. **What's the difference between `variable` and `local`?** Variables are user-overridable inputs. Locals are internal computed values only the module itself defines.

2. **What's the variable value precedence order?** CLI `-var` > `-var-file` > `terraform.tfvars`/`*.auto.tfvars` > `TF_VAR_*` env vars > `default`.

3. **What does `sensitive = true` actually protect?** Hides value from CLI output and plan display. It does NOT encrypt the value in the state file — state file encryption is separate.

4. **Can a local reference another local?** Yes, Terraform resolves them in dependency order. Circular references cause an error.

5. **How do outputs enable module composition?** A child module's outputs are accessed via `module.module_name.output_name` in the parent module, enabling data flow between modules.

6. **What's the difference between `try()` and `can()`?** `try()` evaluates and returns the first non-error result (or fallback). `can()` just returns a boolean indicating if the expression is error-free.

7. **How do you pass a list of objects as a variable?** Use `type = list(object({ ... }))` with object shape defined. Access via `for` expressions or index.

8. **What is a `for` expression used for?** Transforming or filtering collections (lists/maps) into new lists or maps — similar to `map()`/`filter()` in other languages.

9. **When would you use `templatefile()` vs `heredoc`?** `templatefile()` for complex, reusable templates stored in separate files. Heredoc for short inline strings.

10. **How do you make an output available to a CI/CD pipeline?** `terraform output -raw output_name` returns the raw value (no quotes) that can be captured as a shell variable: `URL=$(terraform output -raw app_url)`.

---
