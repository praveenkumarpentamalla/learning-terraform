# 🔐 TOPIC 8: Security & Compliance — Vault, Sentinel & OPA

---

## 🟢 BEGINNER LEVEL

### What is Security & Compliance in Terraform? (Simple Terms)

Imagine you're running a **hospital**. Doctors can prescribe medicine, but there are strict rules:
- Only licensed doctors can prescribe controlled substances
- Every prescription must be logged and audited
- Certain dangerous drugs require supervisor approval
- Patient data must be encrypted at all times
- Nobody can access the medication vault without proper authorization

**Terraform Security & Compliance is the same concept for infrastructure:**

```
Without Security Controls          With Security Controls
─────────────────────────          ──────────────────────────────
Any dev can create open S3         Policy blocks public S3 buckets
Passwords hardcoded in .tf files   Vault provides secrets dynamically
No audit trail of changes          Every change logged and attributed
Prod credentials on laptops        Short-lived credentials via Vault
Anyone can deploy anything          Only approved resource types allowed
```

---

### The Three Pillars of Terraform Security

```
┌─────────────────────────────────────────────────────────────────┐
│                 TERRAFORM SECURITY PILLARS                      │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────┐   │
│  │  HashiCorp    │  │   Sentinel    │  │       OPA         │   │
│  │    Vault      │  │               │  │  (Open Policy     │   │
│  │               │  │               │  │    Agent)         │   │
│  │ SECRETS       │  │  POLICY       │  │  POLICY           │   │
│  │ MANAGEMENT    │  │  ENFORCEMENT  │  │  AS CODE          │   │
│  │               │  │  (TF Cloud/   │  │  (Open Source)    │   │
│  │ "What secrets │  │   Enterprise) │  │                   │   │
│  │  can you use?"│  │ "What infra   │  │ "What infra       │   │
│  │               │  │  is allowed?" │  │  is allowed?"     │   │
│  └───────────────┘  └───────────────┘  └───────────────────┘   │
│                                                                 │
│       ↕                    ↕                    ↕               │
│  Dynamic secrets       Pre-apply gates      Pre/post gates      │
│  Lease management      TFC/TFE native       Any CI/CD system    │
│  Audit logging         Rego-compatible      Conftest tool       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Real-World Analogy

```
VAULT  =  Bank's Secure Vault Room
          - Only authorized people get access
          - Tracks who accessed what and when
          - Access expires after a set time
          - You get a temporary key, not a permanent one

SENTINEL =  Bank's Internal Policy Engine
            - Corporate rule: "Loans above $50k need director approval"
            - Checked BEFORE any transaction is processed
            - Cannot be bypassed even by senior staff
            - Policies written by compliance team

