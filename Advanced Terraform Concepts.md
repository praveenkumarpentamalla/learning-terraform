# Advanced Terraform Concepts: Complete Guide — Production-Ready Mastery

---

# 📘 BEGINNER ORIENTATION

## What Are "Advanced Terraform Concepts"?

Before diving deep, here is the **complete map** of what this topic covers and why each concept matters:

```
Advanced Terraform Concepts
│
├── 1. Terraform Functions        → Transform and compute values in HCL
├── 2. Dynamic Blocks             → Generate repeated nested blocks programmatically
├── 3. Expressions & Conditionals → Logic, loops, filtering inside HCL
├── 4. Moved Blocks               → Rename/move resources without destroying them
├── 5. Import Blocks              → Bring existing infra under Terraform control
├── 6. Check Blocks               → Custom validation and health assertions
├── 7. Terraform Testing          → Unit and integration test your infrastructure
├── 8. Custom Conditions          → Preconditions, postconditions, validations
└── 9. Performance & Scale        → Large state, parallelism, optimization
```

Each of these solves a **real production problem**. By the end of this guide you will understand not just how they work but **exactly when and why to reach for each one**.

---

# 📗 SECTION 1: Terraform Functions — Complete Reference

## What Are Functions?

Functions are **built-in transformations** you can call anywhere in HCL to process and compute values. Terraform has 100+ built-in functions. There are no user-defined functions in HCL — but `locals` blocks can simulate them.

```hcl
# General function call syntax
function_name(argument1, argument2, ...)

# Functions can be composed (nested)
upper(join("-", ["hello", "world"]))   # "HELLO-WORLD"
```

---

## Category 1: String Functions

```hcl
locals {
  # ---- CASE MANIPULATION ----
  upper_env   = upper("production")          # "PRODUCTION"
  lower_env   = lower("PRODUCTION")          # "production"
  title_env   = title("hello world")         # "Hello World"

  # ---- TRIMMING & CLEANING ----
  raw_ip      = "  10.0.0.1\n"
  clean_ip    = trimspace(local.raw_ip)       # "10.0.0.1"
  no_prefix   = trimprefix("vpc-abc123", "vpc-") # "abc123"
  no_suffix   = trimsuffix("server.internal", ".internal") # "server"
  trimmed     = trim("***hello***", "*")      # "hello"

  # ---- JOINING & SPLITTING ----
  parts       = ["web", "server", "01"]
  joined      = join("-", local.parts)        # "web-server-01"
  csv_string  = "us-east-1a,us-east-1b,us-east-1c"
  az_list     = split(",", local.csv_string)  # ["us-east-1a","us-east-1b","us-east-1c"]

  # ---- SEARCHING & REPLACING ----
  has_prod    = strcontains("prod-server", "prod")  # true
  replaced    = replace("hello-world", "-", "_")    # "hello_world"
  # Regex replace
  sanitized   = replace("my app name!", "/[^a-z0-9-]/", "-") # "my-app-name-"

  # ---- FORMATTING ----
  formatted   = format("%-10s | %05d | %s", "server", 42, "running")
  # "server     | 00042 | running"
  formatted_list = formatlist("server-%02d", [1, 2, 3])
  # ["server-01", "server-02", "server-03"]

  # ---- SUBSTRINGS ----
  region      = "us-east-1"
  short       = substr(local.region, 0, 7)    # "us-east"
  last_char   = substr(local.region, -1, 1)   # "1"

  # ---- ENCODING ----
  b64_encoded = base64encode("hello world")   # "aGVsbG8gd29ybGQ="
  b64_decoded = base64decode("aGVsbG8gd29ybGQ=") # "hello world"
  url_encoded = urlencode("hello world&more") # "hello+world%26more"
}
```

---

## Category 2: Collection Functions

```hcl
locals {
  servers     = ["web-01", "api-01", "db-01", "web-02"]
  server_map  = { web = "t2.micro", api = "t2.small", db = "t3.large" }

  # ---- LENGTH ----
  count       = length(local.servers)         # 4
  map_count   = length(local.server_map)      # 3

  # ---- SLICING & INDEXING ----
  first_two   = slice(local.servers, 0, 2)    # ["web-01","api-01"]
  first       = element(local.servers, 0)     # "web-01" (wraps around!)
  fifth       = element(local.servers, 4)     # "web-01" (wraps: 4 % 4 = 0)

  # ---- CONVERTING TYPES ----
  as_set      = toset(local.servers)          # Set (no duplicates, unordered)
  as_list     = tolist(local.as_set)          # Back to list
  as_map      = { for i, v in local.servers : v => i }

  # ---- SEARCHING ----
  has_web     = contains(local.servers, "web-01")        # true
  web_index   = index(local.servers, "api-01")           # 1
  db_type     = lookup(local.server_map, "db", "unknown") # "t3.large"
  safe_lookup = lookup(local.server_map, "cache", "t2.micro") # "t2.micro" (default)

  # ---- SORTING ----
  sorted      = sort(["z", "a", "m"])         # ["a","m","z"]
  reversed    = reverse(local.sorted)         # ["z","m","a"]

  # ---- SET OPERATIONS ----
  list_a      = ["a", "b", "c"]
  list_b      = ["b", "c", "d"]
  union       = setunion(local.list_a, local.list_b)        # ["a","b","c","d"]
  intersect   = setintersection(local.list_a, local.list_b) # ["b","c"]
  subtract    = setsubtract(local.list_a, local.list_b)     # ["a"]
  product     = setproduct(["a","b"], [1,2])  # [["a",1],["a",2],["b",1],["b",2]]

  # ---- MAP OPERATIONS ----
  merged      = merge({ a = 1 }, { b = 2 }, { c = 3 })   # {a=1, b=2, c=3}
  keys_only   = keys(local.server_map)        # ["api","db","web"]
  values_only = values(local.server_map)      # ["t2.small","t3.large","t2.micro"]

  # ---- FLATTENING ----
  nested      = [["a","b"], ["c","d"], ["e"]]
  flat        = flatten(local.nested)         # ["a","b","c","d","e"]

  # ---- REMOVING NULLS ----
  with_nulls  = ["a", null, "b", null, "c"]
  clean       = compact(local.with_nulls)     # ["a","b","c"]
  map_nulls   = { a = "val", b = null, c = "val2" }
  clean_map   = { for k, v in local.map_nulls : k => v if v != null }
}
```

---

## Category 3: Numeric Functions

```hcl
locals {
  # Basic math
  max_val   = max(3, 1, 4, 1, 5, 9, 2, 6)   # 9
  min_val   = min(3, 1, 4, 1, 5, 9, 2, 6)   # 1
  absolute  = abs(-42)                        # 42
  ceiling   = ceil(4.1)                       # 5
  floor_val = floor(4.9)                      # 4
  powered   = pow(2, 10)                      # 1024
  logged    = log(1024, 2)                    # 10

  # Practical: calculate number of subnets needed
  az_count       = 3
  bits_needed    = ceil(log(local.az_count, 2))  # bits to represent 3 AZs
  # log(3, 2) = 1.58 → ceil = 2 bits → can represent 4 subnets
}
```

---

## Category 4: Network & IP Functions

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"

  # ---- CIDR MANIPULATION ----
  # cidrsubnet(prefix, newbits, netnum)
  # newbits: how many bits to ADD to the prefix
  # netnum: which subnet number (0-indexed)

  subnet_0   = cidrsubnet(local.vpc_cidr, 8, 0)   # "10.0.0.0/24"
  subnet_1   = cidrsubnet(local.vpc_cidr, 8, 1)   # "10.0.1.0/24"
  subnet_10  = cidrsubnet(local.vpc_cidr, 8, 10)  # "10.0.10.0/24"
  subnet_255 = cidrsubnet(local.vpc_cidr, 8, 255) # "10.0.255.0/24"

  # Create /20 subnets (4 bits added to /16)
  large_subnet = cidrsubnet(local.vpc_cidr, 4, 0)  # "10.0.0.0/20"

  # cidrhost — get specific IP address in a CIDR block
  gateway_ip  = cidrhost(local.subnet_0, 1)    # "10.0.0.1"  (first usable)
  last_ip     = cidrhost(local.subnet_0, -2)   # "10.0.0.254" (last usable)

  # cidrnetmask — get the subnet mask
  netmask     = cidrnetmask("10.0.0.0/24")     # "255.255.255.0"

  # Generate subnets for all AZs dynamically
  az_count        = 3
  public_subnets  = [for i in range(local.az_count) : cidrsubnet(local.vpc_cidr, 8, i)]
  private_subnets = [for i in range(local.az_count) : cidrsubnet(local.vpc_cidr, 8, i + 10)]
  db_subnets      = [for i in range(local.az_count) : cidrsubnet(local.vpc_cidr, 8, i + 20)]
  # public:  ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
  # private: ["10.0.10.0/24","10.0.11.0/24","10.0.12.0/24"]
  # db:      ["10.0.20.0/24","10.0.21.0/24","10.0.22.0/24"]
}
```

---

## Category 5: Type Conversion & Encoding Functions

```hcl
locals {
  # ---- TYPE CONVERSION ----
  num_str    = tostring(42)              # "42"
  str_num    = tonumber("42")           # 42
  as_bool    = tobool("true")           # true
  as_list    = tolist(["a","b","c"])    # list(string)
  as_set     = toset(["a","b","a"])     # {"a","b"}  (deduped)

  # ---- JSON ENCODING/DECODING ----
  data_map   = { host = "db.internal", port = 5432, ssl = true }
  as_json    = jsonencode(local.data_map)
  # "{\"host\":\"db.internal\",\"port\":5432,\"ssl\":true}"

  json_str   = "{\"key\":\"value\",\"count\":3}"
  as_object  = jsondecode(local.json_str)
  # { key = "value", count = 3 }

  # ---- YAML ENCODING ----
  as_yaml    = yamlencode(local.data_map)
  # "host: db.internal\nport: 5432\nssl: true\n"
  from_yaml  = yamldecode("host: db.internal\nport: 5432\n")

  # ---- HASHING ----
  hash_md5   = md5("my-data")           # md5 hash string
  hash_sha1  = sha1("my-data")          # sha1 hash string
  hash_sha256 = sha256("my-data")       # sha256 hash string

  # Practical: force resource replacement when config file changes
  config_hash = md5(file("${path.module}/configs/app.conf"))
}

