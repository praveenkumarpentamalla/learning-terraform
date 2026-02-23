# 🗄️ TOPIC 4: State Management — Deep Dive

---

## 🟢 BEGINNER LEVEL

### What is Terraform State? (Simple Terms)

Imagine you're a **contractor building houses** for a client. After building 10 houses, you keep a **detailed notebook** that records:
- Which houses you built
- Their exact addresses
- What materials you used
- What the client asked for vs what exists

Now if the client says *"add a garage to house #3"* — you check your notebook, find house #3's details, and make only that change.

**Terraform's state file is that notebook.** Without it, Terraform would have no idea what it already built, and would try to build everything from scratch every time.

```
Your .tf files    =  What you WANT
State file        =  What Terraform THINKS exists
Real cloud        =  What ACTUALLY exists

Terraform's job   =  Make real cloud match what you want
                     using state as the reference point
```

---

### Why Does State Exist?

```
WITHOUT STATE (hypothetical):
──────────────────────────────
terraform apply → Terraform calls AWS API:
"Does bucket 'my-app-bucket' exist?"
AWS: "Yes"
Terraform: "Then I'll skip it" ✅

But how does Terraform know it CREATED that bucket?
And what if the bucket has 200 other attributes?
And what about resources it can't easily query?
This approach would be impossibly slow and incomplete.

WITH STATE:
──────────────────────────────
terraform apply → Terraform checks state file:
"I created bucket 'my-app-bucket' with ID 'my-app-bucket',
 ARN 'arn:aws:s3:::my-app-bucket', created at '2024-01-15'"
Terraform: "Compare desired vs state → No changes needed" ✅
Fast, complete, reliable.
```

---

### Real-World Analogy

```
STATE FILE is like your BANK STATEMENT

Bank Statement            Terraform State
─────────────────────     ──────────────────────────────
Lists all transactions  → Lists all resources created
Shows current balance   → Shows current resource attributes
Used to detect fraud    → Used to detect infrastructure drift
Shared with accountant  → Shared with team via remote backend
Must be accurate        → Must reflect real infrastructure
```

---

### What Does the State File Look Like?

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 14,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {
    "vpc_id": {
      "value": "vpc-0a1b2c3d4e5f67890",
      "type": "string"
    }
  },
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "arn": "arn:aws:ec2:us-east-1:123456789:vpc/vpc-0a1b2c3d4e5f67890",
            "cidr_block": "10.0.0.0/16",
            "id": "vpc-0a1b2c3d4e5f67890",
            "enable_dns_hostnames": true,
            "tags": {
              "Environment": "prod",
              "Name": "main-vpc"
            }
          },
          "sensitive_attributes": [],
          "private": "base64encodedprivatedata=="
        }
      ]
    }
  ]
}
```

**Key fields:**
- `serial` — increments every time state changes (used for conflict detection)
- `lineage` — unique ID for this state file (never changes)
- `resources` — every managed resource with ALL its attributes
- `outputs` — all output values
- `sensitive_attributes` — which attributes are marked sensitive

---

### Basic State Commands

```bash
# See what's in your state
terraform state list

# See details of one resource
terraform state show aws_vpc.main

# See all outputs
terraform output

# Pull remote state to local (read-only inspection)
terraform state pull

# Refresh state from real cloud (sync)
terraform refresh   # Deprecated in TF 1.x — use terraform apply -refresh-only
```

---

### Common Beginner Mistakes

```bash
# ❌ MISTAKE 1: Committing state to Git
# .gitignore MUST contain:
*.tfstate
*.tfstate.*
.terraform/

# ❌ MISTAKE 2: Multiple people applying at the same time
# Person A: terraform apply   (starts)
# Person B: terraform apply   (starts)
# Both read same state, both make changes → STATE CORRUPTION
# Solution: Use remote backend with state locking

# ❌ MISTAKE 3: Manually editing the state file
# Never edit .tfstate with a text editor
# Use: terraform state mv, terraform state rm, terraform import

# ❌ MISTAKE 4: Deleting state file to "start fresh"
# This makes Terraform think nothing exists
# Next apply tries to create everything → duplicate resources → errors

# ❌ MISTAKE 5: Using local state in production
# Local state = only on your laptop
# Teammate has no access, no locking, no backup
```

---

## 🟡 INTERMEDIATE LEVEL

### Local vs Remote State

```
LOCAL STATE (default)           REMOTE STATE (production)
──────────────────────          ──────────────────────────────
File: terraform.tfstate         File: S3, GCS, Azure Blob, TFC
Location: your machine          Location: shared cloud storage
Team access: ❌ None             Team access: ✅ Everyone
Locking: ❌ None                 Locking: ✅ DynamoDB/native
Backup: ❌ Manual                Backup: ✅ Versioned automatically
Encryption: ❌ Plaintext         Encryption: ✅ At rest + transit
CI/CD: ❌ Can't access           CI/CD: ✅ Accessible
Use for: Dev/learning only       Use for: ALL real projects
```

---

### Setting Up Remote State — S3 Backend (Most Common)

**Step 1: Create the state infrastructure (bootstrap)**

```hcl
# bootstrap/main.tf — Run this ONCE manually before everything else