OPA     =  Bank's External Compliance Auditor
           - Independent of the bank's systems
           - Can audit any system (not just the bank's)
           - Writes reports: pass/warn/deny
           - Open standard, works with any platform
```

---

### Why These Tools Matter

```bash
# WITHOUT these tools — real risks:

# Risk 1: Hardcoded secrets in code (ends up in Git history!)
resource "aws_db_instance" "main" {
  password = "MySecret123!"    # ← LEAKED! Anyone with repo access sees this
}

# Risk 2: Overly permissive infrastructure
resource "aws_s3_bucket" "data" {
  acl = "public-read"          # ← Customer data exposed to internet!
}

# Risk 3: Non-compliant resources
resource "aws_instance" "web" {
  instance_type = "m5.24xlarge"  # ← $3,000/month! No budget check
}

# Risk 4: No audit trail
# Who created this? When? Why? What changed? Nobody knows.
```

---

### Basic Concepts at a Glance

```
VAULT:    Dynamic secrets engine
          terraform plan → asks Vault for DB credentials
          Vault issues temp credentials (expire in 1hr)
          Next plan → new temp credentials

SENTINEL: Policy-as-code (HashiCorp native)
          terraform plan finishes →
          Sentinel evaluates plan against policies →
          PASS: apply proceeds
          FAIL: apply blocked, error shown

OPA:      Policy-as-code (open source, cloud native)
          Works with: Terraform, Kubernetes, APIs, microservices
          Conftest tool makes it easy to use with Terraform
          terraform show -json tfplan → conftest test → PASS/FAIL
```

---

### Common Beginner Mistakes

```hcl
# ❌ MISTAKE 1: Using Vault provider without lease management
provider "vault" {
  address = "https://vault.mycompany.com"
  token   = "hvs.hardcoded_token"    # Never hardcode Vault tokens!
}
# ✅ Use environment variables or auth methods (AppRole, AWS, etc.)

# ❌ MISTAKE 2: Not understanding Sentinel is TFC/TFE only
# Sentinel runs ONLY in Terraform Cloud or Terraform Enterprise
# For open-source Terraform: use OPA/Conftest instead

# ❌ MISTAKE 3: Writing OPA policies that are too strict
# Block on every tiny issue → developers bypass by not using Terraform
# ✅ Use soft-mandatory (warn) for most rules, hard-mandatory for critical

# ❌ MISTAKE 4: Not rotating Vault dynamic credentials
# Dynamic credentials have a TTL (time-to-live)
# Terraform apply might take 45 min → credentials expire → mid-apply failure
# ✅ Set credential TTL longer than your longest apply

# ❌ MISTAKE 5: Storing Vault token in terraform.tfvars
vault_token = "hvs.CAESIDdGkH..."    # In terraform.tfvars → ends up in Git!
# ✅ Use: export VAULT_TOKEN=$(vault login -method=aws -field=token)
```

---

## 🟡 INTERMEDIATE LEVEL

### HashiCorp Vault — Complete Integration

**Vault Architecture with Terraform:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    VAULT + TERRAFORM FLOW                       │
│                                                                 │
│  Terraform          Vault Server         AWS/DB/Cloud           │
│  ─────────          ────────────         ──────────────         │
│                                                                 │
│  init          ───► Authenticate ────►  Verify identity        │
│  (AppRole/         (Issue token)                                │
│   AWS auth)    ◄─── Token returned                              │
│                                                                 │
│  plan/apply    ───► Request secret ──►  Check policy           │
│                ◄─── Temp credentials ◄── Create temp user       │
│                                         (e.g., DB user)        │
│  Creates DB    ───────────────────────────────────────────────► │
│  with temp creds                        DB created              │
│                                                                 │
│  [1 hour later]    ◄── Lease expires    Temp user deleted       │
│                                         automatically           │
└─────────────────────────────────────────────────────────────────┘
```

**Setting Up Vault Provider:**

```hcl
# providers.tf
terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 3.20"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# ── Method 1: Environment Variables (simplest, for local dev) ────
# export VAULT_ADDR="https://vault.mycompany.com"
# export VAULT_TOKEN="hvs.CAESID..."
provider "vault" {}    # Reads VAULT_ADDR and VAULT_TOKEN automatically


# ── Method 2: AppRole (for CI/CD pipelines) ──────────────────────
provider "vault" {
  address = "https://vault.mycompany.com"

  auth_login {
    path = "auth/approle/login"

    parameters = {
      role_id   = var.vault_role_id     # From CI/CD secret
      secret_id = var.vault_secret_id   # From CI/CD secret (wrapped)
    }
  }
}


# ── Method 3: AWS Auth (for resources running IN AWS) ────────────
# EC2, Lambda, ECS, etc. can auth to Vault using their IAM identity
provider "vault" {
  address = "https://vault.mycompany.com"

  auth_login_aws {
    role = "terraform-role"    # Vault role that maps to AWS IAM role
    # Vault verifies with AWS STS - no static credentials needed!
  }
}


# ── Method 4: Kubernetes Auth (for K8s workloads) ────────────────
provider "vault" {
  address = "https://vault.mycompany.com"

  auth_login_kubernetes {
    role = "terraform-role"
    jwt  = file("/var/run/secrets/kubernetes.io/serviceaccount/token")
  }
}
```

---

**Vault Secret Engines:**

```hcl
# ── 1. KV (Key-Value) Secrets Engine — Static Secrets ────────────

# Read a static secret from Vault KV v2
data "vault_kv_secret_v2" "app_config" {
  mount = "secret"                    # The KV mount path
  name  = "myapp/production/config"   # Secret path within mount
}

locals {
  api_key      = data.vault_kv_secret_v2.app_config.data["api_key"]
  db_password  = data.vault_kv_secret_v2.app_config.data["db_password"]
  stripe_key   = data.vault_kv_secret_v2.app_config.data["stripe_secret_key"]
}

# Use in resource
resource "aws_ssm_parameter" "api_key" {
  name  = "/myapp/prod/api_key"
  type  = "SecureString"
  value = local.api_key
}


# ── 2. AWS Secrets Engine — Dynamic AWS Credentials ──────────────

# First: Configure Vault AWS engine (usually done by Vault admin)
resource "vault_aws_secret_backend" "aws" {
  path = "aws"

  access_key = var.vault_aws_access_key    # Master credentials for Vault
  secret_key = var.vault_aws_secret_key
  region     = "us-east-1"
}

resource "vault_aws_secret_backend_role" "terraform_role" {
  backend         = vault_aws_secret_backend.aws.path
  name            = "terraform-deploy"
  credential_type = "iam_user"

  policy_document = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:*", "ec2:*", "rds:*"]
      Resource = "*"
    }]
  })
}

# Now Terraform can request dynamic AWS credentials
data "vault_aws_access_credentials" "deploy" {
  backend = "aws"
  role    = "terraform-deploy"
  type    = "iam_user"
}

# Configure AWS provider with dynamic Vault credentials!
provider "aws" {
  access_key = data.vault_aws_access_credentials.deploy.access_key
  secret_key = data.vault_aws_access_credentials.deploy.secret_key
  token      = data.vault_aws_access_credentials.deploy.security_token
  region     = "us-east-1"
}


# ── 3. Database Secrets Engine — Dynamic DB Credentials ──────────

# Configure Vault database engine (done by Vault admin)
resource "vault_database_secret_backend_connection" "postgres" {
  backend       = "database"
  name          = "prod-postgres"
  allowed_roles = ["app-role", "readonly-role", "terraform-role"]

  postgresql {
    connection_url = "postgresql://{{username}}:{{password}}@postgres.mycompany.com:5432/appdb"
    # Note: {{username}} and {{password}} are Vault template variables
    # Vault uses these to create and rotate credentials
  }
}

resource "vault_database_secret_backend_role" "app_role" {
  backend             = "database"
  name                = "app-role"
  db_name             = vault_database_secret_backend_connection.postgres.name
  creation_statements = [
    "CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';",
    "GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"
  ]
  revocation_statements = [
    "DROP ROLE IF EXISTS \"{{name}}\";"
  ]
  default_ttl = "1h"    # Credentials expire in 1 hour
  max_ttl     = "24h"   # Maximum 24 hours with renewal
}

# Terraform requests dynamic DB credentials
data "vault_database_secret_backend_creds" "app_creds" {
  backend = "database"
  role    = "app-role"
}

# Use dynamic credentials in RDS
resource "aws_db_instance" "app" {
  identifier     = "myapp-prod"
  engine         = "postgres"
  instance_class = "db.t3.medium"

  # These credentials are temporary and will expire!
  username = data.vault_database_secret_backend_creds.app_creds.username
  password = data.vault_database_secret_backend_creds.app_creds.password

  # After apply, Vault manages the rotation automatically
}


# ── 4. PKI Secrets Engine — Dynamic TLS Certificates ─────────────

data "vault_pki_secret_backend_cert" "app_cert" {
  backend     = "pki"
  name        = "myapp-role"
  common_name = "api.myapp.com"
  ttl         = "720h"    # 30 days
  ip_sans     = ["10.0.1.5"]
}

resource "aws_acm_certificate" "app" {
  # Import Vault-generated cert into ACM
  private_key       = data.vault_pki_secret_backend_cert.app_cert.private_key
  certificate_body  = data.vault_pki_secret_backend_cert.app_cert.certificate
  certificate_chain = data.vault_pki_secret_backend_cert.app_cert.ca_chain
}
```

---

**Vault Transit Engine — Encrypt Terraform State:**

```hcl
# Encrypt sensitive data before storing in state
# (Even if state is compromised, data is encrypted)

resource "vault_transit_secret_backend_key" "state_key" {
  backend          = "transit"
  name             = "terraform-state-key"
  type             = "aes256-gcm96"
  deletion_allowed = false
}

# Encrypt a value using Vault Transit
data "vault_transit_encrypt" "sensitive_config" {
  backend   = "transit"
  key       = vault_transit_secret_backend_key.state_key.name
  plaintext = base64encode(jsonencode({
    api_key    = var.api_key
    db_password = var.db_password
  }))
}

# Store encrypted ciphertext (safe to have in state)
resource "aws_ssm_parameter" "encrypted_config" {
  name  = "/myapp/prod/encrypted_config"
  type  = "String"    # Not SecureString — Vault handles encryption
  value = data.vault_transit_encrypt.sensitive_config.ciphertext
  # Ciphertext is: vault:v1:ABC123... (only Vault can decrypt this)
}
```

---

**Vault Audit Logging — Complete Audit Trail:**

```hcl
# Enable audit logging in Vault (every secret access is logged)
resource "vault_audit" "file_audit" {
  type = "file"

  options = {
    file_path   = "/var/log/vault/audit.log"
    log_raw     = "false"    # Don't log secret values!
    format      = "json"
    hmac_accessor = "true"   # HMAC-hash sensitive identifiers
  }
}

resource "vault_audit" "syslog_audit" {
  type = "syslog"

  options = {
    tag      = "vault"
    facility = "AUTH"
  }
}

# Every secret access creates a log entry like:
# {
#   "time": "2024-01-15T14:30:00Z",
#   "type": "response",
#   "auth": { "client_token": "hmac-sha256:abc123", "accessor": "hmac-sha256:def456",
#             "entity_id": "terraform-ci-runner" },
#   "request": { "id": "xyz789", "operation": "read",
#                "path": "secret/data/myapp/prod/config" },
#   "response": { "data": { "keys": ["api_key", "db_password"] } }
#                               ↑ Keys logged but NOT values
# }
```

---

### Sentinel — Policy as Code for Terraform Cloud/Enterprise

**What Sentinel Is:**

```
Sentinel is HashiCorp's policy-as-code framework.
It runs AFTER terraform plan but BEFORE terraform apply.
It's a gate — policies decide if apply is allowed to proceed.

Policy enforcement levels:
─────────────────────────────────────────────────────
ADVISORY    → Always passes. Just logs violations. (informational)
SOFT-MANDATORY → Passes unless overridden. Senior engineer can bypass.
HARD-MANDATORY → Always enforced. Cannot be overridden. Ever.
```

**Sentinel Policy Language:**

```python
# policies/restrict-instance-types.sentinel
# Policy: Only allow approved EC2 instance types

# Import Terraform plan data
import "tfplan/v2" as tfplan

# Define allowed instance types
allowed_types = [
  "t3.micro",
  "t3.small",
  "t3.medium",
  "t3.large",
  "m5.large",
  "m5.xlarge",
  "m5.2xlarge",
]

# Find all EC2 instances in the plan
all_ec2_instances = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_instance" and
  rc.mode is "managed" and
  rc.change.actions is not ["delete"]
}

# Check each instance
violations = filter all_ec2_instances as _, instance {
  instance.change.after.instance_type not in allowed_types
}

# Generate helpful error messages
msgs = map violations as _, v {
  v.address + " uses instance type '" +
  v.change.after.instance_type +
  "' which is not in the approved list: " +
  allowed_types
}

# The main rule — this determines pass/fail
main = rule {
  length(violations) is 0
} else {
  print("POLICY VIOLATION - Unapproved instance types:")
  print(msgs)
  false
}
```

```python
# policies/require-tags.sentinel
# Policy: All resources must have required tags

import "tfplan/v2" as tfplan

required_tags = ["Environment", "Owner", "CostCenter", "ManagedBy"]

# Resources that must have tags
taggable_resources = [
  "aws_instance",
  "aws_s3_bucket",
  "aws_db_instance",
  "aws_lb",
  "aws_vpc",
]

# Find all taggable resources being created/updated
all_resources = filter tfplan.resource_changes as _, rc {
  rc.type in taggable_resources and
  rc.mode is "managed" and
  rc.change.actions is not ["delete"]
}

# Check each resource for missing tags
violations = []

for all_resources as address, resource {
  tags = resource.change.after.tags else {}

  for required_tags as tag {
    if tag not in tags {
      append(violations, address + " is missing required tag: " + tag)
    }
  }
}

main = rule {
  length(violations) is 0
} else {
  print("POLICY VIOLATION - Missing required tags:")
  print(violations)
  false
}
```

```python
# policies/restrict-public-s3.sentinel
# Policy: No S3 buckets with public access

import "tfplan/v2" as tfplan

# Find S3 public access block resources
s3_public_access_blocks = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket_public_access_block" and
  rc.mode is "managed" and
  rc.change.actions is not ["delete"]
}

# Find S3 buckets without public access block
s3_buckets = filter tfplan.resource_changes as _, rc {
  rc.type is "aws_s3_bucket" and
  rc.mode is "managed" and
  rc.change.actions is not ["delete"]
}

# Check public access blocks are properly configured
violations = filter s3_public_access_blocks as _, resource {
  resource.change.after.block_public_acls       is not true or
  resource.change.after.block_public_policy     is not true or
  resource.change.after.ignore_public_acls      is not true or
  resource.change.after.restrict_public_buckets is not true
}

main = rule {
  length(violations) is 0
} else {
  print("POLICY VIOLATION - S3 buckets with public access:")
  print(map violations as _, v { v.address })
  false
}
```

```python
# policies/cost-estimation.sentinel
# Policy: Block resources estimated to cost more than $500/month
# (Requires Terraform Cloud Cost Estimation to be enabled)

import "tfcost" as cost

# Maximum monthly cost in USD
max_monthly_cost = 500

# Check total cost estimate
main = rule {
  cost.proposed.total_monthly_cost < max_monthly_cost
} else {
  print("POLICY VIOLATION - Estimated cost exceeds budget:")
  print("  Proposed monthly cost: $" + string(cost.proposed.total_monthly_cost))
  print("  Maximum allowed:       $" + string(max_monthly_cost))
  print("  Please optimize your infrastructure or request a budget exception.")
  false
}
```

**Sentinel Configuration:**

```python
# sentinel.hcl — Policy set configuration (in Terraform Cloud)

policy "restrict-instance-types" {
  source            = "./policies/restrict-instance-types.sentinel"
  enforcement_level = "hard-mandatory"   # Cannot be bypassed
}

policy "require-tags" {
  source            = "./policies/require-tags.sentinel"
  enforcement_level = "soft-mandatory"   # Can be overridden with reason
}

policy "restrict-public-s3" {
  source            = "./policies/restrict-public-s3.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "cost-estimation" {
  source            = "./policies/cost-estimation.sentinel"
  enforcement_level = "soft-mandatory"
}

# Modules for reusable policy logic
module "aws-functions" {
  source = "./modules/aws-functions.sentinel"
}

module "tfplan-functions" {
  source = "./modules/tfplan-functions.sentinel"
}
```

---

### OPA (Open Policy Agent) — Cloud-Native Policy Engine

**Why OPA over Sentinel:**

```
OPA ADVANTAGES:
✅ Open source — no vendor lock-in
✅ Works with ANY system (K8s, APIs, Terraform, Envoy, etc.)
✅ Widely adopted — CNCF graduated project
✅ Works with open-source Terraform
✅ Large community, many pre-built policies
✅ Rego language is powerful and expressive

SENTINEL ADVANTAGES:
✅ Native Terraform Cloud/Enterprise integration
✅ Tighter integration with Terraform plan data
✅ Simpler language for Terraform-specific policies
✅ Cost estimation integration
✅ HashiCorp support
```

**OPA with Conftest — Terraform Integration:**

```bash
# Install conftest
brew install conftest  # macOS
# OR
wget https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz

# Generate Terraform plan as JSON
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Run policies against plan
conftest test plan.json --policy ./policies/

# Output:
# FAIL - plan.json - main - S3 bucket 'aws_s3_bucket.data' must not be public
# FAIL - plan.json - main - EC2 instance 'aws_instance.web' missing required tag: Owner
# PASS - plan.json - Approved instance type t3.large
```

---

**OPA Rego Policies — Complete Examples:**

```rego
# policies/terraform/s3_security.rego
package terraform.s3

import future.keywords.if
import future.keywords.in

# ── Rule 1: No public S3 buckets ─────────────────────────────────
deny[msg] {
  # Find S3 buckets in planned changes
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] != "delete"

  # Check if ACL is public
  resource.change.after.acl == "public-read"

  msg := sprintf(
    "SECURITY VIOLATION: S3 bucket '%s' has public-read ACL. All buckets must be private.",
    [resource.address]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] != "delete"
  resource.change.after.acl == "public-read-write"

  msg := sprintf(
    "SECURITY VIOLATION: S3 bucket '%s' has public-read-write ACL. This is never allowed.",
    [resource.address]
  )
}


# ── Rule 2: S3 must have server-side encryption ───────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] != "delete"

  # Check if encryption configuration exists
  not bucket_has_encryption(resource.address)

  msg := sprintf(
    "COMPLIANCE VIOLATION: S3 bucket '%s' must have server-side encryption enabled.",
    [resource.address]
  )
}

# Helper: check if encryption block resource exists for this bucket
bucket_has_encryption(bucket_address) {
  encryption_resource := input.resource_changes[_]
  encryption_resource.type == "aws_s3_bucket_server_side_encryption_configuration"
  encryption_resource.change.after.bucket != null
}


# ── Rule 3: S3 versioning required for specific buckets ───────────
warn[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] != "delete"

  # Check name suggests it's a data bucket
  contains(resource.change.after.bucket, "data")

  # Check versioning status
  versioning := resource.change.after.versioning
  versioning[_].enabled == false

  msg := sprintf(
    "WARNING: Data bucket '%s' should have versioning enabled for compliance.",
    [resource.address]
  )
}
```

```rego
# policies/terraform/aws_security.rego
package terraform.aws

import future.keywords.if
import future.keywords.in

# ── Rule 1: EC2 instances must not have public IPs ────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  resource.change.actions[_] != "delete"
  resource.change.after.associate_public_ip_address == true

  msg := sprintf(
    "SECURITY: EC2 instance '%s' must not have a public IP. Use a load balancer instead.",
    [resource.address]
  )
}


# ── Rule 2: Security groups must not allow 0.0.0.0/0 on SSH ──────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  resource.change.actions[_] != "delete"

  ingress := resource.change.after.ingress[_]
  ingress.from_port <= 22
  ingress.to_port >= 22
  ingress.cidr_blocks[_] == "0.0.0.0/0"

  msg := sprintf(
    "SECURITY VIOLATION: Security group '%s' allows SSH from 0.0.0.0/0. Restrict to VPN CIDR.",
    [resource.address]
  )
}


# ── Rule 3: RDS must have multi-AZ in prod ────────────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_db_instance"
  resource.change.actions[_] != "delete"

  # Check environment tag
  resource.change.after.tags.Environment == "prod"

  # Check multi-AZ is not enabled
  resource.change.after.multi_az == false

  msg := sprintf(
    "COMPLIANCE: RDS instance '%s' in prod must have multi_az = true for HA.",
    [resource.address]
  )
}


# ── Rule 4: All resources must have required tags ──────────────────
required_tags := {"Environment", "Owner", "ManagedBy", "CostCenter"}

taggable_types := {
  "aws_instance",
  "aws_s3_bucket",
  "aws_db_instance",
  "aws_lb",
  "aws_vpc",
  "aws_subnet",
  "aws_security_group",
  "aws_lambda_function",
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type in taggable_types
  resource.change.actions[_] != "delete"

  # Get actual tags (handle null case)
  tags := object.get(resource.change.after, "tags", {})

  # Find missing required tags
  missing := required_tags - {tag | tags[tag]}
  count(missing) > 0

  msg := sprintf(
    "COMPLIANCE: Resource '%s' (type: %s) is missing required tags: %v",
    [resource.address, resource.type, missing]
  )
}


# ── Rule 5: Approved instance types only ─────────────────────────
approved_instance_families := {"t3", "t3a", "m5", "m5a", "c5", "r5"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  resource.change.actions[_] != "delete"

  instance_type := resource.change.after.instance_type
  family := split(instance_type, ".")[0]

  not approved_instance_families[family]

  msg := sprintf(
    "POLICY: Instance type '%s' for resource '%s' is not in the approved families: %v",
    [instance_type, resource.address, approved_instance_families]
  )
}


# ── Rule 6: No destruction of production resources ────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.change.actions[_] == "delete"

  # Check if it's a production resource by tag
  resource.change.before.tags.Environment == "prod"

  # Except for specifically whitelisted types
  not allow_prod_deletion(resource.type)

  msg := sprintf(
    "CRITICAL: Destruction of prod resource '%s' (type: %s) requires manual override!",
    [resource.address, resource.type]
  )
}

allow_prod_deletion(resource_type) {
  # These types are OK to delete in prod (e.g., temporary resources)
  allowed_types := {"aws_cloudwatch_metric_alarm", "aws_autoscaling_policy"}
  resource_type in allowed_types
}


# ── Rule 7: KMS encryption required for sensitive resources ────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_db_instance"
  resource.change.actions[_] != "delete"

  # Check storage is not encrypted
  not resource.change.after.storage_encrypted

  msg := sprintf(
    "SECURITY: RDS instance '%s' must have storage_encrypted = true.",
    [resource.address]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_ebs_volume"
  resource.change.actions[_] != "delete"

  not resource.change.after.encrypted

  msg := sprintf(
    "SECURITY: EBS volume '%s' must be encrypted.",
    [resource.address]
  )
}
```

```rego
# policies/terraform/cost_controls.rego
package terraform.cost

import future.keywords.if
import future.keywords.in

# ── Rule: Block expensive instance types ──────────────────────────
expensive_types := {
  "m5.16xlarge", "m5.24xlarge",
  "c5.18xlarge", "c5.24xlarge",
  "r5.12xlarge", "r5.16xlarge", "r5.24xlarge",
  "p3.8xlarge",  "p3.16xlarge",
  "x1.16xlarge", "x1.32xlarge",
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  resource.change.actions[_] != "delete"

  instance_type := resource.change.after.instance_type
  instance_type in expensive_types

  msg := sprintf(
    "COST CONTROL: Instance type '%s' for '%s' exceeds cost limits. Max allowed: m5.4xlarge. Request exception via #infra-requests.",
    [instance_type, resource.address]
  )
}

# ── Rule: Multi-AZ RDS only allowed in prod/staging ───────────────
warn[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_db_instance"
  resource.change.actions[_] != "delete"
  resource.change.after.multi_az == true

  env := object.get(resource.change.after.tags, "Environment", "unknown")
  env == "dev"

  msg := sprintf(
    "COST WARNING: RDS instance '%s' has multi_az=true in dev environment. Consider single-AZ to reduce costs.",
    [resource.address]
  )
}
```

---

### Integrating OPA into CI/CD Pipeline

```yaml
# .github/workflows/terraform-security.yml
name: Terraform Security & Compliance

on:
  pull_request:
    branches: [main]

jobs:
  security-compliance:
    name: Security & Compliance Checks
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      # ── Step 1: Generate plan JSON ────────────────────────────
      - name: Terraform Init & Plan
        working-directory: environments/prod
        run: |
          terraform init -input=false
          terraform plan \
            -input=false \
            -out=tfplan \
            -var-file=terraform.tfvars
          terraform show -json tfplan > plan.json

      # ── Step 2: OPA/Conftest Policy Check ─────────────────────
      - name: Install Conftest
        run: |
          VERSION="0.46.0"
          wget -q "https://github.com/open-policy-agent/conftest/releases/download/v${VERSION}/conftest_${VERSION}_Linux_x86_64.tar.gz"
          tar xzf "conftest_${VERSION}_Linux_x86_64.tar.gz"
          sudo mv conftest /usr/local/bin/

      - name: Run OPA Policies
        id: opa
        working-directory: environments/prod
        run: |
          conftest test plan.json \
            --policy ../../policies/terraform/ \
            --output json \
            2>&1 | tee opa_results.json

          # Check exit code
          conftest test plan.json \
            --policy ../../policies/terraform/ \
            --no-color \
            2>&1 | tee opa_results.txt

          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      # ── Step 3: tfsec Scan ────────────────────────────────────
      - name: Run tfsec
        id: tfsec
        uses: aquasecurity/tfsec-action@v1
        with:
          working_directory: environments/prod
          format: json
          additional_args: --out tfsec-results.json
        continue-on-error: true

      # ── Step 4: Checkov Scan ──────────────────────────────────
      - name: Run Checkov
        id: checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: environments/prod
          framework: terraform
          output_format: json
          output_file_path: checkov-results.json
          soft_fail: true
        continue-on-error: true

      # ── Step 5: Aggregate and Post Results ────────────────────
      - name: Post Security Report to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');

            // Read OPA results
            let opaOutput = '';
            try {
              opaOutput = fs.readFileSync(
                'environments/prod/opa_results.txt', 'utf8'
              );
            } catch(e) { opaOutput = 'OPA results not found'; }

            const opaStatus = '${{ steps.opa.outputs.exit_code }}' === '0' ? '✅' : '❌';
            const tfsecStatus = '${{ steps.tfsec.outcome }}' === 'success' ? '✅' : '⚠️';
            const checkovStatus = '${{ steps.checkov.outcome }}' === 'success' ? '✅' : '⚠️';

            const body = `## 🔐 Security & Compliance Report

            | Check | Status |
            |-------|--------|
            | OPA Policy Check | ${opaStatus} |
            | tfsec Security Scan | ${tfsecStatus} |
            | Checkov CIS Benchmark | ${checkovStatus} |

            <details>
            <summary>OPA Policy Details</summary>

            \`\`\`
            ${opaOutput.slice(0, 5000)}
            \`\`\`
            </details>

            ${opaStatus === '❌' ? '> ❌ **OPA policy violations must be resolved before merging.**' : '> ✅ All security checks passed!'}
            `;

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body
            });

      # ── Step 6: Fail if critical violations ───────────────────
      - name: Fail on OPA Violations
        if: steps.opa.outputs.exit_code != '0'
        run: |
          echo "❌ OPA policy violations detected! Review the PR comment."
          echo "Run locally: conftest test plan.json --policy ./policies/terraform/"
          exit 1