# Force instance replacement when user_data changes
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  user_data = templatefile("${path.module}/scripts/setup.sh", {
    version = var.app_version
  })

  # Hash the user_data to detect changes and force replacement
  user_data_replace_on_change = true

  tags = {
    # Forces replacement when config file changes
    ConfigHash = md5(file("${path.module}/configs/app.conf"))
  }
}
```

---

## Category 6: Date & Time Functions

```hcl
locals {
  # Current timestamp (at plan time — use carefully!)
  now         = timestamp()              # "2024-01-15T10:30:00Z"

  # Add/subtract time
  one_week    = timeadd(local.now, "168h")  # 7 days later
  one_hour_ago = timeadd(local.now, "-1h")  # 1 hour earlier

  # Compare timestamps (returns -1, 0, or 1)
  is_after    = timecmp(local.one_week, local.now)   # 1 (one_week > now)

  # Format a timestamp
  date_str    = formatdate("YYYY-MM-DD", local.now)  # "2024-01-15"
  human_date  = formatdate("DD MMM YYYY hh:mm:ss", local.now) # "15 Jan 2024 10:30:00"
}

# Practical: tag resources with creation date
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  tags = {
    Name        = "web-server"
    CreatedAt   = formatdate("YYYY-MM-DD", timestamp())
    # ⚠️ timestamp() changes on every plan — this will always show a diff!
    # ✅ Better: use ignore_changes = [tags["CreatedAt"]]
  }

  lifecycle {
    ignore_changes = [tags["CreatedAt"]]
  }
}
```

---

## Category 7: Filesystem Functions

```hcl
# These functions run at PLAN TIME on the machine running Terraform

# Read file contents as a string
resource "aws_iam_policy" "app" {
  policy = file("${path.module}/policies/app-policy.json")
}

# Check if file exists before reading
locals {
  has_override = fileexists("${path.module}/override.json")
  config       = local.has_override ? jsondecode(file("${path.module}/override.json")) : {}
}

# Get MD5 of a file (detect changes)
locals {
  script_hash = filemd5("${path.module}/scripts/setup.sh")
  script_sha  = filesha256("${path.module}/scripts/setup.sh")
}

# Read base64-encoded binary file (for Lambda zip, certs etc)
resource "aws_lambda_function" "api" {
  function_name    = "my-api"
  filename         = "${path.module}/lambda.zip"
  source_code_hash = filebase64sha256("${path.module}/lambda.zip")
  handler          = "index.handler"
  runtime          = "nodejs18.x"
  role             = aws_iam_role.lambda.arn
}

# Render a template file
resource "aws_instance" "web" {
  ami       = data.aws_ami.ubuntu.id
  user_data = templatefile("${path.module}/templates/cloud-init.yaml.tpl", {
    hostname    = "web-${terraform.workspace}"
    environment = terraform.workspace
    packages    = ["nginx", "nodejs", "pm2"]
  })
}
```

---

# 📗 SECTION 2: Expressions & Advanced Logic

## For Expressions — The Most Powerful HCL Feature

```hcl
# ============================================
# FOR EXPRESSION SYNTAX
# ============================================

# Transform a list → list
[for <item> in <list> : <expression>]

# Transform a list → map
{for <item> in <list> : <key> => <value>}

# Transform a map → map
{for <key>, <value> in <map> : <new_key> => <new_value>}

# With filtering (if clause)
[for <item> in <list> : <expression> if <condition>]


# ============================================
# PRACTICAL EXAMPLES
# ============================================

variable "servers" {
  default = [
    { name = "web-01",    role = "web",    size = "small",  active = true  },
    { name = "api-01",    role = "api",    size = "medium", active = true  },
    { name = "worker-01", role = "worker", size = "small",  active = false },
    { name = "db-01",     role = "db",     size = "large",  active = true  },
  ]
}

locals {
  # LIST → LIST: Extract just the names
  server_names = [for s in var.servers : s.name]
  # ["web-01", "api-01", "worker-01", "db-01"]

  # LIST → LIST with transform: uppercase all names
  upper_names = [for s in var.servers : upper(s.name)]
  # ["WEB-01", "API-01", "WORKER-01", "DB-01"]

  # LIST → LIST with FILTER: only active servers
  active_servers = [for s in var.servers : s.name if s.active]
  # ["web-01", "api-01", "db-01"]

  # LIST → MAP: name as key, role as value
  server_roles = {for s in var.servers : s.name => s.role}
  # { "web-01" = "web", "api-01" = "api", ... }

  # LIST → MAP with transform: name → full object (for for_each)
  server_map = {for s in var.servers : s.name => s}
  # { "web-01" = { name="web-01", role="web", ... }, ... }

  # LIST → MAP with FILTER: only active, name → size
  active_sizes = {for s in var.servers : s.name => s.size if s.active}
  # { "web-01" = "small", "api-01" = "medium", "db-01" = "large" }

  # MAP → MAP: transform map values
  instance_types = {
    small  = "t2.micro"
    medium = "t2.small"
    large  = "t3.large"
  }
  server_instance_types = {
    for s in var.servers :
    s.name => local.instance_types[s.size]
  }
  # { "web-01"="t2.micro", "api-01"="t2.small", "db-01"="t3.large" }

  # MAP → MAP: swap keys and values
  roles_to_names = {
    for name, role in local.server_roles :
    role => name
  }
  # { "web"="web-01", "api"="api-01", "worker"="worker-01", "db"="db-01" }

  # GROUP BY: multiple items with same key (use ... spread operator)
  servers_by_role = {
    for s in var.servers :
    s.role => s.name...   # ← ... groups values with same key into a list
  }
  # { "web"=["web-01"], "api"=["api-01"], "worker"=["worker-01"], "db"=["db-01"] }
}
```

---

## Conditional Expressions — Deep Dive

```hcl
# Basic ternary
condition ? true_value : false_value

locals {
  # Simple boolean condition
  instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"

  # Null check — use something or fall back to default
  custom_domain = var.domain_name != null ? var.domain_name : "app.${var.environment}.example.com"

  # Chained ternary (use sparingly — hard to read)
  db_size = (
    var.environment == "prod"    ? "db.r5.large"   :
    var.environment == "staging" ? "db.t3.small"   :
    "db.t3.micro"                                  # dev default
  )

  # Conditional list inclusion
  base_features   = ["feature-a", "feature-b"]
  extra_features  = var.is_premium ? ["feature-c", "feature-d"] : []
  all_features    = concat(local.base_features, local.extra_features)

  # Conditional with null resource reference
  # Don't reference resource that might not exist:
  monitoring_arn = var.enable_monitoring ? aws_cloudwatch_log_group.app[0].arn : null

  # Null coalescing — try() function
  # try() returns the first expression that doesn't throw an error
  safe_lookup = try(var.config["key"], "default-value")
  safe_attr   = try(data.aws_vpc.main.cidr_block, "10.0.0.0/16")
}


# Conditional resource blocks
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name          = "high-cpu-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 120
  statistic           = "Average"
  threshold           = var.environment == "prod" ? 70 : 90
}

# Safe reference to the optional resource
output "alarm_arn" {
  value = var.enable_monitoring ? aws_cloudwatch_metric_alarm.high_cpu[0].arn : "monitoring-disabled"
}
```

---

## The `try()` and `can()` Functions

```hcl
# try() — attempt an expression, return fallback if it errors
# can() — return true if expression succeeds, false if it errors