provider "aws" {
  region = "us-east-1"
}

# ── S3 Bucket for State Storage ───────────────────────────────────
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state-prod"

  # Prevent accidental deletion
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = "Terraform State Storage"
    Purpose = "terraform-state"
  }
}

# Enable versioning — keeps history of every state change
resource "aws_s3_bucket_versioning" "state_versioning" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Encrypt state at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "state_encryption" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"   # Use KMS for stronger encryption
    }
  }
}

# Block ALL public access to state bucket
resource "aws_s3_bucket_public_access_block" "state_public_access" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable access logging
resource "aws_s3_bucket_logging" "state_logging" {
  bucket        = aws_s3_bucket.terraform_state.id
  target_bucket = aws_s3_bucket.terraform_state.id
  target_prefix = "state-access-logs/"
}

# ── DynamoDB for State Locking ────────────────────────────────────
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"   # No capacity planning needed
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  # Point-in-time recovery for the lock table
  point_in_time_recovery {
    enabled = true
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name    = "Terraform State Lock Table"
    Purpose = "terraform-state-locking"
  }
}

# ── Outputs ───────────────────────────────────────────────────────
output "state_bucket_name" {
  value = aws_s3_bucket.terraform_state.bucket
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

**Step 2: Configure backend in your project**

```hcl
# backend.tf — In your actual project
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state-prod"
    key            = "services/web-app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"

    # Optional: assume role for cross-account
    role_arn       = "arn:aws:iam::123456789:role/TerraformStateRole"
  }
}
```

**Backend key naming strategy:**

```
bucket: mycompany-terraform-state
│
├── global/
│   └── iam/terraform.tfstate
│
├── services/
│   ├── web-app/
│   │   ├── dev/terraform.tfstate
│   │   ├── staging/terraform.tfstate
│   │   └── prod/terraform.tfstate
│   └── api/
│       ├── dev/terraform.tfstate
│       └── prod/terraform.tfstate
│
└── shared/
    ├── networking/terraform.tfstate
    └── monitoring/terraform.tfstate
```

---

### State Locking — How It Works

```
WITHOUT LOCKING (danger):
─────────────────────────────────────────────────────────────────
Engineer A: terraform apply    ──────────────────────────►
                               reads state (serial=5)
Engineer B: terraform apply         ──────────────────────►
                                    reads state (serial=5)
Engineer A: writes state      ──────────────────────────►  serial=6
Engineer B: writes state           ──────────────────────►  serial=6
                                                         CONFLICT!
                                                         STATE CORRUPTED


WITH LOCKING (safe):
─────────────────────────────────────────────────────────────────
Engineer A: terraform apply    ──────────────────────────►
                               Acquires DynamoDB lock    ✅ LOCKED
Engineer B: terraform apply         ─────────►
                                    Tries to get lock    ❌ LOCKED
                                    Error: "state is locked"
                                    (shows who holds lock + when)
Engineer A: apply completes   ──────────────────────────►
                               Releases DynamoDB lock    🔓 UNLOCKED
Engineer B: retry              ─────────────────────────────────►
                               Acquires lock             ✅ LOCKED
```

```bash
# What a lock error looks like:
# Error: Error acquiring the state lock
# 
# Error message: ConditionalCheckFailedException: ...
# Lock Info:
#   ID:        abc123def456
#   Path:      mycompany-terraform-state/services/web-app/terraform.tfstate
#   Operation: OperationTypeApply
#   Who:       alice@mycompany.com
#   Version:   1.6.0
#   Created:   2024-01-15 14:30:00 +0000 UTC
#   Info:
#
# Terraform acquires a state lock to protect the state from being
# written by multiple users at the same time.

# Force unlock (ONLY if you're 100% sure the lock is stale)
terraform force-unlock LOCK_ID_HERE
```

---

### State Commands — Complete Reference

```bash
# ── INSPECTION ────────────────────────────────────────────────────

# List all resources in state
terraform state list

# List with filter
terraform state list aws_instance.*
terraform state list module.networking.*

# Show all attributes of a resource
terraform state show aws_instance.web
terraform state show 'aws_instance.web[0]'          # count resource
terraform state show 'aws_instance.web["prod"]'     # for_each resource
terraform state show 'module.networking.aws_vpc.main'  # inside module

# Download state to local (for inspection)
terraform state pull > current_state.json

# Push modified state (DANGEROUS — use with extreme caution)
terraform state push modified_state.json


# ── MODIFICATION ──────────────────────────────────────────────────

# Rename a resource in state (when you rename in .tf too)
terraform state mv aws_instance.old_name aws_instance.new_name

# Move resource between modules
terraform state mv aws_instance.web module.compute.aws_instance.web

# Move resource to/from count/for_each
terraform state mv 'aws_instance.web[0]' aws_instance.web_primary

# Remove resource from state (stops managing it, doesn't destroy it)
terraform state rm aws_instance.web

# Remove module from state
terraform state rm module.networking


# ── IMPORT ────────────────────────────────────────────────────────

# Import existing resource into state (CLI method)
terraform import aws_instance.web i-1234567890abcdef0
terraform import 'aws_instance.web[0]' i-1234567890abcdef0
terraform import 'module.compute.aws_instance.web' i-1234567890abcdef0


# ── REFRESH ───────────────────────────────────────────────────────

# Check for drift without making changes (TF 1.x+)
terraform apply -refresh-only
terraform plan -refresh-only   # Just show what drifted

# Old way (deprecated but still works)
terraform refresh
```

---

### State Manipulation Scenarios

```bash
# ── SCENARIO 1: Renaming a resource ──────────────────────────────
# You had: resource "aws_instance" "server" { }
# You want: resource "aws_instance" "web_server" { }
# Without help: Terraform destroys server, creates web_server (DOWNTIME)

# Solution A: moved block in .tf (TF 1.1+) — PREFERRED
moved {
  from = aws_instance.server
  to   = aws_instance.web_server
}

# Solution B: CLI state mv
terraform state mv aws_instance.server aws_instance.web_server
# Then rename in .tf file
# Then terraform plan — should show NO changes


# ── SCENARIO 2: Moving resource into a module ──────────────────────
# You had: resource "aws_vpc" "main" { } in root
# You want: module "networking" { } which contains aws_vpc.main

# Step 1: Move in state
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# Step 2: Move/refactor .tf code into module
# Step 3: terraform plan — should show no changes (just refactoring)


# ── SCENARIO 3: Removing a resource from state (adopt externally) ──
# Resource exists in cloud, you don't want Terraform to manage it anymore
terraform state rm aws_s3_bucket.legacy
# Now terraform plan won't show this bucket
# The bucket still EXISTS in AWS — Terraform just won't track it


# ── SCENARIO 4: Importing an existing resource ────────────────────
# A bucket was created manually in AWS, you want Terraform to manage it

# Step 1: Write the resource block in .tf
resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}

# Step 2: Import it
terraform import aws_s3_bucket.existing my-existing-bucket

# Step 3: Run plan — if config matches reality, no changes
terraform plan

# Step 4: Fix any config drift shown in plan


# ── SCENARIO 5: Recovering from accidental state rm ───────────────
# You ran: terraform state rm aws_db_instance.production  (oops!)
# The DB still exists in AWS but Terraform forgot about it

# Option A: Re-import it
terraform import aws_db_instance.production myapp-prod-db

# Option B: Restore from S3 versioned state
# Go to S3 console → state bucket → state file → Show Versions
# Download previous version → terraform state push previous_version.json
```

---

### Remote State as Data Source

```hcl
# Read outputs from ANOTHER Terraform project's state
# This is how separate teams share infrastructure information

# team-A/networking/main.tf creates VPC and outputs its ID
# team-B/app/main.tf needs to deploy into that VPC

# team-B/app/data.tf
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "mycompany-terraform-state"
    key    = "shared/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Now use networking outputs as if they were local
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  # Reference networking team's VPC and subnet
  subnet_id = data.terraform_remote_state.networking.outputs.public_subnet_ids[0]

  vpc_security_group_ids = [
    data.terraform_remote_state.networking.outputs.app_security_group_id
  ]
}
```

---

### Interview-Focused Points 🎯

1. **What is the `serial` number in state?** An integer that increments on every state change. Used to detect conflicts — if two applies try to write the same serial, one fails.

2. **What is `lineage` in state?** A UUID that uniquely identifies a state file's ancestry. If lineages don't match, Terraform refuses to push state (prevents overwriting unrelated state).

3. **What happens during `terraform plan` with remote state?** Terraform downloads the state, calls cloud APIs to refresh resource attributes, builds a diff, then releases state. No lock is held during plan (only during apply).

4. **How does DynamoDB locking work?** Terraform writes a conditional DynamoDB item with the `LockID` key. If the item already exists (another process locked it), the conditional write fails. On unlock, the item is deleted.

5. **When would you use `terraform state rm`?** When you want to stop managing a resource with Terraform but keep it running in the cloud. Common for legacy resources or when splitting state between multiple configs.

6. **What's the difference between `terraform refresh` and `terraform apply -refresh-only`?** `refresh` directly modifies state to match cloud reality. `-refresh-only` shows you what drifted and asks for confirmation. The `-refresh-only` approach is safer.

---

## 🔴 ADVANCED LEVEL

### How State Works Internally — Complete Flow

```
TERRAFORM APPLY — FULL INTERNAL FLOW WITH STATE
─────────────────────────────────────────────────────────────────

1. LOAD CONFIGURATION
   Parse all .tf files → build resource graph

2. LOAD STATE
   If remote: download state file from backend
   Acquire lock (write to DynamoDB)
   Read serial number (e.g., serial = 14)

3. REFRESH RESOURCES (unless -refresh=false)
   For each resource in state:
     Call provider.ReadResource(current_state_attributes)
     Get actual cloud attributes
     Update in-memory state (not written to file yet)

4. BUILD DIFF (Plan)
   For each resource:
     Desired (from .tf) vs Current (from refreshed state)
     Determine: CREATE / UPDATE / REPLACE / DELETE / NO-OP
     Mark unknown values as "known after apply"

5. EXECUTE CHANGES (Apply)
   Walk dependency graph in order:
   For CREATE:
     Call provider.ApplyResourceChange() → cloud API
     Get response with all attributes (including computed ones)
     Write resource to state immediately (even if other resources fail)
   For UPDATE: similar
   For DELETE: similar

6. WRITE STATE
   Increment serial (serial = 15)
   Write complete new state to backend
   State includes ALL resource attributes (even sensitive ones)

7. RELEASE LOCK
   Delete DynamoDB lock item

8. SHOW OUTPUTS
   Display output values to user
```

---

### State Storage Backends — Comparison

```
┌─────────────────┬──────────┬─────────┬───────────┬────────────┐
│ Backend         │ Locking  │ Encrypt │ Versioned │ Best For   │
├─────────────────┼──────────┼─────────┼───────────┼────────────┤
│ local           │ ❌ None  │ ❌ No   │ ❌ No     │ Dev/Learn  │
│ s3 + dynamodb   │ ✅ Yes   │ ✅ Yes  │ ✅ Yes    │ AWS teams  │
│ gcs             │ ✅ Yes   │ ✅ Yes  │ ✅ Yes    │ GCP teams  │
│ azurerm         │ ✅ Yes   │ ✅ Yes  │ ✅ Yes    │ Azure teams│
│ Terraform Cloud │ ✅ Yes   │ ✅ Yes  │ ✅ Yes    │ Any team   │
│ consul          │ ✅ Yes   │ ⚠️ TLS  │ ❌ No     │ HashiStack │
│ http            │ ⚠️ Opt.  │ ⚠️ HTTPS│ ❌ No     │ Custom     │
│ kubernetes      │ ✅ Yes   │ ⚠️ Opt. │ ❌ No     │ K8s teams  │
└─────────────────┴──────────┴─────────┴───────────┴────────────┘
```

---

### Workspaces — One Backend, Multiple States

```hcl
# What workspaces are:
# One backend config, multiple independent state files
# Each workspace = isolated state

# Default workspace is always "default"
```

```bash
# Create and switch workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# List workspaces
terraform workspace list
# * default
#   dev
#   prod
#   staging

# Switch workspace
terraform workspace select prod

# Show current workspace
terraform workspace show
# prod

# Delete workspace (must switch away first)
terraform workspace select default
terraform workspace delete dev
```

```hcl
# Use workspace in config
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t2.large" : "t2.micro"

  tags = {
    Environment = terraform.workspace
  }
}

# Workspace-specific variables
variable "instance_counts" {
  type = map(number)
  default = {
    default = 1
    dev     = 1
    staging = 2
    prod    = 5
  }
}

resource "aws_instance" "web" {
  count = lookup(var.instance_counts, terraform.workspace, 1)
}
```

**State file locations with workspaces:**
```
S3 bucket:
├── env:/
│   ├── dev/
│   │   └── services/web-app/terraform.tfstate
│   ├── staging/
│   │   └── services/web-app/terraform.tfstate
│   └── prod/
│       └── services/web-app/terraform.tfstate
└── services/web-app/terraform.tfstate    ← default workspace
```

**⚠️ Workspace Warning — When NOT to Use:**
```
❌ DON'T use workspaces for strict environment separation when:
   - Environments have different backends
   - Environments have different providers/accounts
   - Different teams manage different environments
   - You need environment-specific IAM permissions

✅ DO use workspaces for:
   - Testing changes in isolated state
   - Feature branch infrastructure
   - Same config, truly identical environments
```

---

### State Migration — Moving Between Backends

```bash
# SCENARIO: Moving from local state to S3 backend
# Step 1: You have local state, everything is working

# Step 2: Add backend config to your .tf
# backend.tf:
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "services/web-app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

# Step 3: Run init — Terraform detects backend change
terraform init

# Terraform asks:
# "Do you want to copy existing state to the new backend?"
# Answer: yes

# Step 4: Verify state was migrated
terraform state list   # Should show all resources

# Step 5: Remove local state file (it's now in S3)
rm terraform.tfstate
rm terraform.tfstate.backup
```

```bash
# SCENARIO: Moving from one S3 path to another
# Useful when reorganizing state layout

# Method 1: Change backend config and run init
# Terraform will detect key change and ask to migrate

# Method 2: Manual migration (more control)
# Step 1: Pull old state
terraform state pull > old_state.json

# Step 2: Update backend config to new location

# Step 3: Push to new location
terraform init  # Initialize new backend
terraform state push old_state.json
```

---

### Handling State Drift

```
DRIFT = When real cloud state differs from Terraform state

Types of drift:
────────────────────────────────────────────────────────
1. ADDITION drift   — Resource created outside Terraform
2. DELETION drift   — Resource deleted outside Terraform
3. MUTATION drift   — Resource modified outside Terraform
```

```bash
# Detect drift without changing anything
terraform plan -refresh-only

# Output shows:
# ~ aws_instance.web (drift detected)
#   ~ tags = {
#       + "LastModifiedBy" = "console-user"    ← Added externally
#       - "Environment"    = "prod"             ← Removed externally
#     }

# Options when drift detected:
# 1. Accept drift (update state to match reality)
terraform apply -refresh-only

# 2. Reject drift (next apply will revert to your config)
terraform apply   # Terraform will revert the external changes

# 3. Update your config to accept the drift
#    Modify .tf file to match current cloud state
#    Then terraform plan shows no changes
```

```hcl
# PREVENT drift with lifecycle ignore_changes
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"

  lifecycle {
    # Ignore if someone manually changes these
    ignore_changes = [
      tags["LastModifiedBy"],    # Auto-managed tag
      tags["PatchGroup"],        # Set by Systems Manager
      user_data,                 # Modified by automation
    ]
  }
}
```

---

### State Security — Production Hardening

```hcl
# ── 1. Encrypt state with customer-managed KMS key ────────────────
resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowTerraformRole"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::123456789:role/TerraformExecutionRole"
        }
        Action   = ["kms:Encrypt", "kms:Decrypt", "kms:GenerateDataKey"]
        Resource = "*"
      }
    ]
  })
}

# ── 2. Restrict state bucket access with bucket policy ────────────
resource "aws_s3_bucket_policy" "state_access" {
  bucket = aws_s3_bucket.terraform_state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyUnencryptedObjectUploads"
        Effect = "Deny"
        Principal = "*"
        Action = "s3:PutObject"
        Resource = "${aws_s3_bucket.terraform_state.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid    = "DenyNonTLS"
        Effect = "Deny"
        Principal = "*"
        Action = "s3:*"
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
        Condition = {
          Bool = { "aws:SecureTransport" = "false" }
        }
      },
      {
        Sid    = "AllowTerraformRole"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::123456789:role/TerraformExecutionRole"
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
      }
    ]
  })
}

# ── 3. Audit all state access with CloudTrail ─────────────────────
resource "aws_cloudtrail" "state_audit" {
  name           = "terraform-state-audit"
  s3_bucket_name = aws_s3_bucket.cloudtrail_logs.id

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["${aws_s3_bucket.terraform_state.arn}/"]
    }
  }
}
```

---

### State Recovery — Disaster Scenarios

```bash
# ══════════════════════════════════════════════════════════════════
# SCENARIO 1: State file accidentally deleted
# ══════════════════════════════════════════════════════════════════

# If using S3 versioning:
# 1. Go to S3 console
# 2. Navigate to state file path
# 3. Click "Show versions"
# 4. Find last good version
# 5. Download it
# 6. Push it back:
terraform state push recovered_state.json

# If no versioning (lesson learned):
# 1. Create empty state or use existing
# 2. Re-import every resource manually:
terraform import aws_vpc.main vpc-0a1b2c3d4e5f67890
terraform import aws_subnet.public subnet-0abc123def456789
terraform import aws_instance.web i-1234567890abcdef0
# ... tedious but recoverable


# ══════════════════════════════════════════════════════════════════
# SCENARIO 2: State is locked and nobody is running apply
# (Stale lock from crashed apply)
# ══════════════════════════════════════════════════════════════════

# 1. Get lock ID from error message
# 2. Verify the lock holder is not actually running anything
# 3. Force unlock:
terraform force-unlock abc123-lock-id-here

# You can also check DynamoDB directly:
aws dynamodb scan --table-name terraform-state-locks


# ══════════════════════════════════════════════════════════════════
# SCENARIO 3: State corruption (invalid JSON)
# ══════════════════════════════════════════════════════════════════

# 1. Download current (corrupted) state for backup
terraform state pull > corrupted_state_backup.json

# 2. Download previous version from S3
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key services/web-app/terraform.tfstate \
  --version-id "previous-version-id" \
  previous_good_state.json

# 3. Push previous good state
terraform state push previous_good_state.json

# 4. Run plan to check what changed between versions
terraform plan


# ══════════════════════════════════════════════════════════════════
# SCENARIO 4: Two separate state files managing same resources
# (Two teams accidentally managing same AWS resources)
# ══════════════════════════════════════════════════════════════════

# 1. Identify which state should "win"
# 2. Remove duplicate from losing state:
terraform state rm aws_vpc.shared  # Remove from team B's state
# 3. Team A's state now owns it exclusively
# 4. Update Team A's .tf to have the definitive config


# ══════════════════════════════════════════════════════════════════
# SCENARIO 5: terraform apply partially failed
# Some resources created, some didn't
# ══════════════════════════════════════════════════════════════════

# State is updated for each resource as it completes
# So state reflects what DID succeed

# 1. Review what's in state:
terraform state list

# 2. Run plan to see what still needs to be created:
terraform plan

# 3. Run apply again — Terraform only creates missing resources:
terraform apply

# 4. If a specific resource keeps failing, target others first:
terraform apply -target=aws_s3_bucket.logs -target=aws_iam_role.app
```

---

### Performance Considerations

```bash
# ── Large state files (1000+ resources) ──────────────────────────

# Problem: Plan takes 15+ minutes refreshing all resources
# Solution 1: Skip refresh when you know nothing changed externally
terraform plan -refresh=false

# Solution 2: Target specific resources for faster iteration
terraform plan -target=module.web_tier
terraform apply -target=aws_instance.web

# Solution 3: Split into multiple smaller state files
# Instead of one giant state for all of production:
# ├── networking/terraform.tfstate      (50 resources)
# ├── databases/terraform.tfstate       (30 resources)
# ├── compute/terraform.tfstate         (100 resources)
# └── monitoring/terraform.tfstate      (40 resources)


# ── Parallel resource operations ─────────────────────────────────
# By default Terraform creates 10 resources in parallel
# Increase for faster apply (careful with API rate limits)
terraform apply -parallelism=20

# Decrease if hitting API rate limits
terraform apply -parallelism=5


# ── State file size optimization ──────────────────────────────────
# State stores EVERYTHING including large computed values
# Check state size:
terraform state pull | wc -c

# Analyze state structure:
terraform state pull | jq '.resources | length'         # resource count
terraform state pull | jq '.resources[].type' | sort | uniq -c | sort -rn
```

---

### Advanced State Patterns — Production Architecture

```hcl
# ── PATTERN: Partial backend configuration ───────────────────────
# Useful when backend config changes per environment
# Don't hardcode bucket name in code

# backend.tf
terraform {
  backend "s3" {
    # Partial config — only stable values here
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    # bucket and key provided at init time via -backend-config
  }
}
```

```bash
# Initialize with environment-specific values
terraform init \
  -backend-config="bucket=mycompany-terraform-state-prod" \
  -backend-config="key=services/web-app/prod/terraform.tfstate"

terraform init \
  -backend-config="bucket=mycompany-terraform-state-dev" \
  -backend-config="key=services/web-app/dev/terraform.tfstate"

# Or use a backend config file
# backends/prod.hcl:
# bucket = "mycompany-terraform-state-prod"
# key    = "services/web-app/prod/terraform.tfstate"

terraform init -backend-config="backends/prod.hcl"
```

```hcl
# ── PATTERN: State-based cross-team data sharing ─────────────────

# networking team outputs.tf
output "vpc_id"              { value = aws_vpc.main.id }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "nat_gateway_ip"      { value = aws_eip.nat.public_ip }

# app team data.tf — reads networking state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "shared/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# app team main.tf
resource "aws_ecs_service" "app" {
  # Uses networking team's outputs
  network_configuration {
    subnets         = data.terraform_remote_state.networking.outputs.private_subnet_ids
    security_groups = [aws_security_group.app.id]
  }
}
```

---

### Debugging State Issues

```bash
# ── Enable trace-level logging for state operations ───────────────
export TF_LOG=TRACE
export TF_LOG_PATH=./terraform_state_debug.log
terraform apply 2>&1 | grep -i "state\|lock\|serial"

# ── Validate state integrity ──────────────────────────────────────
# Check if state is valid JSON
terraform state pull | python3 -m json.tool > /dev/null && echo "✅ Valid JSON"

# Count resources in state
terraform state pull | jq '.resources | length'

# Find specific resource by attribute value
terraform state pull | jq '.resources[] | select(.instances[].attributes.id == "vpc-123abc")'

# ── Check state version compatibility ────────────────────────────
terraform state pull | jq '{version: .version, terraform_version: .terraform_version, serial: .serial}'

# ── Compare state to live cloud ───────────────────────────────────
terraform plan -refresh-only -detailed-exitcode
# Exit code 0 = no drift
# Exit code 1 = error
# Exit code 2 = drift detected (changes to be made)

# ── Audit state changes over time (with S3 versioning) ───────────
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix services/web-app/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified,Size:Size}' \
  --output table
```

---

## 📁 CODE WRITING GUIDANCE

### Complete Production State Setup

```
state-infrastructure/          ← Bootstrap project (run ONCE)
├── main.tf                    ← S3 bucket, DynamoDB, KMS
├── outputs.tf                 ← Bucket name, table name
└── README.md                  ← How to bootstrap

my-project/
├── backend.tf                 ← Backend config (S3 + DynamoDB)
├── versions.tf                ← Terraform + provider versions
├── providers.tf               ← Provider config
├── main.tf                    ← Resources
├── variables.tf
├── outputs.tf
├── backends/                  ← Partial backend configs per env
│   ├── dev.hcl
│   ├── staging.hcl
│   └── prod.hcl
└── .gitignore                 ← Must exclude *.tfstate
```

```hcl
# backends/prod.hcl
bucket         = "mycompany-terraform-state-prod"
key            = "services/web-app/prod/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-locks"
role_arn       = "arn:aws:iam::111111111111:role/TerraformRole"
```

```hcl
# backends/dev.hcl
bucket         = "mycompany-terraform-state-dev"
key            = "services/web-app/dev/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "terraform-state-locks"
role_arn       = "arn:aws:iam::222222222222:role/TerraformRole"
```

```bash
# .gitignore — CRITICAL for state security
# ── Terraform State ───────────────────────
*.tfstate
*.tfstate.*
*.tfstate.backup

# ── Terraform Working Directory ───────────
.terraform/
.terraform.lock.hcl    # Debate: many teams DO commit this

# ── Variable Files with Secrets ──────────
*.tfvars
!example.tfvars
!*.auto.tfvars.example

# ── Plan Files ───────────────────────────
*.tfplan
*.plan

# ── Override Files ────────────────────────
override.tf
override.tf.json
*_override.tf

# ── Crash Logs ────────────────────────────
crash.log
crash.*.log
```

---

## 🧪 HANDS-ON LAB

### Exercise: Complete State Management Setup

**Goal:** Set up a production-grade state backend from scratch with:
1. S3 bucket with versioning, encryption, and public access blocking
2. DynamoDB table for state locking
3. S3 bucket policy restricting access
4. A second Terraform config that uses the backend
5. Demonstrate `state list`, `state show`, `state mv`

**Expected Flow:**
```bash
# Step 1: Bootstrap state infrastructure
cd state-bootstrap/
terraform init && terraform apply

# Step 2: Configure main project with remote backend
cd ../my-app/
terraform init -backend-config="../backends/dev.hcl"
terraform apply

# Step 3: Practice state commands
terraform state list
terraform state show aws_vpc.main
terraform state mv aws_vpc.main aws_vpc.primary

# Step 4: Verify no changes after mv
terraform plan   # Should show: No changes
```

**Try it yourself first!**

---

### ✅ Solution

```hcl
# state-bootstrap/main.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
}

provider "aws" { region = "us-east-1" }

locals {
  name    = "mycompany-terraform-state"
  region  = "us-east-1"
}

# ── S3 Bucket ─────────────────────────────────────────────────────
resource "aws_s3_bucket" "state" {
  bucket = local.name
  lifecycle { prevent_destroy = true }
  tags = { Name = "Terraform State", Purpose = "terraform-state" }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "state" {
  bucket                  = aws_s3_bucket.state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# ── DynamoDB Lock Table ───────────────────────────────────────────
resource "aws_dynamodb_table" "state_lock" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle { prevent_destroy = true }
  tags = { Name = "Terraform State Lock" }
}

# ── Outputs ───────────────────────────────────────────────────────
output "state_bucket"   { value = aws_s3_bucket.state.bucket }
output "lock_table"     { value = aws_dynamodb_table.state_lock.name }
```

```hcl
# my-app/backend.tf
terraform {
  backend "s3" {
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
    # bucket and key provided via -backend-config
  }
}
```

```hcl
# my-app/versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 5.0" }
  }
}
```

```hcl
# my-app/main.tf
provider "aws" { region = "us-east-1" }

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = { Name = "lab-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  tags = { Name = "lab-public-subnet" }
}
```

```hcl
# my-app/outputs.tf
output "vpc_id"    { value = aws_vpc.main.id }
output "subnet_id" { value = aws_subnet.public.id }
```

```hcl
# backends/dev.hcl
bucket = "mycompany-terraform-state"
key    = "lab/my-app/dev/terraform.tfstate"
```

```bash
# Run commands:
cd state-bootstrap && terraform init && terraform apply
cd ../my-app
terraform init -backend-config="../backends/dev.hcl"
terraform apply

# State commands practice:
terraform state list
# aws_subnet.public
# aws_vpc.main

terraform state show aws_vpc.main
# Shows all VPC attributes

terraform state mv aws_vpc.main aws_vpc.primary
# aws_vpc.main now tracked as aws_vpc.primary in state

# Add moved block to .tf to match:
# moved { from = aws_vpc.main; to = aws_vpc.primary }
# OR rename in main.tf and run plan (should show no changes)
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| State file | Terraform's source of truth — maps config to real resources |
| `serial` | Increments every change — used for conflict detection |
| `lineage` | UUID for state identity — never changes |
| Remote state | Required for teams — S3+DynamoDB is the AWS standard |
| State locking | DynamoDB prevents concurrent applies from corrupting state |
| `terraform state mv` | Rename resources in state without destroy/recreate |
| `terraform state rm` | Stop managing resource — it stays in cloud |
| `terraform import` | Bring existing cloud resource into Terraform management |
| `-refresh-only` | Detect/accept drift without applying config changes |
| Workspaces | One backend, multiple isolated states (use carefully) |
| Remote state data | Share state outputs between separate Terraform projects |

---

### Quick Command Reference

```bash
# Inspection
terraform state list                    # All resources
terraform state list 'module.name.*'   # Filter by module
terraform state show resource.name      # Full details
terraform state pull                    # Download state JSON
terraform state pull | jq '.'          # Pretty print

# Modification
terraform state mv OLD NEW              # Rename/move resource
terraform state rm resource.name        # Remove from state
terraform state push file.json          # Upload state file

# Import
terraform import resource.name CLOUD_ID

# Drift
terraform plan -refresh-only            # Detect drift
terraform apply -refresh-only           # Accept drift

# Locking
terraform force-unlock LOCK_ID          # Release stale lock

# Workspace
terraform workspace list
terraform workspace new NAME
terraform workspace select NAME
terraform workspace show
terraform workspace delete NAME

# Init with backend config
terraform init -backend-config="file.hcl"
terraform init -backend-config="key=value"
terraform init -migrate-state           # Migrate between backends
```

---

### Top 10 Interview Questions

1. **Why does Terraform need state?** State maps your config to real cloud resources, stores computed attributes (IDs, IPs, ARNs), enables change detection, and enables team collaboration.

2. **What happens if two people run `terraform apply` simultaneously?** Without locking, state corruption. With DynamoDB locking, the second person gets an error showing who holds the lock and when it was acquired.

3. **How do you recover from a corrupted state file?** Restore a previous version from S3 versioning, then push it with `terraform state push`. This is why S3 versioning is mandatory.

4. **What is the difference between `terraform state rm` and `terraform destroy`?** `state rm` removes the resource from Terraform's tracking only — the cloud resource continues running. `destroy` actually deletes the cloud resource AND removes it from state.

5. **How do separate teams share infrastructure information?** Via `terraform_remote_state` data source — one team's outputs become another team's data source inputs.

6. **What does `terraform apply -refresh-only` do?** Detects drift between state and real cloud, then optionally updates state to match reality — without applying your config changes.

7. **When should you NOT use Terraform workspaces?** When environments need different IAM permissions, different backends, or are managed by different teams. Use separate directories/configs instead.

8. **How do you rename a resource without destroying it?** Either use a `moved` block in `.tf` (TF 1.1+) or run `terraform state mv old_name new_name` followed by renaming in your `.tf` file.

9. **What is partial backend configuration and why use it?** Defining only stable backend values in code (region, encryption) and passing env-specific values (bucket, key) via `-backend-config` at init time. Allows same code for multiple environments.

10. **How does Terraform handle a resource that was manually deleted from the cloud?** On next `plan`, Terraform detects the resource is gone (not in cloud but is in state), and plans to recreate it. Running `apply` recreates it.

---