```

---

### Vault + Terraform — Production Patterns

```hcl
# ── Pattern 1: Vault Agent Sidecar (K8s) ─────────────────────────
# Vault Agent runs alongside Terraform, manages token renewal
# and injects secrets as environment variables

# vault-agent-config.hcl
# auto_auth {
#   method "kubernetes" {
#     mount_path = "auth/kubernetes"
#     config = {
#       role = "terraform-role"
#     }
#   }
# }
# template {
#   source      = "/vault/templates/aws-creds.tpl"
#   destination = "/vault/secrets/aws-creds.env"
#   command     = "reload-terraform"
# }


# ── Pattern 2: Vault Secret Rotation Trigger ──────────────────────
# When Vault rotates a secret, trigger Terraform to update resources

resource "vault_generic_secret" "app_config" {
  path = "secret/myapp/prod"

  data_json = jsonencode({
    api_version = "v2"
    app_name    = "myapp"
    environment = "prod"
  })

  # Disable auto-read on every plan (performance)
  disable_read = false
}

# Watch for changes: Vault webhook → CI/CD → terraform apply


# ── Pattern 3: Namespace Isolation per Environment ────────────────
provider "vault" {
  address   = "https://vault.mycompany.com"
  namespace = "engineering/${var.environment}"    # Vault Enterprise namespaces
}

