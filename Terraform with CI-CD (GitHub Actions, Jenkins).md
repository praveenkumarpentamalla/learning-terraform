# 🚀 TOPIC 7: Terraform with CI/CD — GitHub Actions, Jenkins & GitLab

---

## 🟢 BEGINNER LEVEL

### What is Terraform CI/CD? (Simple Terms)

Imagine you're working at a **bank**. Every time a teller wants to change something — open a new account type, modify interest rates — they can't just walk into the vault and do it themselves. They:

1. **Write a request** (fill out a form)
2. **Get it reviewed** by a supervisor
3. **Get approved** by the branch manager
4. **Changes are logged** in an audit trail
5. **Only then** does the system execute the change

**Terraform CI/CD is exactly that process for infrastructure.** Instead of engineers running `terraform apply` from their laptops (walking into the vault alone), every change goes through:

```
Write code → Open PR → Automated plan runs → Team reviews → 
Approved → Pipeline applies → Audit trail created
```

---

### Why CI/CD for Terraform?

```
WITHOUT CI/CD (dangerous):                WITH CI/CD (safe):
────────────────────────────              ──────────────────────────────
Engineer A applies from laptop            ALL changes go through pipeline
No visibility to teammates                Team sees EVERY planned change
"It worked on my machine"                 Consistent environment always
No audit trail                            Full audit trail in Git history
Prod credentials on laptops               Credentials ONLY in CI/CD system
Anyone can apply anytime                  Requires PR approval + review
Manual, error-prone                       Automated, consistent, reliable
```

---

### Real-World Analogy

```
Terraform CI/CD is like a BANK'S CHANGE CONTROL PROCESS:

Developer's PR     =  Change Request Form
terraform plan     =  Impact Assessment ("here's what will change")
Code Review        =  Supervisor Review
PR Approval        =  Management Sign-off
terraform apply    =  Executing the approved change
Git history        =  Audit Trail
Pipeline logs      =  Transaction Receipt
Environment gates  =  Escalating approvals (teller → supervisor → manager)
```

---

### The Core Pipeline Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                    TERRAFORM CI/CD PIPELINE                      │
│                                                                  │
│  Developer          Pipeline                    Cloud            │
│  ─────────          ────────                    ─────            │
│  git push    ──►   checkout code                                 │
│                    terraform init                                │
│                    terraform validate                            │
│                    terraform fmt --check                         │
│                    terraform plan    ──────────► AWS (read-only) │
│                         │                                        │
│                    Post plan to PR ◄─────── Plan output          │
│                         │                                        │
│  Review plan ◄──── PR Comment                                    │
│  Approve PR  ──►   (gate: approval required)                     │
│                         │                                        │
│                    terraform apply ──────────► AWS (write)       │
│                         │                                        │
│                    Notify team  ◄──────────── Apply output       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

### Authentication — The #1 Concern

```
❌ OLD WAY (Never do this):
─────────────────────────────
Store AWS access keys as CI/CD secrets
AWS_ACCESS_KEY_ID = "AKIAIOSFODNN7EXAMPLE"
AWS_SECRET_ACCESS_KEY = "wJalrXUtnFEMI..."

Problems:
- Keys are long-lived (don't expire automatically)
- If leaked, attacker has permanent access
- Hard to rotate across all pipelines
- Keys in secrets = still a secret management problem


✅ NEW WAY (OIDC — OpenID Connect):
─────────────────────────────────────
No static credentials at all!
CI/CD platform proves its identity to AWS via JWT token
AWS issues temporary credentials (expire in 1 hour)
Nothing to steal, nothing to rotate

GitHub Actions → JWT token → AWS STS → Temporary credentials
```

---

### Minimal GitHub Actions Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'   # Only on main branch
        run: terraform apply -auto-approve
```

---

### Common Beginner Mistakes

```yaml
# ❌ MISTAKE 1: Applying on every push (including PRs)
- name: Terraform Apply
  run: terraform apply -auto-approve   # No condition = runs on PRs too!

# ✅ Fix: Add condition
- name: Terraform Apply
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: terraform apply -auto-approve


# ❌ MISTAKE 2: Not saving plan output — apply can differ from plan
- run: terraform plan
- run: terraform apply -auto-approve  # Might apply different changes!

# ✅ Fix: Save plan and apply the saved plan
- run: terraform plan -out=tfplan
- run: terraform apply tfplan         # Applies EXACTLY what was planned


# ❌ MISTAKE 3: Hardcoding AWS credentials
env:
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"  # Never hardcode!

# ✅ Fix: Use OIDC or GitHub secrets
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}


# ❌ MISTAKE 4: Not running fmt check in CI
# Code inconsistently formatted causes confusing diffs

# ✅ Fix: Add format check
- run: terraform fmt -check -recursive


# ❌ MISTAKE 5: No locking on CI apply
# Two PRs merged quickly → two applies run simultaneously → state corruption

# ✅ Fix: Backend state locking (DynamoDB) handles this automatically
# But also: serialize applies with concurrency groups
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false   # Don't cancel — queue instead
```

---

## 🟡 INTERMEDIATE LEVEL

### Setting Up OIDC Authentication

**Step 1: Create OIDC Provider in AWS (one-time)**

```hcl
# global/iam/github-oidc.tf

# Create OIDC Provider for GitHub Actions
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  # GitHub's OIDC thumbprint
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]
}

# IAM Role that GitHub Actions will assume
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # Restrict to your specific org/repo
            "token.actions.githubusercontent.com:sub" = [
              "repo:myorg/my-repo:ref:refs/heads/main",
              "repo:myorg/my-repo:pull_request"
            ]
          }
        }
      }
    ]
  })
}

# Policy: What GitHub Actions can do
resource "aws_iam_role_policy" "github_actions_terraform" {
  name = "TerraformPermissions"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      # State backend access
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::mycompany-tf-state",
          "arn:aws:s3:::mycompany-tf-state/*"
        ]
      },
      # State locking
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:us-east-1:*:table/terraform-state-locks"
      },
      # Infrastructure permissions (scope to what you actually need)
      {
        Effect   = "Allow"
        Action   = [
          "ec2:*",
          "s3:*",
          "rds:*",
          "iam:*",
          "elasticloadbalancing:*",
          "autoscaling:*",
          "cloudwatch:*"
        ]
        Resource = "*"
      }
    ]
  })
}