locals {
  # ---- try() examples ----

  # Safe attribute access (object might not have the key)
  config = { host = "db.internal" }
  port   = try(local.config.port, 5432)       # Returns 5432 (port doesn't exist)
  host   = try(local.config.host, "localhost") # Returns "db.internal"

  # Safe type conversion
  safe_number = try(tonumber("abc"), 0)   # Returns 0 (can't convert "abc")
  real_number = try(tonumber("42"), 0)    # Returns 42

  # Safe lookup in nested structure
  app_config = {
    database = {
      primary = { host = "db-1.internal", port = 5432 }
    }
  }
  primary_port = try(local.app_config.database.primary.port, 5432)   # 5432
  replica_port = try(local.app_config.database.replica.port, 5432)   # 5432 (no replica key)

  # Safe external data parsing
  raw_json     = "{\"key\": \"value\"}"
  bad_json     = "not-valid-json"
  parsed_good  = try(jsondecode(local.raw_json), {})    # { key = "value" }
  parsed_bad   = try(jsondecode(local.bad_json), {})    # {}  (fallback)


  # ---- can() examples ----

  # Check if a conversion is possible
  is_number    = can(tonumber(var.input_value))    # true if var can be a number
  is_valid_json = can(jsondecode(var.json_input))  # true if valid JSON

  # Use can() for validation
  valid_cidr   = can(cidrnetmask(var.vpc_cidr))    # true if valid CIDR notation
}

# Practical: Optional variable with nested structure
variable "database_config" {
  type    = any
  default = null
}

locals {
  db_host = try(var.database_config.host, "localhost")
  db_port = try(var.database_config.port, 5432)
  db_ssl  = try(var.database_config.ssl,  true)
}
```

---

# 📗 SECTION 3: Dynamic Blocks — Advanced Patterns

## Deep Dive into Dynamic Blocks

```hcl
# ============================================
# DYNAMIC BLOCK SYNTAX
# ============================================
dynamic "<block_label>" {
  for_each = <collection>
  iterator = <custom_iterator_name>  # Optional, defaults to block_label

  content {
    # Use <iterator>.key and <iterator>.value (or <iterator>.value.attribute)
    attribute = <iterator>.value
  }
}