# dev → engineering/dev namespace
# prod → engineering/prod namespace — completely isolated policies


# ── Pattern 4: Break-glass Emergency Access ───────────────────────
# For emergencies when Vault is unavailable:
resource "aws_secretsmanager_secret" "break_glass" {
  name                    = "break-glass/terraform-credentials"
  recovery_window_in_days = 7

  tags = {
    Purpose = "break-glass-emergency-only"
    Audit   = "required"
  }
}

# Access triggers CloudWatch alarm + PagerDuty page
resource "aws_cloudwatch_metric_alarm" "break_glass_accessed" {
  alarm_name          = "break-glass-secret-accessed"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "CallCount"
  namespace           = "CloudTrailMetrics"
  period              = 60
  statistic           = "Sum"
  threshold           = 0

  alarm_actions = [var.pagerduty_sns_arn]
  alarm_description = "ALERT: Break-glass credentials accessed! Investigate immediately."
}
```

---

## 🔴 ADVANCED LEVEL

### Advanced OPA Patterns

```rego
# policies/terraform/advanced.rego
package terraform.advanced

import future.keywords.if
import future.keywords.in
import future.keywords.every

# ── Advanced Rule: Network topology enforcement ───────────────────
# Ensure databases are NEVER in public subnets

deny[msg] {
  # Find RDS instances
  db := input.resource_changes[_]
  db.type == "aws_db_instance"
  db.change.actions[_] != "delete"

  # Find subnet group for this DB
  subnet_group := input.resource_changes[_]
  subnet_group.type == "aws_db_subnet_group"
  subnet_group.change.after.name == db.change.after.db_subnet_group_name

  # Find the subnets in the subnet group
  subnet_id := subnet_group.change.after.subnet_ids[_]

  # Check if any subnet is public (has route to internet gateway)
  subnet := input.resource_changes[_]
  subnet.type == "aws_subnet"
  subnet.change.after.id == subnet_id
  subnet.change.after.map_public_ip_on_launch == true

  msg := sprintf(
    "ARCHITECTURE VIOLATION: RDS instance '%s' is placed in a public subnet '%s'. Databases must be in private subnets only.",
    [db.address, subnet_id]
  )
}