output "github_actions_role_arn" {
  value = aws_iam_role.github_actions.arn
}
```

---

### Complete GitHub Actions Pipeline — Production Grade

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'
  pull_request:
    branches: [main]
    paths:
      - 'environments/**'
      - 'modules/**'

# Prevent concurrent applies to same environment
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false   # Queue, don't cancel

env:
  TF_VERSION:   "1.6.0"
  TF_WORK_DIR:  "environments/prod"
  AWS_REGION:   "us-east-1"

permissions:
  id-token:       write   # Required for OIDC
  contents:       read
  pull-requests:  write   # Post plan comments to PR

jobs:
  # ════════════════════════════════════════════════════
  # JOB 1: Validate (runs on every push/PR)
  # ════════════════════════════════════════════════════
  validate:
    name: Validate Terraform
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true     # Don't fail — report instead

      - name: Configure AWS (read-only role for validate)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region:     ${{ env.AWS_REGION }}

      - name: Terraform Init
        id: init
        working-directory: ${{ env.TF_WORK_DIR }}
        run: terraform init -input=false

      - name: Terraform Validate
        id: validate
        working-directory: ${{ env.TF_WORK_DIR }}
        run: terraform validate -no-color

      - name: Run tflint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest

      - name: tflint Init
        working-directory: ${{ env.TF_WORK_DIR }}
        run: tflint --init

      - name: tflint Run
        working-directory: ${{ env.TF_WORK_DIR }}
        run: tflint --format compact


  # ════════════════════════════════════════════════════
  # JOB 2: Plan (runs on PRs)
  # ════════════════════════════════════════════════════
  plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'

    outputs:
      plan_exit_code: ${{ steps.plan.outputs.exit_code }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version:    ${{ env.TF_VERSION }}
          terraform_wrapper:    false    # Disable wrapper for exit code

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region:     ${{ env.AWS_REGION }}

      - name: Terraform Init
        working-directory: ${{ env.TF_WORK_DIR }}
        run: terraform init -input=false

      - name: Terraform Plan
        id: plan
        working-directory: ${{ env.TF_WORK_DIR }}
        run: |
          terraform plan \
            -input=false \
            -no-color \
            -detailed-exitcode \
            -out=tfplan \
            -var-file="terraform.tfvars" \
            2>&1 | tee plan_output.txt

          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true     # Capture exit code, don't fail yet

      - name: Upload Plan Artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ github.sha }}
          path: ${{ env.TF_WORK_DIR }}/tfplan
          retention-days: 5

      - name: Generate Plan Summary
        working-directory: ${{ env.TF_WORK_DIR }}
        run: |
          terraform show -no-color tfplan > plan_summary.txt
          # Extract just the changes summary
          grep -E "^Plan:|will be created|will be destroyed|will be updated|No changes" \
            plan_summary.txt | head -20 > changes_summary.txt || true

      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');

            // Read plan output
            let planOutput = '';
            try {
              planOutput = fs.readFileSync(
                '${{ env.TF_WORK_DIR }}/plan_output.txt', 'utf8'
              );
              // Truncate if too long (PR comment limit)
              if (planOutput.length > 60000) {
                planOutput = planOutput.substring(0, 60000) +
                  '\n\n... (truncated - see full output in Actions logs)';
              }
            } catch(e) {
              planOutput = 'Failed to read plan output';
            }

            const exitCode = '${{ steps.plan.outputs.exit_code }}';
            const statusEmoji = exitCode === '0' ? '✅' :
                                exitCode === '2' ? '⚠️' : '❌';
            const statusText  = exitCode === '0' ? 'No Changes' :
                                exitCode === '2' ? 'Changes Detected' : 'Failed';

            const output = `## Terraform Plan ${statusEmoji} ${statusText}

            **Environment:** \`${{ env.TF_WORK_DIR }}\`
            **Triggered by:** @${{ github.actor }}
            **Commit:** \`${{ github.sha }}\`

            <details>
            <summary>Show Full Plan Output</summary>

            \`\`\`hcl
            ${planOutput}
            \`\`\`

            </details>

            *Plan ran at: ${ new Date().toISOString() }*`;

            // Delete previous plan comments (keep PR clean)
            const { data: comments } = await github.rest.issues.listComments({
              owner:      context.repo.owner,
              repo:       context.repo.repo,
              issue_number: context.issue.number
            });

            for (const comment of comments) {
              if (comment.body.includes('## Terraform Plan')) {
                await github.rest.issues.deleteComment({
                  owner:      context.repo.owner,
                  repo:       context.repo.repo,
                  comment_id: comment.id
                });
              }
            }

            // Post new comment
            await github.rest.issues.createComment({
              owner:        context.repo.owner,
              repo:         context.repo.repo,
              issue_number: context.issue.number,
              body:         output
            });

      - name: Check Plan Exit Code
        if: steps.plan.outputs.exit_code == '1'
        run: |
          echo "❌ Terraform plan failed!"
          exit 1


  # ════════════════════════════════════════════════════
  # JOB 3: Apply (runs ONLY on merge to main)
  # ════════════════════════════════════════════════════
  apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production    # ← GitHub environment (requires approval)

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region:     ${{ env.AWS_REGION }}

      - name: Terraform Init
        working-directory: ${{ env.TF_WORK_DIR }}
        run: terraform init -input=false

      - name: Terraform Plan (fresh plan before apply)
        working-directory: ${{ env.TF_WORK_DIR }}
        run: |
          terraform plan \
            -input=false \
            -out=tfplan \
            -var-file="terraform.tfvars"

      - name: Terraform Apply
        working-directory: ${{ env.TF_WORK_DIR }}
        run: terraform apply -auto-approve tfplan

      - name: Capture Outputs
        id: outputs
        working-directory: ${{ env.TF_WORK_DIR }}
        run: |
          echo "outputs<<EOF" >> $GITHUB_OUTPUT
          terraform output -json >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: Notify Slack on Success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Terraform apply succeeded!",
              "attachments": [{
                "color": "good",
                "fields": [
                  { "title": "Environment", "value": "${{ env.TF_WORK_DIR }}", "short": true },
                  { "title": "Triggered by", "value": "${{ github.actor }}", "short": true },
                  { "title": "Commit", "value": "${{ github.sha }}", "short": true }
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on Failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Terraform apply FAILED!",
              "attachments": [{
                "color": "danger",
                "fields": [
                  { "title": "Environment", "value": "${{ env.TF_WORK_DIR }}", "short": true },
                  { "title": "Triggered by", "value": "${{ github.actor }}", "short": true },
                  { "title": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false }
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### Multi-Environment Pipeline (Matrix Strategy)

```yaml
# .github/workflows/terraform-multi-env.yml
name: Terraform Multi-Environment

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ── Detect which environments changed ───────────────────────────
  detect-changes:
    name: Detect Changed Environments
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.filter.outputs.environments }}

    steps:
      - uses: actions/checkout@v4

      - name: Detect changes
        id: filter
        run: |
          # Get changed files
          CHANGED=$(git diff --name-only origin/main HEAD)

          # Build list of affected environments
          ENVS=()

          echo "$CHANGED" | grep -q "environments/dev/"     && ENVS+=("dev")
          echo "$CHANGED" | grep -q "environments/staging/" && ENVS+=("staging")
          echo "$CHANGED" | grep -q "environments/prod/"    && ENVS+=("prod")
          echo "$CHANGED" | grep -q "modules/"              && ENVS=("dev" "staging" "prod")

          # Convert to JSON array for matrix
          JSON=$(printf '%s\n' "${ENVS[@]}" | jq -R . | jq -cs .)
          echo "environments=$JSON" >> $GITHUB_OUTPUT


  # ── Plan for each changed environment ───────────────────────────
  plan:
    name: Plan - ${{ matrix.environment }}
    needs: detect-changes
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    strategy:
      fail-fast: false    # Continue other envs even if one fails
      matrix:
        environment: ${{ fromJson(needs.detect-changes.outputs.environments) }}

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"
          terraform_wrapper: false

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_ARN_{0}', upper(matrix.environment))] }}
          aws-region: us-east-1

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Terraform Plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan \
            -input=false \
            -no-color \
            -out=tfplan \
            -var-file="terraform.tfvars" \
            2>&1 | tee plan_output.txt

      - name: Post Plan Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync(
              'environments/${{ matrix.environment }}/plan_output.txt', 'utf8'
            );
            const truncated = plan.length > 30000 ?
              plan.substring(0, 30000) + '\n...(truncated)' : plan;

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `### 📋 Plan: \`${{ matrix.environment }}\`\n\`\`\`\n${truncated}\n\`\`\``
            });


  # ── Apply: Dev auto, staging/prod with gates ────────────────────
  apply-dev:
    name: Apply - dev
    needs: detect-changes
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && contains(fromJson(needs.detect-changes.outputs.environments), 'dev')
    environment: dev    # No approval required

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.6.0" }

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_DEV }}
          aws-region: us-east-1

      - run: terraform init -input=false
        working-directory: environments/dev

      - run: terraform apply -auto-approve -var-file="terraform.tfvars"
        working-directory: environments/dev


  apply-staging:
    name: Apply - staging
    needs: [detect-changes, apply-dev]   # Must pass dev first
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && contains(fromJson(needs.detect-changes.outputs.environments), 'staging')
    environment: staging    # Requires approval in GitHub settings

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.6.0" }

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: us-east-1

      - run: terraform init -input=false
        working-directory: environments/staging

      - run: terraform apply -auto-approve -var-file="terraform.tfvars"
        working-directory: environments/staging


  apply-prod:
    name: Apply - prod
    needs: [detect-changes, apply-staging]   # Must pass staging first
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && contains(fromJson(needs.detect-changes.outputs.environments), 'prod')
    environment: production    # Requires MULTIPLE approvers in GitHub settings

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "1.6.0" }

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: us-east-1

      - run: terraform init -input=false
        working-directory: environments/prod

      - run: terraform apply -auto-approve -var-file="terraform.tfvars"
        working-directory: environments/prod
```

---

### Interview-Focused Points 🎯

1. **Why use OIDC instead of static AWS credentials in CI/CD?** OIDC issues temporary credentials (1hr) with no static secrets to manage, rotate, or leak. AWS verifies the JWT from GitHub/GitLab directly — nothing to store.

2. **What is `terraform_wrapper: false` in setup-terraform action?** Disables the wrapper script that wraps Terraform commands. Required when you need to capture the actual Terraform exit code (e.g., `2` = changes in plan).

3. **What does `detailed-exitcode` do in `terraform plan`?** Returns exit code 0=no changes, 1=error, 2=changes detected. Essential for CI to differentiate between "plan succeeded with changes" vs "plan succeeded with no changes."

4. **Why save the plan with `-out=tfplan` and apply it separately?** Guarantees what was reviewed is exactly what gets applied. Without this, a second `terraform apply` might plan fresh changes that weren't reviewed.

5. **What is a GitHub Environment?** A named deployment target with configurable protection rules — required reviewers, wait timer, branch restrictions. Used to gate the `apply` step behind human approval.

---

## 🔴 ADVANCED LEVEL

### Jenkins Pipeline — Complete Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: terraform
    image: hashicorp/terraform:1.6.0
    command: ['cat']
    tty: true
    env:
    - name: AWS_DEFAULT_REGION
      value: us-east-1
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ['cat']
    tty: true
  serviceAccountName: jenkins-terraform   # K8s service account with IRSA
"""
        }
    }

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target environment'
        )
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Terraform action'
        )
        booleanParam(
            name: 'REFRESH_ONLY',
            defaultValue: false,
            description: 'Only refresh state (drift detection)'
        )
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        // Prevent concurrent builds for same environment
        lock(resource: "terraform-${params.ENVIRONMENT}")
    }

    environment {
        TF_WORK_DIR      = "environments/${params.ENVIRONMENT}"
        TF_IN_AUTOMATION = "true"    // Disables interactive prompts
        TF_CLI_ARGS      = "-no-color"
    }

    stages {
        // ── Stage 1: Checkout ────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        // ── Stage 2: Validate ────────────────────────────────────
        stage('Validate') {
            steps {
                container('terraform') {
                    dir(env.TF_WORK_DIR) {
                        sh '''
                            echo "=== Terraform Version ==="
                            terraform version

                            echo "=== Format Check ==="
                            terraform fmt -check -recursive ../../

                            echo "=== Init ==="
                            terraform init -input=false

                            echo "=== Validate ==="
                            terraform validate
                        '''
                    }
                }
            }
        }

        // ── Stage 3: Security Scan ───────────────────────────────
        stage('Security Scan') {
            steps {
                container('terraform') {
                    sh '''
                        # Install and run tfsec
                        wget -q https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 \
                             -O /usr/local/bin/tfsec
                        chmod +x /usr/local/bin/tfsec
                        tfsec . --format junit > tfsec-results.xml || true
                    '''
                    // Publish security scan results
                    junit allowEmptyResults: true, testResults: 'tfsec-results.xml'
                }
            }
        }

        // ── Stage 4: Plan ────────────────────────────────────────
        stage('Plan') {
            steps {
                container('terraform') {
                    dir(env.TF_WORK_DIR) {
                        script {
                            def planArgs = "-input=false -detailed-exitcode -out=tfplan"

                            if (params.REFRESH_ONLY) {
                                planArgs += " -refresh-only"
                            }

                            def exitCode = sh(
                                script: "terraform plan ${planArgs} -var-file=terraform.tfvars 2>&1 | tee plan_output.txt",
                                returnStatus: true
                            )

                            // 0=no changes, 2=changes, 1=error
                            if (exitCode == 1) {
                                error "Terraform plan failed!"
                            } else if (exitCode == 0) {
                                echo "✅ No changes detected"
                                currentBuild.description = "No changes in ${params.ENVIRONMENT}"
                            } else if (exitCode == 2) {
                                echo "⚠️ Changes detected — review required"
                                currentBuild.description = "Changes pending in ${params.ENVIRONMENT}"
                            }

                            // Show human-readable summary
                            sh 'grep -E "Plan:|No changes" plan_output.txt || true'
                        }
                    }
                }
            }
            post {
                always {
                    // Archive plan output
                    archiveArtifacts artifacts: "${env.TF_WORK_DIR}/plan_output.txt",
                                     allowEmptyArchive: true
                }
            }
        }

        // ── Stage 5: Approval Gate ───────────────────────────────
        stage('Approval') {
            when {
                allOf {
                    expression { params.ACTION == 'apply' || params.ACTION == 'destroy' }
                    expression { params.ENVIRONMENT == 'prod' || params.ENVIRONMENT == 'staging' }
                }
            }
            steps {
                script {
                    def planSummary = sh(
                        script: "cat ${env.TF_WORK_DIR}/plan_output.txt | tail -20",
                        returnStdout: true
                    ).trim()

                    // Send notification
                    slackSend(
                        channel: '#infra-approvals',
                        color: 'warning',
                        message: """
🔔 *Terraform ${params.ACTION.toUpperCase()} requires approval*
*Environment:* ${params.ENVIRONMENT}
*Triggered by:* ${env.BUILD_USER ?: 'Automated'}
*Commit:* ${env.GIT_COMMIT_SHORT}
*Plan Summary:*
```${planSummary}```
*Approve here:* ${env.BUILD_URL}input
                        """
                    )

                    // Pause for human input
                    timeout(time: 2, unit: 'HOURS') {
                        input(
                            message: "Review the plan above. Approve ${params.ACTION} to ${params.ENVIRONMENT}?",
                            ok: "Approve ${params.ACTION.toUpperCase()}",
                            submitterParameter: 'APPROVER'
                        )
                    }

                    echo "Approved by: ${env.APPROVER}"
                }
            }
        }

        // ── Stage 6: Apply or Destroy ────────────────────────────
        stage('Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                container('terraform') {
                    dir(env.TF_WORK_DIR) {
                        sh 'terraform apply -auto-approve tfplan'
                    }
                }
            }
        }

        stage('Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                container('terraform') {
                    dir(env.TF_WORK_DIR) {
                        sh '''
                            terraform plan -destroy \
                              -input=false \
                              -out=destroy_plan \
                              -var-file=terraform.tfvars
                            terraform apply -auto-approve destroy_plan
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#infra-deployments',
                color: 'good',
                message: "✅ Terraform ${params.ACTION} succeeded for *${params.ENVIRONMENT}* | ${env.GIT_COMMIT_SHORT}"
            )
        }
        failure {
            slackSend(
                channel: '#infra-deployments',
                color: 'danger',
                message: "❌ Terraform ${params.ACTION} FAILED for *${params.ENVIRONMENT}* | ${env.BUILD_URL}"
            )
        }
        always {
            // Clean workspace between runs
            cleanWs()
        }
    }
}
```

---

### GitLab CI Pipeline — Complete Configuration

```yaml
# .gitlab-ci.yml
image:
  name: hashicorp/terraform:1.6.0
  entrypoint: [""]

variables:
  TF_ROOT:         ${CI_PROJECT_DIR}/environments/prod
  TF_ADDRESS:      "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/prod"
  TF_IN_AUTOMATION: "true"

# ── Cache terraform plugins ───────────────────────────────────────
cache:
  key: terraform
  paths:
    - ${TF_ROOT}/.terraform

# ── Stages ────────────────────────────────────────────────────────
stages:
  - validate
  - security
  - plan
  - apply
  - notify

# ── Before script: Configure AWS via OIDC ─────────────────────────
.aws_auth: &aws_auth
  before_script:
    - apk add --no-cache curl jq python3 py3-pip
    - pip3 install awscli --quiet
    # GitLab OIDC token → AWS credentials
    - |
      CREDENTIALS=$(aws sts assume-role-with-web-identity \
        --role-arn "${AWS_ROLE_ARN}" \
        --role-session-name "GitLab-${CI_JOB_NAME}-${CI_PIPELINE_ID}" \
        --web-identity-token "${CI_JOB_JWT_V2}" \
        --duration-seconds 3600 \
        --query 'Credentials' \
        --output json)
      export AWS_ACCESS_KEY_ID=$(echo $CREDENTIALS | jq -r '.AccessKeyId')
      export AWS_SECRET_ACCESS_KEY=$(echo $CREDENTIALS | jq -r '.SecretAccessKey')
      export AWS_SESSION_TOKEN=$(echo $CREDENTIALS | jq -r '.SessionToken')
    - cd ${TF_ROOT}
    - terraform init -input=false


# ════════════════════════════════════════════════════════════════
# STAGE 1: VALIDATE
# ════════════════════════════════════════════════════════════════
fmt:
  stage: validate
  script:
    - terraform fmt -check -recursive ${CI_PROJECT_DIR}
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

validate:
  stage: validate
  <<: *aws_auth
  script:
    - terraform validate
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH


# ════════════════════════════════════════════════════════════════
# STAGE 2: SECURITY
# ════════════════════════════════════════════════════════════════
tfsec:
  stage: security
  image: aquasec/tfsec-ci:latest
  script:
    - tfsec ${CI_PROJECT_DIR} --format junit --out tfsec-results.xml
  artifacts:
    reports:
      junit: tfsec-results.xml
    when: always
  allow_failure: true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

checkov:
  stage: security
  image: bridgecrew/checkov:latest
  script:
    - checkov -d ${TF_ROOT} --framework terraform --output junitxml > checkov-results.xml
  artifacts:
    reports:
      junit: checkov-results.xml
    when: always
  allow_failure: true
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"


# ════════════════════════════════════════════════════════════════
# STAGE 3: PLAN
# ════════════════════════════════════════════════════════════════
plan:
  stage: plan
  <<: *aws_auth
  script:
    - |
      terraform plan \
        -input=false \
        -no-color \
        -out=tfplan \
        -var-file=terraform.tfvars \
        2>&1 | tee plan_output.txt
    # Generate JSON plan for MR widget
    - terraform show -json tfplan > plan.json
  artifacts:
    name: tfplan-${CI_COMMIT_SHORT_SHA}
    paths:
      - ${TF_ROOT}/tfplan
      - ${TF_ROOT}/plan_output.txt
      - ${TF_ROOT}/plan.json
    expire_in: 1 week
    reports:
      terraform: ${TF_ROOT}/plan.json   # GitLab native Terraform MR widget!
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH


# ════════════════════════════════════════════════════════════════
# STAGE 4: APPLY
# ════════════════════════════════════════════════════════════════
apply:
  stage: apply
  <<: *aws_auth
  script:
    - terraform apply -auto-approve tfplan
    - terraform output -json > outputs.json
  artifacts:
    paths:
      - ${TF_ROOT}/outputs.json
    expire_in: 1 week
  environment:
    name: production
    url: $APP_URL
  # ← GitLab environment with deployment tracking
  when: manual              # Requires manual trigger
  allow_failure: false
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  needs:
    - job: plan
      artifacts: true


# ════════════════════════════════════════════════════════════════
# STAGE 5: NOTIFY
# ════════════════════════════════════════════════════════════════
notify-success:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -s -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-type: application/json' \
        --data '{
          "text": "✅ Terraform apply succeeded!",
          "attachments": [{
            "color": "good",
            "fields": [
              {"title": "Environment", "value": "production", "short": true},
              {"title": "Triggered by", "value": "'"${GITLAB_USER_LOGIN}"'", "short": true},
              {"title": "Pipeline", "value": "'"${CI_PIPELINE_URL}"'", "short": false}
            ]
          }]
        }'
  when: on_success
  needs: [apply]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

notify-failure:
  stage: notify
  image: curlimages/curl:latest
  script:
    - |
      curl -s -X POST "${SLACK_WEBHOOK_URL}" \
        -H 'Content-type: application/json' \
        --data '{
          "text": "❌ Terraform apply FAILED!",
          "attachments": [{
            "color": "danger",
            "fields": [
              {"title": "Pipeline", "value": "'"${CI_PIPELINE_URL}"'", "short": false}
            ]
          }]
        }'
  when: on_failure
  needs: [apply]
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

---

### Security Scanning Integration

```yaml
# Complete security scanning stage for any CI platform

# ── tfsec: AWS-focused security scanner ──────────────────────────
- name: Run tfsec
  uses: aquasecurity/tfsec-action@v1
  with:
    working_directory: environments/prod
    format: lovely
    additional_args: >
      --severity HIGH
      --exclude-path .terraform
      --custom-check-dir .tfsec-checks

# ── Checkov: Policy-as-code scanner ──────────────────────────────
- name: Run Checkov
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: environments/prod
    framework: terraform
    output_format: junitxml
    output_file_path: checkov-results.xml
    skip_check: >
      CKV_AWS_18,   # Skip: S3 access logging (we use CloudTrail)
      CKV_AWS_21    # Skip: S3 versioning (handled separately)

# ── Terrascan: Multi-cloud policy scanner ────────────────────────
- name: Run Terrascan
  uses: tenable/terrascan-action@main
  with:
    iac_type: terraform
    iac_version: v15
    policy_type: aws
    only_warn: true

# ── OPA/Conftest: Custom policy enforcement ───────────────────────
- name: Run Conftest
  run: |
    # Download conftest
    wget https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz
    tar xzf conftest_0.46.0_Linux_x86_64.tar.gz

    # Generate plan JSON
    terraform show -json tfplan > plan.json

    # Run custom policies
    ./conftest test plan.json --policy ./policies/

# ── Custom OPA policies ───────────────────────────────────────────
# policies/terraform.rego

package main

# Deny if any S3 bucket is public
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.acl == "public-read"
  msg := sprintf(
    "S3 bucket '%s' must not be public",
    [resource.name]
  )
}