# ============================================
# PATTERN 1: Generate security group rules from variable
# ============================================
variable "security_rules" {
  type = list(object({
    type        = string           # "ingress" or "egress"
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    { type="ingress", from_port=80,  to_port=80,  protocol="tcp",
      cidr_blocks=["0.0.0.0/0"],    description="HTTP" },
    { type="ingress", from_port=443, to_port=443, protocol="tcp",
      cidr_blocks=["0.0.0.0/0"],    description="HTTPS" },
    { type="ingress", from_port=22,  to_port=22,  protocol="tcp",
      cidr_blocks=["10.0.0.0/8"],   description="SSH internal" },
    { type="egress",  from_port=0,   to_port=0,   protocol="-1",
      cidr_blocks=["0.0.0.0/0"],    description="All outbound" },
  ]
}

resource "aws_security_group" "app" {
  name   = "app-sg-${var.environment}"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = [for r in var.security_rules : r if r.type == "ingress"]
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  dynamic "egress" {
    for_each = [for r in var.security_rules : r if r.type == "egress"]
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
      description = egress.value.description
    }
  }
}


# ============================================
# PATTERN 2: Custom iterator name (cleaner code)
# ============================================
variable "environment_vars" {
  default = {
    APP_ENV     = "production"
    DB_HOST     = "db.internal"
    CACHE_HOST  = "redis.internal"
    LOG_LEVEL   = "WARN"
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "myapp"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([{
    name  = "app"
    image = "myapp:latest"

    environment = [
      for key, value in var.environment_vars : {
        name  = key
        value = value
      }
    ]
  }])
}


# ============================================
# PATTERN 3: Nested dynamic blocks
# ============================================
variable "load_balancer_rules" {
  type = list(object({
    priority = number
    conditions = list(object({
      field  = string
      values = list(string)
    }))
    target_group_arn = string
  }))
}

resource "aws_lb_listener_rule" "routing" {
  count        = length(var.load_balancer_rules)
  listener_arn = aws_lb_listener.https.arn
  priority     = var.load_balancer_rules[count.index].priority

  action {
    type             = "forward"
    target_group_arn = var.load_balancer_rules[count.index].target_group_arn
  }

  dynamic "condition" {
    for_each = var.load_balancer_rules[count.index].conditions
    content {
      dynamic "path_pattern" {
        for_each = condition.value.field == "path-pattern" ? [condition.value] : []
        content {
          values = path_pattern.value.values
        }
      }

      dynamic "host_header" {
        for_each = condition.value.field == "host-header" ? [condition.value] : []
        content {
          values = host_header.value.values
        }
      }
    }
  }
}


# ============================================
# PATTERN 4: Dynamic block with computed for_each
# ============================================
locals {
  # Only add certain tags in production
  extra_tags = var.environment == "prod" ? {
    CostCenter  = "engineering"
    Compliance  = "sox"
    DataClass   = "confidential"
  } : {}

  all_tags = merge(local.common_tags, local.extra_tags)
}

resource "aws_autoscaling_group" "app" {
  min_size = 1
  max_size = 5

  dynamic "tag" {
    for_each = local.all_tags

    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}
```

---

# 📕 SECTION 4: Moved Blocks

## What Are Moved Blocks?

`moved` blocks let you **rename or reorganize resources in Terraform state** without destroying and recreating them. This is one of the most important production safety features added to Terraform (1.1+).

**The Problem they solve:**

```hcl
# Before: resource has this address in state
resource "aws_instance" "web" { }
# State: aws_instance.web

# You want to rename it to:
resource "aws_instance" "web_server" { }
# Without moved block: Terraform would DESTROY "web" and CREATE "web_server"
# That's a production outage for a simple rename!
```

---

## Complete Moved Block Reference

```hcl
# ============================================
# PATTERN 1: Simple rename
# ============================================
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

resource "aws_instance" "web_server" {
  # (previously named "web")
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
}


# ============================================
# PATTERN 2: Moving into a module
# ============================================
# Before: resource was in root module
# resource "aws_s3_bucket" "app_data" { }

# After: refactored into a module
module "storage" {
  source = "./modules/storage"
}

moved {
  from = aws_s3_bucket.app_data          # Root module resource
  to   = module.storage.aws_s3_bucket.main  # Module resource
}


# ============================================
# PATTERN 3: count → for_each migration
# ============================================
# Before (count-based, fragile):
# resource "aws_iam_user" "devs" {
#   count = length(var.dev_names)
#   name  = var.dev_names[count.index]
# }
# State: aws_iam_user.devs[0], aws_iam_user.devs[1], aws_iam_user.devs[2]

# After (for_each, stable):
resource "aws_iam_user" "devs" {
  for_each = toset(var.dev_names)
  name     = each.value
}
# State target: aws_iam_user.devs["alice"], aws_iam_user.devs["bob"]

# Migration — map each old index to new key
moved {
  from = aws_iam_user.devs[0]
  to   = aws_iam_user.devs["alice"]
}

moved {
  from = aws_iam_user.devs[1]
  to   = aws_iam_user.devs["bob"]
}

moved {
  from = aws_iam_user.devs[2]
  to   = aws_iam_user.devs["charlie"]
}


# ============================================
# PATTERN 4: Moving between modules (refactoring)
# ============================================
# Before: networking was in a "legacy" module
# module "legacy_network" {
#   source = "./modules/legacy-network"
# }
# State: module.legacy_network.aws_vpc.main

# After: refactored to new module
module "networking" {
  source = "./modules/networking"
}

moved {
  from = module.legacy_network.aws_vpc.main
  to   = module.networking.aws_vpc.main
}

moved {
  from = module.legacy_network.aws_subnet.public[0]
  to   = module.networking.aws_subnet.public[0]
}

moved {
  from = module.legacy_network.aws_subnet.private[0]
  to   = module.networking.aws_subnet.private[0]
}


# ============================================
# PATTERN 5: Splitting a module into two
# ============================================
# Before: single "app" module had everything
# After: split into "compute" and "database" modules

moved {
  from = module.app.aws_instance.server
  to   = module.compute.aws_instance.server
}

moved {
  from = module.app.aws_db_instance.main
  to   = module.database.aws_db_instance.main
}
```

---

## Moved Blocks in Practice

```bash
# Workflow for safe resource renaming:

# Step 1: Add the moved block to your configuration
# Step 2: Run plan — Terraform shows "move" not "destroy+create"
terraform plan
# Output should show:
#   # aws_instance.web_server will be updated in-place
#   # (moved from aws_instance.web)

# Step 3: Apply — no destruction!
terraform apply

# Step 4: After confirmed working, you CAN remove the moved block
# (but it's safe to leave it in — it's idempotent)

# ⚠️ If you see destroy+create in the plan, STOP
# The moved block isn't matching correctly
# Check: exact resource addresses, module paths, index values
```

---

# 📕 SECTION 5: Import Blocks (Terraform 1.5+)

## What Are Import Blocks?

Import blocks let you **bring existing infrastructure under Terraform management** declaratively — written in code, reviewable in a plan, and reproducible.

Before Terraform 1.5, the only way to import was the imperative `terraform import` CLI command, which was error-prone and not code-reviewable.

```hcl
# ============================================
# OLD WAY (CLI import — still works but limited)
# ============================================
# terraform import aws_instance.web i-0a1b2c3d4e5f67890
# ❌ Not in code — can't review in PR
# ❌ Must know exact resource address
# ❌ Must run manually — not reproducible

# ============================================
# NEW WAY (Import blocks — Terraform 1.5+)
# ============================================
import {
  id = "i-0a1b2c3d4e5f67890"            # The real-world resource ID
  to = aws_instance.web                   # Where to put it in state
}

resource "aws_instance" "web" {
  # You still need to write (or generate) the resource config
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  # ... other attributes matching the existing resource
}
```

---

## Generate Config with `-generate-config-out` (Terraform 1.5+)

```bash
# The most powerful workflow:
# 1. Write the import block pointing to the existing resource
# 2. Let Terraform GENERATE the resource configuration for you!

# Step 1: Write just the import block (no resource block yet)
cat > import.tf << 'EOF'
import {
  id = "i-0a1b2c3d4e5f67890"
  to = aws_instance.existing_web_server
}
EOF

# Step 2: Let Terraform generate the resource config
terraform plan -generate-config-out=generated_resources.tf

# Step 3: Review generated file
cat generated_resources.tf
# resource "aws_instance" "existing_web_server" {
#   ami                    = "ami-0c55b159cbfafe1f0"
#   instance_type          = "t2.micro"
#   availability_zone      = "us-east-1a"
#   vpc_security_group_ids = ["sg-0a1b2c3d"]
#   subnet_id              = "subnet-0a1b2c3d"
#   ... (complete configuration!)
# }

# Step 4: Review and clean up the generated config
# Remove any read-only attributes, tidy up formatting

# Step 5: Apply to add to state
terraform apply
```

---

## Bulk Import Pattern

```hcl
# ============================================
# Import multiple existing resources at once
# ============================================

# Existing S3 buckets created manually — bring them all under Terraform
import {
  id = "company-production-data"
  to = aws_s3_bucket.production_data
}

import {
  id = "company-staging-data"
  to = aws_s3_bucket.staging_data
}

import {
  id = "company-backups"
  to = aws_s3_bucket.backups
}

# Existing IAM roles
import {
  id = "existing-lambda-execution-role"
  to = aws_iam_role.lambda_execution
}

# Existing security groups
import {
  id = "sg-0a1b2c3d4e5f67890"
  to = aws_security_group.legacy_web
}

# After running plan + apply, all these resources are in state
# and Terraform manages their full lifecycle going forward
```

---

## Import Block with for_each (Terraform 1.7+)

```hcl
# ============================================
# Import multiple similar resources at once
# ============================================
locals {
  existing_buckets = {
    logs     = "company-logs-bucket-prod"
    backups  = "company-backups-bucket-prod"
    artifacts = "company-artifacts-bucket-prod"
  }
}

import {
  for_each = local.existing_buckets
  id       = each.value                      # Real S3 bucket name
  to       = aws_s3_bucket.managed[each.key] # Target resource address
}

resource "aws_s3_bucket" "managed" {
  for_each = local.existing_buckets
  bucket   = each.value
}
```

---

# 📕 SECTION 6: Check Blocks (Terraform 1.5+)

## What Are Check Blocks?

Check blocks let you define **custom assertions** about your infrastructure that are evaluated after every apply. Unlike `precondition`/`postcondition` (which stop the apply), check blocks produce **warnings** — the apply completes but you get notified of the violation.

```hcl
# ============================================
# CHECK BLOCK SYNTAX
# ============================================
check "<check_name>" {
  # Optional: data source to fetch real-world state for checking
  data "<type>" "<name>" {
    # same syntax as a regular data source
  }

  assert {
    condition     = <boolean expression>
    error_message = "Human-readable description of what failed"
  }
}
```

---

## Check Block Patterns

```hcl
# ============================================
# PATTERN 1: Validate infrastructure health after apply
# ============================================

# Check that a website is actually reachable after deploying
check "website_is_reachable" {
  data "http" "app_health" {
    url = "https://${aws_lb.main.dns_name}/health"
  }

  assert {
    condition     = data.http.app_health.status_code == 200
    error_message = "Website health check failed. Status: ${data.http.app_health.status_code}"
  }
}

# ============================================
# PATTERN 2: Validate security posture
# ============================================
check "s3_bucket_not_public" {
  data "aws_s3_bucket_acl" "app_data" {
    bucket = aws_s3_bucket.app_data.id
  }

  assert {
    condition     = data.aws_s3_bucket_acl.app_data.acl != "public-read"
    error_message = "S3 bucket ${aws_s3_bucket.app_data.id} must not be public!"
  }
}

# ============================================
# PATTERN 3: Validate configuration rules
# ============================================
check "rds_backup_enabled" {
  assert {
    condition     = aws_db_instance.main.backup_retention_period > 0
    error_message = "RDS instance must have backup retention enabled."
  }
}

check "production_has_multi_az" {
  assert {
    condition     = terraform.workspace != "prod" || aws_db_instance.main.multi_az == true
    error_message = "Production RDS must have Multi-AZ enabled for high availability."
  }
}

check "ssl_certificate_not_expiring" {
  data "aws_acm_certificate" "app" {
    domain   = var.domain_name
    statuses = ["ISSUED"]
  }

  assert {
    condition = timecmp(
      data.aws_acm_certificate.app.not_after,
      timeadd(timestamp(), "720h")  # 30 days
    ) > 0
    error_message = "SSL certificate expires within 30 days! Renew immediately."
  }
}

# ============================================
# PATTERN 4: Workspace and environment guards
# ============================================
check "valid_workspace" {
  assert {
    condition     = contains(["dev", "staging", "prod"], terraform.workspace)
    error_message = "Workspace must be dev, staging, or prod. Current: '${terraform.workspace}'"
  }
}

check "prod_has_monitoring" {
  assert {
    condition     = terraform.workspace != "prod" || var.enable_monitoring == true
    error_message = "Monitoring must be enabled in production. Set enable_monitoring=true."
  }
}
```

---

# 📕 SECTION 7: Custom Conditions — Preconditions & Postconditions

## Deep Dive

Unlike `check` blocks (warnings), preconditions and postconditions **halt the apply** if they fail. They are assertions tied directly to a specific resource or data source.

```hcl
# ============================================
# PRECONDITIONS — checked BEFORE creating/modifying
# ============================================

variable "instance_type" {
  type = string
}

variable "environment" {
  type = string
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      # Ensure production uses approved instance types
      condition = (
        var.environment != "prod" ||
        contains(["t3.medium","t3.large","t3.xlarge","m5.large","m5.xlarge"], var.instance_type)
      )
      error_message = "Production requires an approved instance type. Got: ${var.instance_type}"
    }

    precondition {
      # Ensure we're not using deprecated instance families
      condition     = !startswith(var.instance_type, "t1.") && !startswith(var.instance_type, "m1.")
      error_message = "t1.* and m1.* instances are deprecated. Use t3.* or m5.* instead."
    }

    postcondition {
      # Verify instance was actually assigned a public IP if we expected one
      condition     = self.associate_public_ip_address == false || self.public_ip != ""
      error_message = "Instance was configured for public IP but none was assigned."
    }

    postcondition {
      # Verify the instance is in the expected AZ
      condition     = contains(data.aws_availability_zones.available.names, self.availability_zone)
      error_message = "Instance was placed in an unexpected availability zone: ${self.availability_zone}"
    }
  }
}


# ============================================
# PRECONDITIONS ON DATA SOURCES
# ============================================

data "aws_ami" "golden_image" {
  most_recent = true
  owners      = [data.aws_caller_identity.current.account_id]

  filter {
    name   = "tag:Status"
    values = ["approved"]
  }

  lifecycle {
    postcondition {
      # Ensure the AMI we found is actually approved and recent
      condition     = self.tags["Status"] == "approved"
      error_message = "The selected AMI (${self.id}) is not approved for production use."
    }

    postcondition {
      # AMI should not be older than 90 days
      condition = timecmp(
        self.creation_date,
        timeadd(timestamp(), "-2160h")   # -90 days
      ) > 0
      error_message = "AMI ${self.id} is older than 90 days. Run the image pipeline."
    }
  }
}


# ============================================
# VARIABLE VALIDATION (simpler preconditions)
# ============================================

variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "The vpc_cidr must be a valid CIDR block like '10.0.0.0/16'."
  }

  validation {
    condition = (
      tonumber(split("/", var.vpc_cidr)[1]) >= 16 &&
      tonumber(split("/", var.vpc_cidr)[1]) <= 28
    )
    error_message = "VPC CIDR prefix must be between /16 and /28."
  }
}