# ── Advanced Rule: Ensure security group rules are least-privilege ─
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  resource.change.actions[_] != "delete"

  # Check each ingress rule
  ingress := resource.change.after.ingress[_]

  # Port range is too broad (more than 100 ports open at once)
  port_range := ingress.to_port - ingress.from_port
  port_range > 100

  # And it's open to a wide CIDR
  cidr := ingress.cidr_blocks[_]
  startswith(cidr, "0.0.0.0") == true

  msg := sprintf(
    "SECURITY: Security group '%s' has an overly broad ingress rule (ports %d-%d open to %s). Use specific ports.",
    [resource.address, ingress.from_port, ingress.to_port, cidr]
  )
}


# ── Advanced Rule: Cross-resource relationship validation ──────────
# Ensure every EC2 instance has monitoring enabled in prod

deny[msg] {
  instance := input.resource_changes[_]
  instance.type == "aws_instance"
  instance.change.actions[_] != "delete"

  # It's a prod instance
  instance.change.after.tags.Environment == "prod"

  # Check if monitoring is disabled
  not instance.change.after.monitoring

  msg := sprintf(
    "COMPLIANCE: EC2 instance '%s' in prod must have detailed monitoring enabled. Add: monitoring = true",
    [instance.address]
  )
}


# ── Advanced Rule: Naming convention enforcement ──────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type in {"aws_instance", "aws_s3_bucket", "aws_db_instance", "aws_lb"}
  resource.change.actions[_] != "delete"

  name := resource.change.after.tags.Name

  # Name must match pattern: <app>-<env>-<component>
  # e.g., myapp-prod-web, myapp-dev-db
  not regex.match("^[a-z][a-z0-9]+-(?:dev|staging|prod)-[a-z][a-z0-9-]+$", name)

  msg := sprintf(
    "NAMING: Resource '%s' has invalid Name tag '%s'. Required format: <app>-<env>-<component>",
    [resource.address, name]
  )
}