# Deny if EC2 instance type not in approved list
approved_instance_types := {
  "t3.micro", "t3.small", "t3.medium",
  "t3.large", "m5.large", "m5.xlarge"
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  instance_type := resource.change.after.instance_type
  not approved_instance_types[instance_type]
  msg := sprintf(
    "Instance type '%s' is not in the approved list",
    [instance_type]
  )
}

# Warn if environment tag missing
warn[msg] {
  resource := input.resource_changes[_]
  resource.change.after != null
  not resource.change.after.tags.Environment
  msg := sprintf(
    "Resource '%s' is missing required 'Environment' tag",
    [resource.address]
  )
}
```

---

### Advanced Pipeline Patterns

**Pattern 1: Drift Detection as Scheduled Job**

```yaml
# .github/workflows/drift-detection.yml
name: Terraform Drift Detection

on:
  schedule:
    - cron: '0 8 * * 1-5'    # 8am UTC, Monday-Friday
  workflow_dispatch:           # Allow manual trigger

jobs:
  drift-check:
    name: Check Drift - ${{ matrix.environment }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.0"
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_ARN_{0}', upper(matrix.environment))] }}
          aws-region: us-east-1

      - run: terraform init -input=false
        working-directory: environments/${{ matrix.environment }}

      - name: Check for Drift
        id: drift
        working-directory: environments/${{ matrix.environment }}
        run: |
          terraform plan \
            -refresh-only \
            -detailed-exitcode \
            -no-color \
            -input=false \
            -var-file=terraform.tfvars \
            2>&1 | tee drift_output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
        continue-on-error: true

      - name: Alert on Drift
        if: steps.drift.outputs.exit_code == '2'
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⚠️ Infrastructure drift detected in *${{ matrix.environment }}*!",
              "attachments": [{
                "color": "warning",
                "text": "Run `terraform apply -refresh-only` to accept drift or `terraform apply` to revert it.",
                "fields": [
                  {"title": "Environment", "value": "${{ matrix.environment }}", "short": true},
                  {"title": "Detected at", "value": "${{ github.event.head_commit.timestamp }}", "short": true},
                  {"title": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

**Pattern 2: PR-based Terraform Workflow with Atlantis**

```yaml
# atlantis.yaml — Atlantis config for PR-based workflow
version: 3
automerge: false
delete_source_branch_on_merge: false

projects:
  - name: dev
    dir: environments/dev
    workspace: default
    terraform_version: v1.6.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved                    # Requires PR approval
      - mergeable                   # PR must be mergeable

  - name: staging
    dir: environments/staging
    workspace: default
    terraform_version: v1.6.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable

  - name: prod
    dir: environments/prod
    workspace: default
    terraform_version: v1.6.0
    autoplan:
      when_modified: ["*.tf", "*.tfvars", "../../modules/**/*.tf"]
      enabled: true
    apply_requirements:
      - approved
      - mergeable
      - undiverged    # Branch must be up to date with main
```

```bash
# With Atlantis, engineers use PR comments to trigger Terraform:

# In a PR comment:
atlantis plan -d environments/prod        # Runs plan
atlantis apply -d environments/prod       # Runs apply (if approved)
atlantis plan -d environments/prod -var-file=prod.tfvars

# Atlantis bot responds with plan output directly in PR
# Apply is locked until PR approved by required reviewers
```

---

**Pattern 3: Terraform Cloud Integration**

```hcl
# versions.tf — Use Terraform Cloud as backend
terraform {
  cloud {
    organization = "mycompany"

    workspaces {
      name = "web-app-prod"
      # OR use tags for dynamic workspace selection:
      # tags = ["app:web", "env:prod"]
    }
  }
}
```

```yaml
# GitHub Actions with Terraform Cloud (simplest approach)
- name: Setup Terraform with Terraform Cloud
  uses: hashicorp/setup-terraform@v3
  with:
    cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}
    # No AWS creds needed here — TFC manages them via variable sets

- name: Terraform Init
  run: terraform init    # Connects to Terraform Cloud automatically

- name: Terraform Plan
  run: terraform plan    # Runs remotely in TFC, outputs stream back

- name: Terraform Apply
  if: github.ref == 'refs/heads/main'
  run: terraform apply -auto-approve
  # In TFC, apply can require approval via TFC UI policy
```

---

### Production Best Practices

```yaml
# ✅ 1. ALWAYS use concurrency groups to prevent parallel applies
concurrency:
  group: terraform-prod
  cancel-in-progress: false   # Never cancel — queue instead


# ✅ 2. Always set TF_IN_AUTOMATION=true in CI
env:
  TF_IN_AUTOMATION: "true"    # Disables interactive prompts + colored output


# ✅ 3. Always pin ALL action versions
uses: actions/checkout@v4                    # ✅ Pinned
uses: hashicorp/setup-terraform@v3           # ✅ Pinned
uses: actions/checkout@main                  # ❌ Mutable — can change!


# ✅ 4. Restrict OIDC role to specific repos and branches
Condition:
  StringLike:
    "token.actions.githubusercontent.com:sub":
      - "repo:myorg/infra:ref:refs/heads/main"   # Only main branch
      # NOT: "repo:myorg/infra:*"                # Too broad!


# ✅ 5. Save and apply the same plan file
- run: terraform plan -out=tfplan
- run: terraform apply tfplan    # Not: terraform apply -auto-approve


# ✅ 6. Upload plan as artifact for audit trail
- uses: actions/upload-artifact@v4
  with:
    name: tfplan-${{ github.sha }}
    path: tfplan
    retention-days: 90    # Keep for compliance audit


# ✅ 7. Always run security scanning before apply
# tfsec, checkov, terrascan → fail PR if HIGH severity issues


# ✅ 8. Post plan to PR — never apply without visible review
# Use github-script to post formatted plan as PR comment


# ✅ 9. Use separate IAM roles per environment
# Dev role: limited permissions, dev account only
# Prod role: specific permissions, prod account, MFA required, restricted to main branch


# ✅ 10. Use -detailed-exitcode for intelligent pipeline flow
# Exit 0 = no changes (skip apply notification)
# Exit 2 = changes (require review)
# Exit 1 = error (fail loudly)
```

---

### Debugging CI/CD Pipeline Issues

```bash
# ── Debug GitHub Actions ──────────────────────────────────────────
# Enable debug logging
# Set secret: ACTIONS_STEP_DEBUG = true
# Set secret: ACTIONS_RUNNER_DEBUG = true

# ── Debug Terraform in CI ─────────────────────────────────────────
- name: Debug Terraform
  env:
    TF_LOG: DEBUG
    TF_LOG_PATH: /tmp/terraform-debug.log
  run: terraform plan

- name: Upload Debug Log
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: terraform-debug-log
    path: /tmp/terraform-debug.log


# ── Common CI/CD Errors and Fixes ────────────────────────────────

# Error: "Error acquiring the state lock"
# Fix: Another pipeline is running. Check concurrency group.
# Or: terraform force-unlock LOCK_ID (if stale)

# Error: "No valid credential sources found"
# Fix: OIDC not configured. Check role ARN, check OIDC conditions.
# Verify: aws sts get-caller-identity

# Error: "exit code 2" on plan (not treated as success)
# Fix: Add continue-on-error: true + check exit code manually

# Error: "Provider produced inconsistent result after apply"
# Fix: Usually eventual consistency in AWS APIs. Add retry logic or depends_on.

# Error: "Error: Invalid count argument — value depends on resource attribute"
# Fix: count/for_each must be known at plan time. Use static values.
```

---

## 📁 CODE WRITING GUIDANCE

### Complete CI/CD Repository Structure

```
infrastructure/
│
├── .github/
│   └── workflows/
│       ├── terraform-validate.yml    ← Runs on all PRs
│       ├── terraform-plan.yml        ← Plans per environment on PR
│       ├── terraform-apply.yml       ← Applies on merge to main
│       ├── terraform-drift.yml       ← Scheduled drift detection
│       └── terraform-destroy.yml     ← Manual destroy workflow
│
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
│
├── modules/
│
├── policies/                         ← OPA/Conftest policies
│   ├── terraform.rego
│   ├── tagging.rego
│   └── approved-resources.rego
│
├── .tfsec-checks/                    ← Custom tfsec rules
│   └── custom-checks.json
│
├── .tflint.hcl                       ← tflint configuration
│
└── scripts/
    ├── apply-env.sh
    └── drift-check.sh
```

```hcl
# .tflint.hcl — Linting rules
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_required_version" { enabled = true }
rule "terraform_required_providers" { enabled = true }
rule "terraform_naming_convention" {
  enabled = true
  # Resources must use snake_case
  resource {
    format = "snake_case"
  }
}
rule "terraform_documented_variables" { enabled = true }
rule "terraform_documented_outputs" { enabled = true }

# AWS-specific rules
rule "aws_instance_invalid_type" { enabled = true }
rule "aws_instance_previous_type" { enabled = true }
rule "aws_s3_bucket_name" { enabled = true }
```

---

## 🧪 HANDS-ON LAB

### Exercise: Build a Complete Terraform CI/CD Pipeline

**Goal:** Create a GitHub Actions pipeline that:
1. On every PR: runs `fmt`, `validate`, `plan`, posts plan to PR comment
2. On merge to `main`: runs `apply` with production environment gate
3. Nightly (7am UTC): runs drift detection across all environments
4. On any failure: sends a Slack notification
5. Uses OIDC for AWS authentication (no static credentials)
6. Serializes applies with concurrency groups

**Try implementing it yourself, then check the solution!**

---

### ✅ Solution

```yaml
# .github/workflows/terraform-complete.yml
name: Terraform Complete Pipeline

on:
  push:
    branches: [main]
    paths: ['environments/**', 'modules/**']
  pull_request:
    branches: [main]
    paths: ['environments/**', 'modules/**']
  schedule:
    - cron: '0 7 * * *'    # Drift detection: 7am UTC daily
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [dev, staging, prod]
      action:
        description: 'Action'
        required: true
        type: choice
        options: [plan, apply, drift-check]

concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false

env:
  TF_VERSION:      "1.6.0"
  TF_IN_AUTOMATION: "true"
  AWS_REGION:      "us-east-1"

permissions:
  id-token:      write
  contents:      read
  pull-requests: write

jobs:
  # ══════════════════════════════════════════════════
  # VALIDATE — runs on every PR
  # ══════════════════════════════════════════════════
  validate:
    name: Validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_DEV }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Init + Validate (all environments)
        run: |
          for ENV in dev staging prod; do
            echo "=== Validating $ENV ==="
            cd environments/$ENV
            terraform init -input=false -backend=false
            terraform validate
            cd -
          done

      - name: Report fmt status on PR
        if: steps.fmt.outcome == 'failure'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: '❌ **Terraform fmt check failed!** Run `terraform fmt -recursive` and push again.'
            });


  # ══════════════════════════════════════════════════
  # PLAN — runs on PRs for each environment
  # ══════════════════════════════════════════════════
  plan:
    name: Plan - ${{ matrix.environment }}
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'

    strategy:
      fail-fast: false
      matrix:
        environment: [dev, staging, prod]

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_ARN_{0}', upper(matrix.environment))] }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        working-directory: environments/${{ matrix.environment }}
        run: terraform init -input=false

      - name: Terraform Plan
        id: plan
        working-directory: environments/${{ matrix.environment }}
        run: |
          set +e
          terraform plan \
            -input=false \
            -no-color \
            -detailed-exitcode \
            -out=tfplan \
            -var-file=terraform.tfvars \
            2>&1 | tee plan_output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
          set -e

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ matrix.environment }}-${{ github.sha }}
          path: environments/${{ matrix.environment }}/tfplan
          retention-days: 7

      - name: Post Plan Comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const exitCode = '${{ steps.plan.outputs.exit_code }}';
            let plan = fs.readFileSync(
              'environments/${{ matrix.environment }}/plan_output.txt', 'utf8'
            );
            if (plan.length > 30000) plan = plan.slice(0,30000) + '\n...(truncated)';

            const icons = { '0':'✅', '2':'⚠️', '1':'❌' };
            const labels = { '0':'No Changes', '2':'Changes Detected', '1':'Error' };

            const body = [
              `### ${icons[exitCode]||'❓'} Plan: \`${{ matrix.environment }}\` — ${labels[exitCode]||'Unknown'}`,
              `<details><summary>Show Plan Output</summary>`,
              ``,`\`\`\`hcl`, plan, `\`\`\``,``,`</details>`,
              `*Commit: \`${{ github.sha }}\`*`
            ].join('\n');

            // Remove old plan comments for this env
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number
            });
            for (const c of comments) {
              if (c.body.includes(`Plan: \`${{ matrix.environment }}\``)) {
                await github.rest.issues.deleteComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  comment_id: c.id
                });
              }
            }
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body
            });

      - name: Fail if plan errored
        if: steps.plan.outputs.exit_code == '1'
        run: exit 1


  # ══════════════════════════════════════════════════
  # APPLY — runs on merge to main (dev auto, prod gated)
  # ══════════════════════════════════════════════════
  apply-dev:
    name: Apply - dev
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: dev

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "${{ env.TF_VERSION }}" }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_DEV }}
          aws-region: ${{ env.AWS_REGION }}
      - run: terraform init -input=false
        working-directory: environments/dev
      - run: terraform apply -auto-approve -var-file=terraform.tfvars -input=false
        working-directory: environments/dev

  apply-staging:
    name: Apply - staging
    runs-on: ubuntu-latest
    needs: apply-dev
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: staging    # Requires approval

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "${{ env.TF_VERSION }}" }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_STAGING }}
          aws-region: ${{ env.AWS_REGION }}
      - run: terraform init -input=false
        working-directory: environments/staging
      - run: terraform apply -auto-approve -var-file=terraform.tfvars -input=false
        working-directory: environments/staging

  apply-prod:
    name: Apply - prod
    runs-on: ubuntu-latest
    needs: apply-staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production    # Requires multiple approvers

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with: { terraform_version: "${{ env.TF_VERSION }}" }
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_PROD }}
          aws-region: ${{ env.AWS_REGION }}
      - run: terraform init -input=false
        working-directory: environments/prod
      - run: |
          terraform plan -input=false -out=tfplan -var-file=terraform.tfvars
          terraform apply -auto-approve tfplan
        working-directory: environments/prod


  # ══════════════════════════════════════════════════
  # DRIFT DETECTION — scheduled nightly
  # ══════════════════════════════════════════════════
  drift-detection:
    name: Drift Detection - ${{ matrix.environment }}
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    strategy:
      fail-fast: false
      matrix:
        environment: [dev, staging, prod]

    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
          terraform_wrapper: false

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets[format('AWS_ROLE_ARN_{0}', upper(matrix.environment))] }}
          aws-region: ${{ env.AWS_REGION }}

      - run: terraform init -input=false
        working-directory: environments/${{ matrix.environment }}

      - name: Detect Drift
        id: drift
        working-directory: environments/${{ matrix.environment }}
        run: |
          set +e
          terraform plan -refresh-only -detailed-exitcode \
            -input=false -no-color -var-file=terraform.tfvars \
            2>&1 | tee drift_output.txt
          echo "exit_code=$?" >> $GITHUB_OUTPUT
          set -e

      - name: Alert on Drift
        if: steps.drift.outputs.exit_code == '2'
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⚠️ Drift detected in *${{ matrix.environment }}*!",
              "attachments": [{
                "color": "warning",
                "fields": [
                  {"title": "Environment", "value": "${{ matrix.environment }}", "short": true},
                  {"title": "Run URL", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 📋 SUMMARY CHEAT SHEET

### Key Points

| Concept | Key Takeaway |
|---|---|
| CI/CD for Terraform | All changes via pipeline — no manual `apply` from laptops |
| OIDC auth | No static credentials — CI/CD gets temp tokens via JWT |
| `terraform_wrapper: false` | Required to capture Terraform's actual exit code in GH Actions |
| `detailed-exitcode` | 0=no changes, 1=error, 2=changes — essential for CI decisions |
| `-out=tfplan` | Save plan → apply exact same plan — guarantees reviewed changes get applied |
| Concurrency groups | Prevent parallel applies to same environment — serializes runs |
| GitHub Environments | Gate apply jobs behind required reviewers and branch restrictions |
| `TF_IN_AUTOMATION` | Disables interactive prompts — must be set in all CI pipelines |
| Drift detection | Scheduled `plan -refresh-only` — catches manual changes in cloud |
| Security scanning | tfsec + checkov + OPA/conftest — runs before apply, blocks HIGH severity |

---

### Quick Reference

```yaml
# OIDC Auth
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1

# Setup Terraform (no wrapper for exit codes)
- uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: "1.6.0"
    terraform_wrapper: false

# Plan with exit code capture
- id: plan
  run: |
    terraform plan -detailed-exitcode -out=tfplan 2>&1 | tee plan.txt
    echo "exit_code=$?" >> $GITHUB_OUTPUT
  continue-on-error: true

# Apply saved plan
- run: terraform apply -auto-approve tfplan

# Concurrency (prevents parallel applies)
concurrency:
  group: terraform-${{ github.ref }}
  cancel-in-progress: false

# Gate with environment
environment: production   # Requires reviewer approval in GitHub settings

# Key env vars
env:
  TF_IN_AUTOMATION: "true"
  TF_VERSION: "1.6.0"

# Drift detection
- run: terraform plan -refresh-only -detailed-exitcode
  continue-on-error: true
# Exit 0 = clean, 2 = drift, 1 = error
```

---

### Top 10 Interview Questions

1. **Why use OIDC instead of AWS access keys in CI/CD?** OIDC issues short-lived temporary credentials (1hr) via JWT federation with no static secrets to manage, rotate, or risk leaking. AWS directly validates the token from GitHub/GitLab.

2. **What is `TF_IN_AUTOMATION` and why set it?** An environment variable that disables Terraform's interactive prompts and adjusts output formatting for machine consumption — essential to prevent pipelines from hanging waiting for user input.

3. **Why save the plan with `-out` and apply it separately?** Guarantees the reviewed plan is exactly what gets applied. A fresh `terraform apply` re-plans and might include new changes that weren't reviewed.

4. **What does exit code `2` mean in `terraform plan -detailed-exitcode`?** Plan succeeded and there ARE changes to apply. Exit 0 = success + no changes. Exit 1 = error. This enables CI pipelines to differentiate between "clean" and "has changes."

5. **How do you prevent two pipeline runs from corrupting Terraform state?** Two layers: GitHub Actions concurrency groups serialize pipeline runs, and DynamoDB state locking prevents simultaneous state writes at the Terraform level.

6. **What is a GitHub Environment in the context of Terraform CI/CD?** A named deployment target with protection rules — required reviewers, wait timers, branch restrictions. Used to gate the `apply` step behind human approval for staging/production.

7. **How do you show the Terraform plan in a PR without running apply?** Use `terraform plan -out=tfplan`, capture output to a file, then post it as a PR comment via the GitHub API using `actions/github-script`.

8. **What tools do you use for Terraform security scanning in CI?** tfsec (AWS-focused rules), Checkov (multi-cloud CIS benchmarks), Terrascan, and OPA/Conftest for custom organizational policies.

9. **How do you handle secrets in a Terraform CI/CD pipeline?** OIDC for cloud credentials (no secrets), GitHub/GitLab encrypted secrets for API tokens, AWS Secrets Manager / SSM Parameter Store for application secrets referenced via data sources.

10. **What is drift detection and how do you automate it?** Detecting when real cloud infrastructure differs from Terraform state (due to manual changes). Automated via scheduled pipeline running `terraform plan -refresh-only -detailed-exitcode` and alerting on exit code 2.

---