variable "retention_days" {
  type = number
  validation {
    condition     = contains([1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365], var.retention_days)
    error_message = "Log retention must be one of: 1,3,5,7,14,30,60,90,120,150,180,365."
  }
}

variable "tags" {
  type = map(string)
  validation {
    condition     = contains(keys(var.tags), "Environment") && contains(keys(var.tags), "Project")
    error_message = "Tags must include at minimum 'Environment' and 'Project' keys."
  }
}
```

---

# 📕 SECTION 8: Terraform Testing (terraform test)

## What Is Terraform Testing?

Terraform 1.6+ introduced a native testing framework. It lets you write **automated tests for your Terraform modules** that actually apply real infrastructure, assert expected values, and then destroy everything — like integration testing for infrastructure.

```
tests/
├── unit.tftest.hcl         ← Tests that mock providers (fast, no infra)
└── integration.tftest.hcl  ← Tests that create real infra (slow, thorough)
```

---

## Writing Terraform Tests

```hcl
# ============================================
# tests/basic.tftest.hcl — Test your module
# ============================================

# Variables for this test run
variables {
  environment  = "test"
  project_name = "myapp-test"
  vpc_cidr     = "10.99.0.0/16"
}

# Run 1: Apply the module and check outputs
run "creates_vpc_with_correct_cidr" {
  command = apply   # actually creates infra (use "plan" for plan-only)

  assert {
    condition     = output.vpc_cidr == "10.99.0.0/16"
    error_message = "VPC CIDR doesn't match expected value"
  }

  assert {
    condition     = length(output.private_subnet_ids) == 3
    error_message = "Expected 3 private subnets, got ${length(output.private_subnet_ids)}"
  }

  assert {
    condition     = output.environment_tag == "test"
    error_message = "Environment tag was not set correctly"
  }
}

# Run 2: Apply again with different variables (tests idempotency)
run "update_instance_count" {
  variables {
    instance_count = 3   # Override just this variable
  }

  assert {
    condition     = output.instance_count == 3
    error_message = "Instance count update failed"
  }
}

# Run 3: Test destruction behavior
run "verify_outputs_before_destroy" {
  command = plan   # Just plan — verify expected plan output

  variables {
    enable_monitoring = false
  }

  assert {
    condition     = !output.monitoring_enabled
    error_message = "Monitoring should be disabled when enable_monitoring=false"
  }
}
```

```hcl
# ============================================
# tests/mock.tftest.hcl — Tests with mock providers
# (No real infrastructure created — very fast!)
# ============================================

mock_provider "aws" {
  # Mock the AWS provider — no real API calls
  mock_resource "aws_instance" {
    defaults = {
      id         = "i-mock1234567890"
      public_ip  = "1.2.3.4"
      private_ip = "10.0.1.10"
      arn        = "arn:aws:ec2:us-east-1:123456789:instance/i-mock1234567890"
    }
  }

  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-mock1234567890"
      cidr_block = "10.0.0.0/16"
    }
  }

  mock_data "aws_ami" {
    defaults = {
      id   = "ami-mock1234567890"
      name = "ubuntu-22.04-mock"
    }
  }
}

variables {
  environment  = "test"
  project_name = "mock-test"
}

run "unit_test_naming_convention" {
  command = plan

  assert {
    condition     = aws_instance.app.tags["Environment"] == "test"
    error_message = "Instance must be tagged with correct environment"
  }

  assert {
    condition     = startswith(aws_instance.app.tags["Name"], "mock-test-")
    error_message = "Instance name must start with project name prefix"
  }
}
```

```bash
# Run all tests
terraform test

# Run specific test file
terraform test -filter=tests/basic.tftest.hcl

# Run with verbose output
terraform test -verbose

# Run only plan-based tests (no infrastructure created)
terraform test -filter=tests/mock.tftest.hcl
```

---

# 📕 SECTION 9: Performance & Scale

## Large State File Optimization

```bash
# ============================================
# DIAGNOSING SLOW TERRAFORM OPERATIONS
# ============================================

# 1. Time your operations
time terraform plan

# 2. Enable trace logging to see what's slow
TF_LOG=TRACE terraform plan 2>&1 | grep -E "(Reading|Refreshing|seconds)"

# 3. Count resources in state
terraform state list | wc -l

# 4. Find the largest resources (JSON state inspection)
cat terraform.tfstate | python3 -c "
import json, sys
state = json.load(sys.stdin)
for r in state.get('resources', []):
    print(f\"{r['type']}.{r['name']}: {r['mode']}\")
" | sort | head -50


# ============================================
# STRATEGIES FOR LARGE STATES
# ============================================

# Strategy 1: Increase parallelism (default is 10)
terraform apply -parallelism=20   # More concurrent API calls
terraform apply -parallelism=5    # Fewer (for rate-limited APIs)

# Strategy 2: Skip refresh (use cached state)
terraform plan -refresh=false      # Don't re-read existing resources
terraform apply -refresh=false     # Faster but might miss drift

# Strategy 3: Target specific resources
terraform plan  -target=module.compute
terraform apply -target=module.database

# Strategy 4: Partial refresh
terraform apply -refresh-only -target=data.aws_ami.ubuntu

# Strategy 5: Split the state (the real solution for scale)
# Instead of ONE big state:
# monolith/terraform.tfstate (500 resources)

# Use MULTIPLE focused states:
# networking/terraform.tfstate  (50 resources)
# compute/terraform.tfstate     (100 resources)
# database/terraform.tfstate    (30 resources)
# monitoring/terraform.tfstate  (40 resources)
```

---

## State Manipulation Commands

```bash
# ============================================
# TERRAFORM STATE COMMANDS — Production Toolkit
# ============================================

# List all resources in state
terraform state list

# Show details of a specific resource
terraform state show aws_instance.web
terraform state show 'module.compute.aws_instance.web_server["web-01"]'

# Move a resource within state (rename without destroy)
terraform state mv aws_instance.old_name aws_instance.new_name
terraform state mv 'aws_instance.web[0]' 'aws_instance.web["web-01"]'

# Remove resource from state (stop managing without destroying)
terraform state rm aws_instance.web
# ⚠️ Resource still exists in AWS — Terraform just forgets about it
# Useful for: handing off to another team, manual management, etc.

# Import existing resource into state (CLI method)
terraform import aws_instance.web i-0a1b2c3d4e5f67890
terraform import 'aws_s3_bucket.data["logs"]' my-logs-bucket

# Pull current state to local file (for inspection)
terraform state pull > current-state.json

# Push modified state (DANGER — use with extreme caution)
terraform state push modified-state.json

# Taint a resource (force replacement on next apply)
# Deprecated in favor of:
terraform apply -replace=aws_instance.web

# Refresh — sync state with real-world
terraform refresh               # Updates state to match reality
terraform plan -refresh-only    # Shows what would change if you refresh
```

---

## The `terraform_data` Resource (Terraform 1.4+)

```hcl
# ============================================
# terraform_data — Replacement for null_resource
# ============================================

# Use case 1: Store arbitrary values in state (no provider needed)
resource "terraform_data" "app_version" {
  input = {
    version    = var.app_version
    build_id   = var.build_id
    deployed_at = timestamp()
  }
}

# Access it later
output "deployment_info" {
  value = terraform_data.app_version.output
}


# Use case 2: Trigger other resources to rebuild
resource "terraform_data" "force_replacement" {
  # When this changes, anything with replace_triggered_by on it will be replaced
  input = var.force_recreate_token   # e.g., a rotating string or timestamp
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [terraform_data.force_replacement]
  }
}


# Use case 3: Run scripts when things change (provisioners)
resource "terraform_data" "db_migration" {
  # Triggers when app version changes
  triggers_replace = [var.app_version]

  provisioner "local-exec" {
    command = "flyway -url=jdbc:postgresql://${aws_db_instance.main.endpoint}/mydb migrate"
  }
}
```

---

## Production Debugging Strategies

```bash
# ============================================
# COMPLETE DEBUGGING TOOLKIT
# ============================================

# 1. Understand what Terraform will do
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | {address: .address, action: .change.actions}'

# 2. Full verbose logging
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform-debug-$(date +%Y%m%d-%H%M%S).log"
terraform plan