# ── Advanced Rule: IAM policy least privilege ─────────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type in {"aws_iam_role_policy", "aws_iam_policy"}
  resource.change.actions[_] != "delete"

  # Parse the policy document
  policy := json.unmarshal(resource.change.after.policy)

  statement := policy.Statement[_]
  statement.Effect == "Allow"

  # Check for wildcard action
  statement.Action == "*"

  msg := sprintf(
    "SECURITY: IAM policy '%s' has a wildcard Action ('*'). IAM policies must follow least-privilege.",
    [resource.address]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type in {"aws_iam_role_policy", "aws_iam_policy"}
  resource.change.actions[_] != "delete"

  policy := json.unmarshal(resource.change.after.policy)
  statement := policy.Statement[_]
  statement.Effect == "Allow"

  # Check for wildcard resource
  statement.Resource == "*"

  # With non-read-only actions
  action := statement.Action[_]
  not startswith(action, "Get")
  not startswith(action, "List")
  not startswith(action, "Describe")

  msg := sprintf(
    "SECURITY: IAM policy '%s' grants '%s' on all resources ('*'). Restrict to specific resource ARNs.",
    [resource.address, action]
  )
}
```

---

### Compliance Framework Mapping

```rego
# policies/terraform/cis-aws.rego
# Maps to CIS AWS Foundations Benchmark v1.5.0
package terraform.cis

import future.keywords.if
import future.keywords.in

# ── CIS 2.1.1: S3 Block Public Access ─────────────────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.actions[_] != "delete"

  not resource.change.after.block_public_acls

  msg := sprintf(
    "[CIS-2.1.1] S3 bucket '%s': block_public_acls must be true",
    [resource.address]
  )
}


# ── CIS 2.2.1: EBS Volumes Encrypted ─────────────────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_ebs_volume"
  resource.change.actions[_] != "delete"
  not resource.change.after.encrypted

  msg := sprintf(
    "[CIS-2.2.1] EBS volume '%s' must be encrypted",
    [resource.address]
  )
}


# ── CIS 3.1: CloudTrail Enabled ──────────────────────────────────
# (Check that CloudTrail resource exists in plan)
deny[msg] {
  # Count CloudTrail resources being created
  trails := [r |
    r := input.resource_changes[_]
    r.type == "aws_cloudtrail"
    r.change.actions[_] != "delete"
  ]

  # If we're creating other AWS resources but no CloudTrail
  other_resources := [r |
    r := input.resource_changes[_]
    r.type != "aws_cloudtrail"
    r.change.actions[_] != "delete"
  ]

  count(trails) == 0
  count(other_resources) > 0

  msg := "[CIS-3.1] CloudTrail must be enabled. Add an aws_cloudtrail resource."
}


# ── CIS 4.1: No Security Groups Allow SSH from 0.0.0.0/0 ─────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  resource.change.actions[_] != "delete"

  ingress := resource.change.after.ingress[_]
  ingress.from_port <= 22
  ingress.to_port >= 22
  ingress.cidr_blocks[_] == "0.0.0.0/0"

  msg := sprintf(
    "[CIS-4.1] Security group '%s' allows SSH (port 22) from 0.0.0.0/0",
    [resource.address]
  )
}


# ── CIS 4.2: No Security Groups Allow RDP from 0.0.0.0/0 ─────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  resource.change.actions[_] != "delete"

  ingress := resource.change.after.ingress[_]
  ingress.from_port <= 3389
  ingress.to_port >= 3389
  ingress.cidr_blocks[_] == "0.0.0.0/0"

  msg := sprintf(
    "[CIS-4.2] Security group '%s' allows RDP (port 3389) from 0.0.0.0/0",
    [resource.address]
  )
}
```

---

### Building a Complete Security Pipeline

```
COMPLETE SECURITY PIPELINE FLOW:
─────────────────────────────────────────────────────────────────

Developer opens PR
        │
        ▼
┌─────────────────┐
│  Static Analysis │  terraform validate + tflint
│  (pre-plan)      │  Catches syntax errors, bad practices
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  Terraform Plan  │  Generate plan.json
│                  │  Uses Vault for dynamic credentials
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  Security Scan   │  tfsec + Checkov + custom rules
│  (post-plan)     │  Scan plan.json for known vulnerabilities
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  OPA/Conftest    │  Custom business + compliance policies
│  Policy Check    │  CIS benchmark, naming, tagging, cost
└────────┬─────────┘
         │ All checks pass?
         ▼
┌─────────────────┐
│  PR Review       │  Human review of plan + security report
│  (human gate)    │  Required: 2 approvers for prod
└────────┬─────────┘
         │ Approved
         ▼
┌─────────────────┐
│  Sentinel Check  │  (If using TF Cloud/Enterprise)
│  (pre-apply)     │  Final policy gate before apply
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  Apply           │  terraform apply
│                  │  Vault issues temp credentials
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  Post-Apply      │  Verify outputs
│  Verification    │  Run infrastructure tests
│                  │  Update CMDB/asset registry
└────────┬─────────┘
         │
         ▼
┌─────────────────┐
│  Audit & Alert   │  All actions logged to Vault audit
│                  │  CloudTrail records API calls
│                  │  Slack notification to team
└─────────────────┘
```

---

### Production Best Practices

```hcl
# ✅ 1. Never store Vault tokens in files — use env vars or auth methods
# export VAULT_ADDR=https://vault.mycompany.com
# export VAULT_TOKEN=$(vault login -method=aws -field=token)

# ✅ 2. Set appropriate TTLs for dynamic credentials
resource "vault_database_secret_backend_role" "app" {
  default_ttl = "2h"     # Enough for longest apply
  max_ttl     = "24h"    # Hard maximum

  # If apply takes 90 min, TTL=2h gives plenty of buffer
}

# ✅ 3. Use OPA warn[] for advisory, deny[] for blocking
# warn  → shows in output, doesn't block apply
# deny  → blocks apply entirely

# ✅ 4. Version-pin your policy files (keep in Git with semantic versions)
# policies/
# ├── v1.0.0/
# │   ├── aws_security.rego
# │   └── tagging.rego
# └── v1.1.0/
#     ├── aws_security.rego  (updated)
#     └── tagging.rego

# ✅ 5. Test OPA policies with conftest verify
# Create test files:
# policies/terraform/aws_security_test.rego

# ✅ 6. Separate policy repos from infrastructure repos
# infra-policies/ (separate repo, policy team owns)
# ├── README.md
# ├── policies/
# └── tests/
#
# my-infrastructure/ (app team owns)
# Uses policies as a submodule or downloads from registry

# ✅ 7. Use Vault namespaces for environment isolation (Enterprise)
provider "vault" {
  namespace = "engineering/${var.environment}"
}

# ✅ 8. Implement policy exceptions as code (not verbal approvals)
# exceptions.json
# {
#   "exception_id": "EXC-2024-001",
#   "resource": "aws_instance.legacy",
#   "policy": "restrict-instance-types",
#   "reason": "Legacy workload requires t2.medium — migration planned Q2 2024",
#   "approved_by": "jane.smith@company.com",
#   "expires": "2024-06-30"
# }
```

---

### Debugging Security & Compliance Issues

```bash
# ── Debug OPA Policies ────────────────────────────────────────────