# 3. Interactive expression testing
terraform console
> cidrsubnet("10.0.0.0/16", 8, 5)
"10.0.5.0/24"
> { for k, v in var.servers : k => v.instance_type }
{ "web" = "t2.micro", "api" = "t2.small" }
> length(data.aws_availability_zones.available.names)
3

# 4. Validate configuration without deploying
terraform validate            # Syntax and config validation
terraform fmt -check          # Check formatting (CI-friendly)
terraform fmt -recursive      # Fix formatting in all files

# 5. Visualize the dependency graph
terraform graph | dot -Tpng > dependency-graph.png
terraform graph -type=plan | dot -Tsvg > plan-graph.svg

# 6. Lock state inspection and force-unlock
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID":{"S":"my-bucket/path/to/terraform.tfstate"}}'

terraform force-unlock <LOCK_ID>   # Emergency only!

# 7. Drift detection
terraform plan -refresh-only       # Shows only real-world drift
terraform apply -refresh-only      # Sync state to match real world

# 8. Provider-level debugging
export AWS_SDK_LOAD_CONFIG=1
export TF_LOG_PROVIDER=DEBUG
terraform plan 2>&1 | grep -i "request\|response\|error"
```

---

# 🏗️ REAL-WORLD PRODUCTION ARCHITECTURE

## Complete Advanced Pattern: Self-Validating, Testable Infrastructure Module

```hcl
# ============================================
# modules/web-application/main.tf
# A production-grade module using all advanced concepts
# ============================================

# --- variables.tf ---
variable "project_name" {
  type        = string
  description = "Project name — used in all resource names and tags"

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{2,28}[a-z0-9]$", var.project_name))
    error_message = "Project name must be 4-30 chars, lowercase alphanumeric and hyphens, starting with letter."
  }
}

variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "vpc_id" {
  type        = string
  description = "VPC ID to deploy into"

  validation {
    condition     = can(regex("^vpc-[a-f0-9]{8,17}$", var.vpc_id))
    error_message = "vpc_id must be a valid VPC ID like 'vpc-0a1b2c3d4e5f67890'."
  }
}

variable "subnet_ids" {
  type        = list(string)
  description = "List of subnet IDs for the ASG"

  validation {
    condition     = length(var.subnet_ids) >= 2
    error_message = "At least 2 subnets are required for high availability."
  }
}

variable "ingress_rules" {
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { description="HTTP",  from_port=80,  to_port=80,  protocol="tcp", cidr_blocks=["0.0.0.0/0"] },
    { description="HTTPS", from_port=443, to_port=443, protocol="tcp", cidr_blocks=["0.0.0.0/0"] },
  ]
}

variable "scaling_config" {
  type = object({
    min_size         = number
    max_size         = number
    desired_capacity = number
  })

  validation {
    condition = (
      var.scaling_config.min_size >= 1 &&
      var.scaling_config.max_size >= var.scaling_config.min_size &&
      var.scaling_config.desired_capacity >= var.scaling_config.min_size &&
      var.scaling_config.desired_capacity <= var.scaling_config.max_size
    )
    error_message = "Invalid scaling config: min<=desired<=max, all >= 1."
  }
}


# --- locals.tf ---
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  is_prod     = var.environment == "prod"

  instance_type_map = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
  instance_type = local.instance_type_map[var.environment]

  # Build ingress rules map for stable for_each
  ingress_map = {
    for rule in var.ingress_rules :
    "${rule.protocol}-${rule.from_port}" => rule
  }

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "web-application"
  }
}


# --- data.tf ---
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  lifecycle {
    postcondition {
      condition     = self.state == "available"
      error_message = "Selected AMI ${self.id} is not in 'available' state."
    }
  }
}

data "aws_vpc" "selected" {
  id = var.vpc_id

  lifecycle {
    postcondition {
      condition     = self.state == "available"
      error_message = "VPC ${var.vpc_id} is not available."
    }
  }
}


# --- security.tf ---
resource "aws_security_group" "app" {
  name_prefix = "${local.name_prefix}-app-"
  vpc_id      = var.vpc_id
  description = "Security group for ${local.name_prefix} application"

  dynamic "ingress" {
    for_each = local.ingress_map
    iterator = rule

    content {
      description = rule.value.description
      from_port   = rule.value.from_port
      to_port     = rule.value.to_port
      protocol    = rule.value.protocol
      cidr_blocks = rule.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  lifecycle {
    create_before_destroy = true

    precondition {
      condition     = data.aws_vpc.selected.state == "available"
      error_message = "Cannot create security group: VPC ${var.vpc_id} is not available."
    }
  }

  tags = merge(local.common_tags, { Name = "${local.name_prefix}-app-sg" })
}


# --- compute.tf ---
resource "aws_launch_template" "app" {
  name_prefix   = "${local.name_prefix}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = local.instance_type

  vpc_security_group_ids = [aws_security_group.app.id]

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  monitoring {
    enabled = local.is_prod
  }

  user_data = base64encode(templatefile("${path.module}/templates/user-data.sh.tpl", {
    project     = var.project_name
    environment = var.environment
  }))

  lifecycle {
    create_before_destroy = true

    postcondition {
      condition     = self.image_id == data.aws_ami.ubuntu.id
      error_message = "Launch template AMI doesn't match expected AMI."
    }
  }

  tags = local.common_tags
}

resource "aws_autoscaling_group" "app" {
  name                = "${local.name_prefix}-asg"
  vpc_zone_identifier = var.subnet_ids
  min_size            = var.scaling_config.min_size
  max_size            = var.scaling_config.max_size
  desired_capacity    = var.scaling_config.desired_capacity

  health_check_type         = "ELB"
  health_check_grace_period = 300

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes        = [desired_capacity]
    create_before_destroy = true
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


# --- checks.tf ---
check "asg_has_healthy_instances" {
  data "aws_autoscaling_group" "verify" {
    name = aws_autoscaling_group.app.name
  }

  assert {
    condition     = data.aws_autoscaling_group.verify.desired_capacity > 0
    error_message = "ASG has 0 desired instances — check the launch template."
  }
}

check "security_group_not_wide_open" {
  assert {
    condition = !anytrue([
      for rule in var.ingress_rules :
      rule.from_port == 22 && contains(rule.cidr_blocks, "0.0.0.0/0")
    ])
    error_message = "SSH (port 22) must not be open to 0.0.0.0/0. Restrict to internal CIDRs."
  }
}


# --- outputs.tf ---
output "asg_name" {
  description = "Name of the Auto Scaling Group"
  value       = aws_autoscaling_group.app.name
}

output "security_group_id" {
  description = "ID of the application security group"
  value       = aws_security_group.app.id
}

output "launch_template_id" {
  description = "ID of the launch template"
  value       = aws_launch_template.app.id
}

output "ami_id" {
  description = "AMI used for instances"
  value       = data.aws_ami.ubuntu.id
}

output "instance_type" {
  description = "Instance type used for this environment"
  value       = local.instance_type
}
```

```hcl
# ============================================
# tests/web_application.tftest.hcl
# ============================================

mock_provider "aws" {
  mock_data "aws_ami" {
    defaults = {
      id    = "ami-0mock123456789"
      state = "available"
    }
  }

  mock_data "aws_vpc" {
    defaults = {
      id    = "vpc-0mock123456789"
      state = "available"
    }
  }

  mock_resource "aws_security_group" {
    defaults = {
      id  = "sg-0mock123456789"
      arn = "arn:aws:ec2:us-east-1:123456789:security-group/sg-0mock123"
    }
  }

  mock_resource "aws_launch_template" {
    defaults = {
      id       = "lt-0mock123456789"
      image_id = "ami-0mock123456789"
    }
  }

  mock_resource "aws_autoscaling_group" {
    defaults = {
      desired_capacity = 2
    }
  }
}

# Test 1: Valid dev configuration
run "dev_environment_uses_small_instances" {
  command = plan

  variables {
    project_name = "myapp"
    environment  = "dev"
    vpc_id       = "vpc-0mock123456789"
    subnet_ids   = ["subnet-mock1", "subnet-mock2"]
    scaling_config = {
      min_size         = 1
      max_size         = 3
      desired_capacity = 1
    }
  }

  assert {
    condition     = output.instance_type == "t3.micro"
    error_message = "Dev should use t3.micro"
  }
}

# Test 2: Production validation
run "prod_environment_uses_larger_instances" {
  command = plan

  variables {
    project_name = "myapp"
    environment  = "prod"
    vpc_id       = "vpc-0mock123456789"
    subnet_ids   = ["subnet-mock1", "subnet-mock2", "subnet-mock3"]
    scaling_config = {
      min_size         = 3
      max_size         = 10
      desired_capacity = 3
    }
  }

  assert {
    condition     = output.instance_type == "t3.medium"
    error_message = "Prod should use t3.medium"
  }
}

# Test 3: Validation catches bad input
run "rejects_invalid_scaling_config" {
  command = plan

  variables {
    project_name = "myapp"
    environment  = "dev"
    vpc_id       = "vpc-0mock123456789"
    subnet_ids   = ["subnet-mock1", "subnet-mock2"]
    scaling_config = {
      min_size         = 5
      max_size         = 3   # max < min — should fail!
      desired_capacity = 1
    }
  }

  expect_failures = [var.scaling_config]   # This test EXPECTS a failure
}

# Test 4: Rejects invalid environment
run "rejects_invalid_environment" {
  command = plan

  variables {
    project_name = "myapp"
    environment  = "production"   # Invalid — should be "prod"
    vpc_id       = "vpc-0mock123456789"
    subnet_ids   = ["subnet-mock1", "subnet-mock2"]
    scaling_config = { min_size=1, max_size=3, desired_capacity=1 }
  }

  expect_failures = [var.environment]
}
```

---

# 🧪 HANDS-ON LAB

## Lab: Advanced HCL Processor

**Objective:** Build a system using ALL the advanced concepts from this guide — functions, for expressions, dynamic blocks, conditions, validations, and check blocks — using only the local provider.

**Requirements:**
1. Accept a list of server objects with name, role, tier, and active status
2. Use **for expressions** to filter and transform the data
3. Use **dynamic blocks** to generate config sections
4. Use **custom validation** to enforce naming rules and tier values
5. Use **check blocks** to validate the output
6. Use **10+ different functions** across the implementation
7. Generate a structured JSON report and a formatted text summary

**Try it yourself first!**

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
# variables.tf
# ============================================
variable "project_name" {
  type = string
  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,20}$", var.project_name))
    error_message = "Project name must be lowercase alphanumeric/hyphens, 2-21 chars."
  }
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "servers" {
  type = list(object({
    name   = string
    role   = string
    tier   = string
    active = bool
    tags   = optional(map(string), {})
  }))

  validation {
    condition = alltrue([
      for s in var.servers : can(regex("^[a-z][a-z0-9-]+$", s.name))
    ])
    error_message = "All server names must be lowercase alphanumeric and hyphens."
  }

  validation {
    condition = alltrue([
      for s in var.servers : contains(["web", "api", "worker", "db", "cache"], s.role)
    ])
    error_message = "Server role must be one of: web, api, worker, db, cache."
  }

  validation {
    condition = alltrue([
      for s in var.servers : contains(["small", "medium", "large"], s.tier)
    ])
    error_message = "Server tier must be: small, medium, or large."
  }
}

# ============================================
# locals.tf
# ============================================
locals {
  env         = var.environment
  is_prod     = local.env == "prod"
  name_prefix = "${var.project_name}-${local.env}"

  # ---- FUNCTION: size mapping ----
  tier_to_type = {
    small  = "t2.micro"
    medium = "t2.small"
    large  = "t3.medium"
  }

  # ---- FOR EXPRESSION: filter active servers ----
  active_servers = [for s in var.servers : s if s.active]

  # ---- FOR EXPRESSION: group by role ----
  servers_by_role = {
    for s in local.active_servers :
    s.role => s.name...
  }

  # ---- FOR EXPRESSION: build enriched server map ----
  server_map = {
    for s in var.servers :
    s.name => merge(s, {
      instance_type = local.tier_to_type[s.tier]
      display_name  = "${local.name_prefix}-${s.name}"
      status        = s.active ? "active" : "standby"
      merged_tags   = merge(s.tags, {
        Name        = "${local.name_prefix}-${s.name}"
        Role        = s.role
        Environment = local.env
        ManagedBy   = "terraform"
      })
    })
  }

  # ---- FUNCTIONS: aggregate statistics ----
  total_servers      = length(var.servers)
  active_count       = length(local.active_servers)
  standby_count      = local.total_servers - local.active_count
  unique_roles       = toset([for s in var.servers : s.role])
  unique_tiers       = toset([for s in var.servers : s.tier])
  active_percentage  = local.total_servers > 0 ? floor((local.active_count / local.total_servers) * 100) : 0

  # ---- FUNCTIONS: compute costs (mock) ----
  tier_cost = { small = 10, medium = 20, large = 40 }
  estimated_monthly_cost = sum([
    for s in local.active_servers :
    local.tier_cost[s.tier]
  ])

  # ---- FUNCTIONS: role distribution summary ----
  role_summary = {
    for role in local.unique_roles :
    role => {
      total   = length([for s in var.servers : s if s.role == role])
      active  = length([for s in local.active_servers : s if s.role == role])
      servers = try(local.servers_by_role[role], [])
    }
  }

  # ---- FUNCTIONS: build config sections ----
  config_sections = {
    for role in local.unique_roles :
    role => {
      servers  = try(local.servers_by_role[role], [])
      replicas = length(try(local.servers_by_role[role], []))
      enabled  = length(try(local.servers_by_role[role], [])) > 0
    }
  }

  # ---- FUNCTIONS: generate deployment manifest ----
  deployment_manifest = {
    metadata = {
      project     = var.project_name
      environment = local.env
      generated   = formatdate("YYYY-MM-DD", timestamp())
      prefix      = local.name_prefix
    }
    statistics = {
      total_servers          = local.total_servers
      active_servers         = local.active_count
      standby_servers        = local.standby_count
      active_percentage      = "${local.active_percentage}%"
      estimated_monthly_cost = "$${local.estimated_monthly_cost}"
      unique_roles           = sort(tolist(local.unique_roles))
      unique_tiers           = sort(tolist(local.unique_tiers))
    }
    servers    = local.server_map
    by_role    = local.role_summary
    config     = local.config_sections
  }

  # ---- FUNCTIONS: text summary ----
  summary_lines = concat(
    [
      "=" * 60,
      "  DEPLOYMENT SUMMARY: ${upper(var.project_name)} / ${upper(local.env)}",
      "=" * 60,
      "",
      "STATISTICS",
      "-" * 40,
      format("  %-25s %d", "Total Servers:",    local.total_servers),
      format("  %-25s %d (%d%%)", "Active:",    local.active_count, local.active_percentage),
      format("  %-25s %d", "Standby:",          local.standby_count),
      format("  %-25s $%d/month", "Est. Cost:", local.estimated_monthly_cost),
      "",
      "ROLE DISTRIBUTION",
      "-" * 40,
    ],
    flatten([
      for role in sort(tolist(local.unique_roles)) : [
        format("  %-12s %d active / %d total",
          "${role}:",
          local.role_summary[role].active,
          local.role_summary[role].total
        ),
        format("  %-12s [%s]",
          "",
          join(", ", local.role_summary[role].servers)
        ),
      ]
    ]),
    [
      "",
      "SERVER DETAILS",
      "-" * 40,
    ],
    [
      for name, s in local.server_map :
      format("  %-20s %-8s %-8s %-10s %s",
        name, s.role, s.tier, s.instance_type, s.status
      )
    ],
    ["", "=" * 60]
  )
}

# ============================================
# resources.tf
# ============================================

# Resource 1: JSON manifest
resource "local_file" "deployment_manifest" {
  filename        = "./output/${local.name_prefix}-manifest.json"
  content         = jsonencode(local.deployment_manifest)
  file_permission = "0644"
}

# Resource 2: Text summary
resource "local_file" "summary" {
  filename        = "./output/${local.name_prefix}-summary.txt"
  content         = join("\n", local.summary_lines)
  file_permission = "0644"
}

# Resource 3: Individual server configs using for_each + dynamic approach
resource "local_file" "server_configs" {
  for_each = local.server_map

  filename = "./output/servers/${each.key}.conf"
  content  = <<-EOT
    # Server Configuration: ${each.key}
    # Generated: ${formatdate("YYYY-MM-DD hh:mm:ss", timestamp())}

    [identity]
    name          = ${each.value.display_name}
    role          = ${each.value.role}
    status        = ${each.value.status}

    [compute]
    tier          = ${each.value.tier}
    instance_type = ${each.value.instance_type}

    [tags]
    %{ for key, val in each.value.merged_tags ~}
    ${key} = ${val}
    %{ endfor ~}
  EOT
}

# Resource 4: Role-level config (only for active roles)
resource "local_file" "role_configs" {
  for_each = { for role, cfg in local.config_sections : role => cfg if cfg.enabled }

  filename = "./output/roles/${each.key}.conf"
  content  = <<-EOT
    [${each.key}]
    replicas = ${each.value.replicas}
    servers  = ${join(",", each.value.servers)}
    enabled  = ${each.value.enabled}
  EOT
}

# ============================================
# checks.tf
# ============================================

check "has_active_servers" {
  assert {
    condition     = local.active_count > 0
    error_message = "No active servers found. At least one server must have active=true."
  }
}

check "production_has_web_servers" {
  assert {
    condition     = !local.is_prod || contains(tolist(local.unique_roles), "web")
    error_message = "Production deployments must include at least one web server."
  }
}

check "cost_within_budget" {
  assert {
    condition     = local.estimated_monthly_cost <= (local.is_prod ? 1000 : 200)
    error_message = "Estimated cost $${local.estimated_monthly_cost}/month exceeds ${local.is_prod ? "prod ($1000)" : "non-prod ($200)"} budget."
  }
}

check "no_standby_in_prod" {
  assert {
    condition     = !local.is_prod || local.standby_count == 0
    error_message = "Production should not have standby servers. Activate or remove them."
  }
}

# ============================================
# outputs.tf
# ============================================

output "deployment_summary" {
  value = {
    project              = var.project_name
    environment          = local.env
    total_servers        = local.total_servers
    active_servers       = local.active_count
    active_percentage    = "${local.active_percentage}%"
    unique_roles         = sort(tolist(local.unique_roles))
    estimated_cost       = "$${local.estimated_monthly_cost}/month"
    files_generated      = concat(
      [local_file.deployment_manifest.filename, local_file.summary.filename],
      [for f in local_file.server_configs : f.filename],
      [for f in local_file.role_configs   : f.filename]
    )
  }
}

output "servers_by_role" {
  value = local.role_summary
}
```

```hcl
# terraform.tfvars
project_name = "myplatform"
environment  = "prod"

servers = [
  { name="web-01",    role="web",    tier="medium", active=true,  tags={ "team"="frontend" } },
  { name="web-02",    role="web",    tier="medium", active=true,  tags={ "team"="frontend" } },
  { name="api-01",    role="api",    tier="large",  active=true,  tags={ "team"="backend"  } },
  { name="api-02",    role="api",    tier="large",  active=true,  tags={ "team"="backend"  } },
  { name="worker-01", role="worker", tier="small",  active=true,  tags={} },
  { name="worker-02", role="worker", tier="small",  active=false, tags={} },
  { name="db-01",     role="db",     tier="large",  active=true,  tags={ "team"="data"     } },
  { name="cache-01",  role="cache",  tier="small",  active=true,  tags={} },
]
```

```bash
# Run the lab
mkdir -p output/servers output/roles

terraform init
terraform validate
terraform plan
terraform apply -auto-approve

# Explore outputs
cat output/myplatform-prod-summary.txt
cat output/myplatform-prod-manifest.json | python3 -m json.tool
ls output/servers/
cat output/servers/web-01.conf
ls output/roles/
cat output/roles/api.conf

# Test validation
terraform apply -var='environment=invalid' -auto-approve
# ❌ Error: Environment must be dev, staging, or prod.

# Test check block
terraform apply -var='servers=[{"name":"db-01","role":"db","tier":"large","active":true}]' -auto-approve
# ⚠️ Warning: Production should not have standby servers (none here)
# ⚠️ Warning: Production requires web server

# Test in dev (different budget check)
terraform apply -var='environment=dev' -auto-approve
```

---

# 📋 SUMMARY CHEAT SHEET

## Functions Quick Reference

```hcl
# STRINGS
upper(str)          lower(str)         title(str)
trimspace(str)      trim(str, chars)   trimprefix(str, prefix)
join(sep, list)     split(sep, str)    replace(str, old, new)
format(fmt, args)   substr(str, o, l)  strcontains(str, sub)
base64encode(str)   base64decode(str)  urlencode(str)

# COLLECTIONS
length(coll)        element(list, idx) slice(list, s, e)
contains(list, val) index(list, val)   lookup(map, key, def)
keys(map)           values(map)        merge(maps...)
flatten(list)       compact(list)      sort(list)       reverse(list)
toset(list)         tolist(set)        setunion(a,b)    setintersection(a,b)
setsubtract(a,b)    setproduct(a,b)    range(n)         range(s,e,step)

# NUMERIC
max(nums...)   min(nums...)   abs(n)   ceil(n)   floor(n)   pow(b,e)

# NETWORK
cidrsubnet(cidr, newbits, netnum)
cidrhost(cidr, hostnum)
cidrnetmask(cidr)

# ENCODING / HASHING
jsonencode(val)     jsondecode(str)     yamlencode(val)
md5(str)            sha256(str)         filemd5(path)
base64encode(str)   base64decode(str)

# FILESYSTEM
file(path)          fileexists(path)    filebase64(path)
templatefile(path, vars)                filemd5(path)

# TYPE / LOGIC
try(expr, fallback)  can(expr)          tobool(v)  tonumber(v)  tostring(v)
coalesce(vals...)    coalescelist(lists...)

# DATE/TIME
timestamp()              timeadd(ts, dur)
formatdate(fmt, ts)      timecmp(ts1, ts2)
```

---

## Advanced Syntax Quick Reference

```hcl
# FOR EXPRESSIONS
[for item in list : expression]
[for item in list : expression if condition]
{for item in list : item.key => item.value}
{for k, v in map  : k => transform(v)}
{for item in list : item.key => item.value...}  # Group by (...)

# DYNAMIC BLOCKS
dynamic "block_name" {
  for_each = collection
  iterator = custom_name      # optional
  content {
    attr = custom_name.value.field
  }
}

# MOVED BLOCKS
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}

# IMPORT BLOCKS (1.5+)
import {
  id = "real-world-resource-id"
  to = aws_resource_type.name
}

# CHECK BLOCKS (1.5+)
check "name" {
  data "type" "name" { ... }   # optional
  assert {
    condition     = <bool>
    error_message = "message"
  }
}

# PRECONDITIONS / POSTCONDITIONS
lifecycle {
  precondition {
    condition     = <bool>
    error_message = "message"
  }
  postcondition {
    condition     = self.attribute != null
    error_message = "message"
  }
}

# VARIABLE VALIDATION
variable "name" {
  validation {
    condition     = <bool using var.name>
    error_message = "message"
  }
}

# TERRAFORM TEST (1.6+)
# tests/my_test.tftest.hcl
mock_provider "aws" { ... }
variables { env = "test" }
run "test_name" {
  command = plan  # or apply
  assert {
    condition     = output.value == "expected"
    error_message = "message"
  }
  expect_failures = [var.bad_var]  # test that something FAILS
}
```

---

## Top Interview Questions

**Q1: What is the difference between `check` blocks, `precondition`, and `postcondition`?**
> All three validate infrastructure, but they behave differently. `precondition` runs before a resource is created and aborts the apply if it fails — it's a pre-flight check. `postcondition` runs after creation using `self.*` attributes and aborts if it fails. `check` blocks run after the full apply and produce warnings rather than errors — great for health checks and soft compliance rules. Use pre/postcondition for hard requirements, check blocks for informational assertions.

**Q2: What does a `moved` block do and when would you use it?**
> A `moved` block tells Terraform that a resource has been renamed or reorganized in state without destroying and recreating it. Use it when renaming resources, refactoring from `count` to `for_each`, moving resources into or between modules, or splitting modules. Without a moved block, Terraform would destroy the old resource and create a new one — potentially causing production outages.

**Q3: What is the difference between `terraform import` (CLI) and `import` blocks?**
> CLI `terraform import` is imperative — you run it manually, it's not version-controlled, and it doesn't show up in code reviews. Import blocks (Terraform 1.5+) are declarative — written in `.tf` files, visible in PRs, shown in `terraform plan`, and reproducible. Import blocks also support `-generate-config-out` which auto-generates the resource configuration from the real-world resource.

**Q4: Explain the `try()` function and when you'd use it.**
> `try()` evaluates a list of expressions and returns the value of the first one that doesn't produce an error. It's used for safe attribute access on objects that might not have a key, safe type conversions, graceful fallbacks when external data might be missing, and optional nested configuration. `can()` is the boolean companion — it returns true/false whether an expression would succeed.

**Q5: How do you test Terraform infrastructure code?**
> Terraform 1.6+ has a native testing framework using `.tftest.hcl` files. Tests can use mock providers (fast, no real infra) or real providers (slower, creates actual infrastructure). Each `run` block applies or plans the module with specific variables and asserts expected outputs. `expect_failures` lets you test that validation rules correctly reject bad input. Tests run with `terraform test`.

**Q6: How would you handle importing 50 existing AWS resources into Terraform?**
> Use import blocks (1.5+) with `for_each` if they're similar resources, or individual import blocks for different types. Run `terraform plan -generate-config-out=generated.tf` to let Terraform generate the resource configurations automatically. Review and clean the generated config, then apply. For very large imports, batch them in groups and use `-target` to control what's imported in each run.

**Q7: What is `terraform_data` and what replaced it?**
> `terraform_data` (Terraform 1.4+) is a replacement for the `null_resource` from the null provider. It stores arbitrary values in state with `input`/`output`, triggers other resources through `replace_triggered_by`, and runs provisioners when `triggers_replace` changes. Unlike `null_resource`, it requires no external provider and supports the `replace_triggered_by` lifecycle argument properly.

---