# Test a specific policy against plan
conftest test plan.json \
  --policy ./policies/terraform/aws_security.rego \
  --trace    # Show evaluation trace

# Test with verbose output
conftest test plan.json \
  --policy ./policies/terraform/ \
  --output table \
  --all-namespaces

# Interactive OPA evaluation
opa eval \
  --data ./policies/terraform/aws_security.rego \
  --input plan.json \
  "data.terraform.aws.deny" \
  --format pretty

# Test individual rules
opa eval \
  --data ./policies/terraform/aws_security.rego \
  --input plan.json \
  'data.terraform.aws.deny[x]' \
  --format pretty


# ── Debug Vault Issues ────────────────────────────────────────────

# Check Vault status
vault status

# Verify auth method works
vault login -method=aws -field=token

# Test secret access manually
vault read secret/data/myapp/prod/config

# Check policies attached to token
vault token lookup

# Verify dynamic DB credentials work
vault read database/creds/app-role

# View audit logs
vault audit list
tail -f /var/log/vault/audit.log | jq .


# ── Debug Sentinel ────────────────────────────────────────────────

# Test Sentinel policy locally (requires Sentinel CLI)
sentinel test ./policies/restrict-instance-types.sentinel

# Apply with policy override (soft-mandatory only)
# In Terraform Cloud UI: "Override & Continue" button
# Requires "policy-override" permission on the team


# ── Common Errors and Fixes ───────────────────────────────────────

# Error: "permission denied" in Vault
# Fix: Check Vault policy attached to your auth method's role
vault policy read terraform-policy

# Error: "lease expired during apply"
# Fix: Increase TTL on the secret engine role
# Or: Use Vault Agent to auto-renew leases

# Error: "conftest: no files provided"
# Fix: Generate plan.json first:
terraform show -json tfplan > plan.json
conftest test plan.json --policy ./policies/

# Error: OPA policy "undefined" result
# Fix: Check package name matches conftest namespace
# package terraform.aws → conftest test --namespace terraform.aws
```

---

## 📁 CODE WRITING GUIDANCE

### Complete Security Project Structure

```
infrastructure/
│
├── policies/                          ← Policy repository
│   ├── terraform/
│   │   ├── aws_security.rego          ← Security rules
│   │   ├── cis_aws.rego               ← CIS benchmark
│   │   ├── tagging.rego               ← Tagging policy
│   │   ├── cost_controls.rego         ← Cost policies
│   │   ├── naming_conventions.rego    ← Naming standards
│   │   └── tests/                     ← Policy unit tests
│   │       ├── aws_security_test.rego
│   │       └── fixtures/
│   │           ├── pass_plan.json     ← Valid plan for testing
│   │           └── fail_plan.json     ← Invalid plan for testing
│   │
│   ├── sentinel/                      ← TFC/TFE policies
│   │   ├── sentinel.hcl
│   │   ├── restrict-instance-types.sentinel
│   │   ├── require-tags.sentinel
│   │   └── modules/
│   │       └── aws-functions.sentinel
│   │
│   └── exceptions/                    ← Approved exceptions
│       └── exceptions.json
│
├── vault/                             ← Vault configuration
│   ├── main.tf                        ← Vault resources
│   ├── auth-methods.tf                ← AppRole, AWS, K8s auth
│   ├── secret-engines.tf              ← KV, AWS, DB engines
│   ├── policies/                      ← Vault HCL policies
│   │   ├── terraform-dev.hcl
│   │   ├── terraform-prod.hcl
│   │   └── readonly.hcl
│   └── audit.tf                       ← Audit configuration
│
├── environments/
│   └── prod/
│       ├── main.tf                    ← Uses Vault provider
│       ├── vault.tf                   ← Vault data sources
│       └── ...
│
└── .github/
    └── workflows/
        ├── security-scan.yml          ← OPA + tfsec + checkov
        └── terraform-apply.yml        ← Applies with Vault auth
```

---

## 🧪 HANDS-ON LAB

### Exercise: Complete Security Pipeline

**Goal:** Build a security-first Terraform setup with:
1. Three OPA policies: no-public-s3, required-tags, approved-instance-types
2. A GitHub Actions workflow that runs OPA checks on every PR
3. A Vault data source reading a KV secret (can mock with local Vault dev server)
4. Generate a plan, run it through conftest, and see pass/fail output

**Setup:**
```bash
# Start local Vault dev server (for testing)
vault server -dev -dev-root-token-id="dev-root-token"
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="dev-root-token"

# Create test secret
vault kv put secret/myapp/dev \
  api_key="test-api-key-12345" \
  db_password="test-db-password-67890"

# Install conftest
brew install conftest
```

**Try implementing the policies and pipeline yourself!**

---

### ✅ Solution

```rego
# policies/terraform/main.rego
package main

import future.keywords.if
import future.keywords.in

# ── Policy 1: No Public S3 ────────────────────────────────────────
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.actions[_] != "delete"

  acl := object.get(resource.change.after, "acl", "private")
  acl in {"public-read", "public-read-write", "authenticated-read"}

  msg := sprintf(
    "❌ [S3-PUBLIC] Bucket '%s' has ACL '%s'. All buckets must be private.",
    [resource.address, acl]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket_public_access_block"
  resource.change.actions[_] != "delete"

  not resource.change.after.block_public_acls

  msg := sprintf(
    "❌ [S3-PUBLIC] Resource '%s': block_public_acls must be true",
    [resource.address]
  )
}


# ── Policy 2: Required Tags ───────────────────────────────────────
required_tags := {"Environment", "Owner", "ManagedBy"}

taggable_types := {
  "aws_instance", "aws_s3_bucket",
  "aws_db_instance", "aws_vpc", "aws_lb"
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type in taggable_types
  resource.change.actions[_] != "delete"

  tags := object.get(resource.change.after, "tags", {})
  missing := required_tags - {tag | tags[tag]}
  count(missing) > 0

  msg := sprintf(
    "❌ [TAGS] Resource '%s' missing required tags: %v",
    [resource.address, missing]
  )
}


# ── Policy 3: Approved Instance Types ────────────────────────────
approved_types := {
  "t3.micro", "t3.small", "t3.medium", "t3.large",
  "m5.large", "m5.xlarge", "m5.2xlarge"
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  resource.change.actions[_] != "delete"

  instance_type := resource.change.after.instance_type
  not instance_type in approved_types

  msg := sprintf(
    "❌ [COST] Instance type '%s' for '%s' not approved. Approved: %v",
    [instance_type, resource.address, approved_types]
  )
}

# ── Warnings (non-blocking) ───────────────────────────────────────
warn[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  resource.change.actions[_] != "delete"

  not resource.change.after.monitoring

  msg := sprintf(
    "⚠️  [MONITORING] Instance '%s' should have monitoring = true",
    [resource.address]
  )
}
```

```hcl
# main.tf — Test infrastructure (intentionally has violations)
provider "aws" { region = "us-east-1" }

# ❌ VIOLATION: Public S3 bucket
resource "aws_s3_bucket" "bad_bucket" {
  bucket = "my-public-bucket-test"
  acl    = "public-read"   # ← Will be caught by policy
  # Missing required tags! ← Will be caught by policy
}

# ✅ COMPLIANT: Proper bucket
resource "aws_s3_bucket" "good_bucket" {
  bucket = "my-private-bucket-test"

  tags = {
    Environment = "dev"
    Owner       = "devops-team"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_public_access_block" "good_bucket" {
  bucket                  = aws_s3_bucket.good_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ❌ VIOLATION: Unapproved instance type + missing tags
resource "aws_instance" "bad_instance" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "m5.24xlarge"    # ← Not approved!
  # Missing tags ← Caught!
}
```

```bash
# Run the lab:
terraform init
terraform plan -out=tfplan
terraform show -json tfplan > plan.json

# Run OPA checks
conftest test plan.json --policy ./policies/terraform/

# Expected output:
# FAIL - plan.json - main - ❌ [S3-PUBLIC] Bucket 'aws_s3_bucket.bad_bucket' has ACL 'public-read'
# FAIL - plan.json - main - ❌ [TAGS] Resource 'aws_s3_bucket.bad_bucket' missing required tags: {"Environment", "ManagedBy", "Owner"}
# FAIL - plan.json - main - ❌ [COST] Instance type 'm5.24xlarge' for 'aws_instance.bad_instance' not approved
# FAIL - plan.json - main - ❌ [TAGS] Resource 'aws_instance.bad_instance' missing required tags: ...
# WARN - plan.json - main - ⚠️  [MONITORING] Instance 'aws_instance.bad_instance' should have monitoring = true
# 5 tests, 0 passed, 1 warning, 4 failures
```

```yaml
# .github/workflows/security.yml
name: Security & Compliance

on:
  pull_request:
    branches: [main]

jobs:
  opa-check:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Terraform Plan
        run: |
          terraform init -input=false
          terraform plan -out=tfplan -input=false -var-file=terraform.tfvars
          terraform show -json tfplan > plan.json

      - name: Install Conftest
        run: |
          wget -q https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
          tar xzf conftest_0.46.0_Linux_x86_64.tar.gz && sudo mv conftest /usr/local/bin/

      - name: Run OPA Policies
        id: opa
        run: |
          conftest test plan.json \
            --policy ./policies/terraform/ \
            --no-color 2>&1 | tee opa_results.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Post Results to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const results = fs.readFileSync('opa_results.txt', 'utf8');
            const passed = '${{ steps.opa.outputs.exit_code }}' === '0';
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## ${passed ? '✅' : '❌'} OPA Policy Results\n\`\`\`\n${results}\n\`\`\``
            });

      - name: Fail if violations
        if: steps.opa.outputs.exit_code != '0'
        run: exit 1
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Tool | Purpose | Works With | Key Feature |
|---|---|---|---|
| **Vault** | Secrets management | All Terraform | Dynamic secrets, auto-expiry, audit logs |
| **Sentinel** | Policy enforcement | TFC/TFE only | Native integration, cost estimation |
| **OPA/Conftest** | Policy enforcement | Any CI/CD | Open source, multi-system, Rego language |
| **tfsec** | Security scanning | Any | AWS/Azure/GCP rule library |
| **Checkov** | Compliance scanning | Any | CIS benchmarks, SOC2, HIPAA |

---

### Quick Reference

```bash
# Vault
vault server -dev                             # Start dev server
vault kv put secret/path key=value           # Store secret
vault kv get secret/path                     # Read secret
vault read database/creds/my-role            # Get dynamic DB creds
vault token lookup                            # Check current token

# OPA/Conftest
terraform show -json tfplan > plan.json       # Generate plan JSON
conftest test plan.json --policy ./policies/  # Run all policies
conftest test plan.json --policy file.rego    # Run specific policy
conftest test plan.json --output table        # Table format
conftest test plan.json --trace               # Debug trace
opa eval -d policy.rego -i plan.json "data.pkg.deny"  # Direct OPA eval

# Sentinel (TFC/TFE)
sentinel test ./policies/                     # Test policies locally
sentinel apply -config=sentinel.hcl          # Apply policies

# Security Scanners
tfsec . --severity HIGH                       # High severity only
checkov -d . --framework terraform            # Checkov scan
```

```rego
# OPA Policy Template
package main                          # or: package terraform.aws

import future.keywords.if
import future.keywords.in

# Block (hard)
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "RESOURCE_TYPE"
  resource.change.actions[_] != "delete"
  # YOUR CONDITION
  msg := sprintf("Error message for %s", [resource.address])
}

# Warn (soft)
warn[msg] {
  resource := input.resource_changes[_]
  # YOUR CONDITION
  msg := sprintf("Warning for %s", [resource.address])
}
```

---

### Top 10 Interview Questions

1. **What is the difference between Vault static and dynamic secrets?** Static secrets are stored key-value pairs that don't change unless manually rotated. Dynamic secrets are generated on-demand with a TTL — Vault creates a temporary credential (e.g., DB user), and automatically destroys it when the lease expires.

2. **How does Vault AppRole authentication work?** AppRole uses a `role_id` (like a username) and `secret_id` (like a password). CI/CD pipelines use AppRole to authenticate — `role_id` is static and stored in code, `secret_id` is dynamic and rotated regularly.

3. **What's the difference between Sentinel and OPA?** Sentinel is HashiCorp-native, runs only in Terraform Cloud/Enterprise, has tighter plan integration and cost estimation. OPA is open-source, works with any system, uses Rego language, and integrates via Conftest in any CI/CD pipeline.

4. **What are Sentinel enforcement levels?** Advisory (always passes, logs violations), soft-mandatory (blocks unless senior engineer overrides), hard-mandatory (always blocks, cannot be overridden by anyone).

5. **How do you handle Vault lease expiry during a long Terraform apply?** Set the credential TTL longer than your longest expected apply (e.g., TTL=4h for a 90-minute apply). Alternatively, use Vault Agent which automatically renews leases while Terraform runs.

6. **What does `terraform show -json tfplan > plan.json` produce?** A machine-readable JSON representation of the Terraform plan, including all resource changes, before/after values, and actions. This is what OPA/Conftest policies evaluate.

7. **How do you test OPA policies without running Terraform?** Create fixture JSON files (`pass_plan.json` and `fail_plan.json`) that simulate plan output, then run `conftest test pass_plan.json --policy ./policies/` and `conftest verify` to unit-test your Rego rules.

8. **What is the Vault Transit secrets engine?** A "encryption as a service" engine that lets applications encrypt and decrypt data without having access to the encryption key itself. Useful for encrypting sensitive values before storing them in Terraform state.

9. **How do you implement policy exceptions without bypassing security?** Code policy exceptions as structured data in a JSON file with fields like `exception_id`, `resource`, `reason`, `approved_by`, and `expires`. OPA policies check the exceptions file and allow the resource if a valid, non-expired exception exists.

10. **What is OIDC and how does it relate to Vault and CI/CD?** OpenID Connect is a federated identity protocol. GitHub Actions, GitLab, and others issue JWT tokens that Vault and AWS can verify directly — eliminating the need for static credentials entirely. CI/CD authenticates to Vault via OIDC, Vault issues dynamic cloud credentials, Terraform uses them to apply.

---
