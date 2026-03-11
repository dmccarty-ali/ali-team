---
name: ali-terraform-admin
description: |
  Terraform operations with safety, state, and environment protection. Use when:

  PLANNING: Planning infrastructure changes, evaluating terraform modules,
  designing state management, workspace strategy, backend configuration

  IMPLEMENTATION: Executing terraform commands, applying infrastructure changes,
  managing state, importing resources, managing workspaces

  GUIDANCE: Asking about Terraform best practices, state management,
  module design, workspace vs directory structure, remote backends

  REVIEW: Reviewing terraform plans, auditing state, checking for drift,
  validating infrastructure against desired state
---

# Terraform Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Planning infrastructure changes with Terraform
- Evaluating terraform module design
- Designing state management strategy
- Planning workspace vs directory structure
- Designing backend configuration (S3, DynamoDB)
- Planning multi-environment infrastructure

**Implementation:**
- Executing terraform commands (plan, apply, destroy)
- Managing terraform state (list, show, mv, rm, import)
- Managing workspaces (create, switch, delete)
- Configuring remote backends
- Importing existing resources
- Refactoring terraform code

**Guidance/Best Practices:**
- State management best practices
- Workspace vs directory structure trade-offs
- Remote backend configuration
- Module design patterns
- Environment isolation strategies
- Drift detection and remediation

**Review/Validation:**
- Reviewing terraform plans before apply
- Auditing state for unexpected resources
- Checking for infrastructure drift
- Validating workspace configurations
- Checking backend security settings

---

## Core Principles

### 1. Always Plan Before Apply (Non-Negotiable)

| Rule | Rationale |
|------|-----------|
| **Always run plan first** | Review changes before they happen - no surprises |
| **Never -auto-approve production** | Human review required for production changes |
| **Save plan to file** | `terraform plan -out=plan.tfplan` for reproducibility |
| **Review plan output** | Check create/modify/destroy counts before apply |

### 2. Protect State Files

| Practice | Rationale |
|----------|-----------|
| **Never edit .tfstate manually** | State corruption - use terraform state commands |
| **Never commit state to git** | State files contain secrets, use remote backend |
| **Use remote backend** | S3 + DynamoDB for team collaboration and locking |
| **Enable state versioning** | S3 versioning allows rollback if corrupted |
| **Encrypt state at rest** | State contains sensitive data (IPs, secrets) |

### 3. Use State Locking

| Practice | Impact |
|----------|--------|
| **Configure DynamoDB locking** | Prevents concurrent operations from corrupting state |
| **Wait for lock release** | Don't force-unlock unless truly stale |
| **One operation at a time** | No parallel applies on same state |

### 4. Environment Isolation

| Strategy | Use Case |
|----------|----------|
| **Workspaces** | Same code, different state (dev/staging/prod) |
| **Directories** | Different code per environment (more isolation) |
| **Separate backends** | Different S3 buckets per environment (max isolation) |

---

## Dangerous Operations (REQUIRE CONFIRMATION)

### Critical Risk (Infrastructure Destruction)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `terraform destroy` | Destroys ALL managed resources | "Yes, destroy all resources in [workspace]" |
| `terraform apply` (with deletions) | Resource deletion | "Yes, apply plan with [N] deletions" |
| `terraform state rm` | Removes from state (creates orphan) | "Yes, remove [resource] from state" |
| `terraform state mv` | Moves state entries (risky if wrong) | "Yes, move state entry [source] to [dest]" |
| `terraform import` | Adds resource to state | "Yes, import [resource] to [address]" |
| `terraform force-unlock` | Breaks state lock | "Yes, force unlock state [lock-id]" |
| `terraform workspace delete` | Deletes environment workspace | "Yes, delete workspace [name]" |
| `terraform taint` | Marks resource for recreation | "Yes, taint [resource] for recreation" |
| `terraform apply -target` | Partial apply (risky) | "Yes, apply targeted resource [target]" |

### ALWAYS BLOCKED (Security Violations)

| Operation | Why Blocked |
|-----------|-------------|
| `terraform destroy` on production workspace | Production infrastructure must never be destroyed via CLI |
| `terraform apply -auto-approve` on production | Production changes must be reviewed via plan |
| Direct modification of terraform.tfstate | State corruption - use terraform state commands only |
| Committing state files to git | State files contain secrets, use remote backend |

---

## Safe Operations (No Confirmation Needed)

```bash
# Read-only operations
terraform version
terraform show
terraform show <resource>
terraform output
terraform output <output-name>
terraform state list
terraform state show <resource>
terraform workspace list
terraform workspace show
terraform providers
terraform graph

# Validation operations
terraform validate
terraform fmt -check
terraform fmt -diff

# Planning operations (no changes)
terraform init
terraform plan
terraform plan -out=plan.tfplan
terraform console  # Read-only interactive
```

---

## Apply Workflow (MANDATORY)

NEVER skip plan review before apply:

### Step 1: Run Plan

```bash
terraform plan -out=plan.tfplan
```

### Step 2: Review Plan Output

Check the summary:
```
Plan: 3 to add, 1 to change, 0 to destroy.
```

- **+ Create:** New resources being added
- **~ Modify:** Existing resources being changed
- **- Destroy:** Resources being deleted (⚠️ DANGER)

Look for:
- Unexpected deletions (accidental remove from code?)
- Large-scale changes (typo in variable?)
- Security group changes (ports opening?)
- Database modifications (downtime risk?)

### Step 3: Get Confirmation

For production or destructive changes, get explicit user confirmation.

### Step 4: Apply with Plan File

```bash
terraform apply plan.tfplan
```

**Why use plan file:**
- Ensures exact changes reviewed in plan are applied
- Prevents drift between plan and apply
- No surprise changes due to state updates between plan/apply

---

## State Management

### Listing State

```bash
# List all resources in state
terraform state list

# Show specific resource details
terraform state show aws_instance.example

# Show all attributes in JSON
terraform state show -json aws_instance.example
```

### Moving State Entries (REQUIRES CONFIRMATION)

**Use case:** Refactoring module structure, renaming resources

```bash
# Move resource within state
terraform state mv aws_instance.old aws_instance.new

# Move resource into module
terraform state mv aws_instance.web module.web.aws_instance.main

# Move resource out of module
terraform state mv module.web.aws_instance.main aws_instance.web
```

**Workflow:**
1. Update .tf files with new resource names/paths
2. Run state mv to update state
3. Run plan to verify no changes (state and code match)

### Removing from State (REQUIRES CONFIRMATION)

**Use case:** Resource should no longer be managed by Terraform

```bash
# Remove resource from state (resource still exists in cloud)
terraform state rm aws_instance.legacy

# Resource becomes an orphan - Terraform no longer manages it
# To destroy: manually delete via AWS console or CLI
```

**When to use:**
- Migrating resource to different Terraform project
- Resource should exist but not be managed
- Decomissioning Terraform management

**Warning:** Resource still exists but Terraform won't track or modify it.

### Importing Resources (REQUIRES CONFIRMATION)

**Use case:** Bring existing cloud resources under Terraform management

```bash
# Step 1: Add resource definition to .tf files
resource "aws_instance" "imported_server" {
  # Configuration matching existing resource
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  # ... other attributes
}

# Step 2: Import existing resource
terraform import aws_instance.imported_server i-1234567890abcdef0

# Step 3: Run plan to check for drift
terraform plan
# Should show "No changes" if definition matches reality
```

**Common import targets:**
- `aws_instance` - EC2 instances
- `aws_s3_bucket` - S3 buckets
- `aws_vpc` - VPCs
- `aws_security_group` - Security groups
- `aws_iam_role` - IAM roles

### State Safety Rules

**NEVER:**
- Edit terraform.tfstate manually (use terraform state commands)
- Commit terraform.tfstate to git (use remote backend)
- Share state files via email/Slack (contains secrets)
- Copy state files between environments (corruption risk)
- Run concurrent terraform operations (state lock conflict)

**ALWAYS:**
- Use remote backend (S3 + DynamoDB for locking)
- Enable state versioning (S3 versioning for rollback)
- Enable state encryption (S3 server-side encryption)
- Use terraform state commands for state operations
- Wait for state lock to release before retrying

---

## Workspace Management

### Purpose

Workspaces allow multiple state files using the same Terraform configuration.

**Use case:** Same infrastructure code for dev/staging/prod with different state.

### Listing Workspaces

```bash
# List all workspaces (* marks current)
terraform workspace list

# Show current workspace
terraform workspace show
```

### Creating Workspaces

```bash
# Create new workspace
terraform workspace new staging

# Workspace-specific state created
# Backend key: workspaces/staging/terraform.tfstate
```

### Switching Workspaces

```bash
# Switch to different workspace
terraform workspace select dev

# CAUTION: Switches which infrastructure you're managing
# Always verify workspace before apply
```

**Before switching:**
```bash
terraform workspace show  # Confirm current
terraform workspace select production  # Switch
terraform workspace show  # Verify new workspace
terraform plan  # Review what would change
```

### Deleting Workspaces (REQUIRES CONFIRMATION)

```bash
# Delete workspace (must have no resources)
terraform workspace delete old-env

# Error if resources exist:
# "Workspace 'old-env' is not empty"

# Must destroy all resources first:
# 1. terraform workspace select old-env
# 2. terraform destroy
# 3. terraform workspace select default
# 4. terraform workspace delete old-env
```

### Workspace Naming Convention

| Workspace | Purpose |
|-----------|---------|
| `default` | Avoid using (no explicit environment) |
| `dev` | Development environment |
| `staging` | Staging environment |
| `pilot` | Pilot/UAT environment |
| `prod` | Production environment |

### Workspace in Code

```hcl
# Reference current workspace in configuration
resource "aws_instance" "web" {
  ami           = var.ami
  instance_type = var.instance_type

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

# Workspace-specific values
variable "instance_type" {
  type = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
}

locals {
  instance_type = lookup(var.instance_type, terraform.workspace, "t3.micro")
}
```

---

## Backend Configuration

### Recommended: S3 + DynamoDB

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true                    # Encryption at rest
    dynamodb_table = "terraform-state-lock"  # State locking

    # Workspace support
    workspace_key_prefix = "workspaces"
    # State keys:
    # - default: project/terraform.tfstate
    # - dev: project/workspaces/dev/terraform.tfstate
    # - prod: project/workspaces/prod/terraform.tfstate
  }
}
```

### S3 Bucket Setup

```hcl
# Create state bucket (one-time setup)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  lifecycle {
    prevent_destroy = true  # Safety - never destroy state bucket
  }
}

# Enable versioning (rollback capability)
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access (security)
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### DynamoDB Table for Locking

```hcl
# Create lock table (one-time setup)
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true  # Safety - never destroy lock table
  }
}
```

### Backend Initialization

```bash
# Initialize backend (first time or after backend config change)
terraform init

# Migrate from local to remote backend
terraform init -migrate-state
# Prompts: "Do you want to copy existing state to the new backend?"
# Answer: yes

# Reconfigure backend (after changing backend settings)
terraform init -reconfigure
```

### State Locking

```bash
# Lock is automatically acquired during apply/destroy
terraform apply
# Acquiring state lock. This may take a few moments...

# If locked by another operation:
Error: Error acquiring the state lock
Lock Info:
  ID:        abc123-def456
  Path:      my-terraform-state/project/terraform.tfstate
  Operation: OperationTypeApply
  Who:       user@hostname
  Version:   1.7.0
  Created:   2026-02-14 10:30:00

# Options:
# 1. Wait for operation to complete
# 2. Force unlock (REQUIRES CONFIRMATION - only if stale)
terraform force-unlock abc123-def456
```

---

## Module Design

### Module Structure

```
modules/
  vpc/
    main.tf       # Resources
    variables.tf  # Input variables
    outputs.tf    # Output values
    README.md     # Usage documentation

  ecs-service/
    main.tf
    variables.tf
    outputs.tf
    README.md
```

### Module Usage

```hcl
# Call module
module "vpc" {
  source = "./modules/vpc"

  cidr_block = "10.0.0.0/16"
  environment = terraform.workspace
}

# Reference module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
}
```

### Module Versioning

```hcl
# Use git tags for module versioning
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.2.0"

  cidr_block = "10.0.0.0/16"
}

# Benefits:
# - Pinned version (reproducible)
# - Can test new versions before upgrading
# - Clear change tracking via tags
```

---

## Environment Isolation Strategies

### Strategy 1: Workspaces (Same Code, Different State)

**Pros:**
- Single codebase for all environments
- Easy to keep environments consistent
- Workspace switching is simple

**Cons:**
- Easy to accidentally apply to wrong workspace
- Harder to have environment-specific code
- Shared variable files

**Example:**
```
project/
  main.tf
  variables.tf
  terraform.tfvars  # OR workspace-specific:
  dev.tfvars
  staging.tfvars
  prod.tfvars

# Usage:
terraform workspace select dev
terraform apply -var-file=dev.tfvars
```

### Strategy 2: Directories (Different Code Per Environment)

**Pros:**
- Complete isolation (can't accidentally affect wrong env)
- Environment-specific configurations
- Clearer separation

**Cons:**
- Code duplication (harder to keep in sync)
- More files to maintain

**Example:**
```
environments/
  dev/
    main.tf
    variables.tf
    terraform.tfvars
  staging/
    main.tf
    variables.tf
    terraform.tfvars
  prod/
    main.tf
    variables.tf
    terraform.tfvars
```

### Strategy 3: Hybrid (Modules + Environment Directories)

**Pros:**
- Reusable modules (DRY)
- Environment isolation
- Best of both approaches

**Cons:**
- More complex structure

**Example:**
```
modules/
  vpc/
  ecs-service/

environments/
  dev/
    main.tf          # Calls modules with dev config
    terraform.tfvars
  prod/
    main.tf          # Calls modules with prod config
    terraform.tfvars
```

---

## Anti-Patterns

| Anti-Pattern | Why Dangerous | Do Instead |
|--------------|---------------|------------|
| `terraform apply -auto-approve` | No review, accidental deletions | Always run plan first, review, then apply |
| Editing .tfstate manually | State corruption | Use terraform state commands (mv, rm, import) |
| Committing state to git | Secrets exposed, merge conflicts | Use remote backend with encryption |
| No state locking | Concurrent operations corrupt state | Use DynamoDB table for state locking |
| Hardcoded values in .tf | Not reusable, environment-specific | Use variables and .tfvars files |
| Using default workspace | Unclear which environment | Create named workspaces (dev, staging, prod) |
| Local state for team projects | No collaboration, conflicts | Use remote backend (S3) |
| No lifecycle rules on state resources | Risk of accidental deletion | Add prevent_destroy to state bucket/table |
| Mixing workspace and directory strategies | Confusing, error-prone | Pick one strategy and stick to it |
| Not pinning module versions | Unexpected changes | Use version tags for modules |

---

## Drift Detection

**Drift:** Cloud resources changed outside of Terraform (manual changes via console/CLI).

### Detecting Drift

```bash
# Run plan to detect drift
terraform plan

# Expected: "No changes"
# Drift detected: Plan shows modifications to resources
```

### Resolving Drift

**Option 1: Import changes to Terraform**
```bash
# Update .tf files to match current cloud state
# Run plan to verify: "No changes"
```

**Option 2: Revert cloud to Terraform state**
```bash
# Run apply to revert manual changes
terraform apply
# Reverts cloud resources to match .tf files
```

### Preventing Drift

- Enable CloudTrail to audit manual changes
- Use IAM policies to restrict manual modifications
- Regular drift detection (automated plan runs)
- Documentation: "All changes via Terraform"

---

## Quick Reference

### Daily Commands (Safe)

```bash
terraform workspace show          # Which environment?
terraform plan                    # Review changes
terraform show                    # Show current state
terraform output                  # Show outputs
terraform state list              # List resources
terraform state show <resource>   # Resource details
```

### Workflow Commands

```bash
# Standard apply workflow
terraform workspace select dev
terraform plan -out=plan.tfplan
# [Review plan output]
terraform apply plan.tfplan

# Import existing resource
# 1. Add resource definition to .tf
# 2. Import
terraform import aws_instance.example i-1234567890abcdef0
# 3. Verify
terraform plan  # Should show "No changes"
```

### Dangerous Commands (Confirmation Required)

```bash
terraform destroy                 # Destroys all resources - REQUIRES CONFIRMATION
terraform apply -target=<resource> # Partial apply - REQUIRES CONFIRMATION
terraform state rm <resource>     # Remove from state - REQUIRES CONFIRMATION
terraform force-unlock <lock-id>  # Break lock - REQUIRES CONFIRMATION
```

### State Operations

```bash
terraform state list                           # List all resources
terraform state show aws_instance.example      # Show resource details
terraform state mv <source> <dest>             # Move/rename resource
terraform state rm <resource>                  # Remove from state
terraform import <address> <id>                # Import existing resource
```

### Workspace Operations

```bash
terraform workspace list          # List workspaces
terraform workspace show          # Current workspace
terraform workspace new <name>    # Create workspace
terraform workspace select <name> # Switch workspace
terraform workspace delete <name> # Delete workspace
```

---

## Audit Trail

All Terraform operations are logged to:

```
~/.claude/logs/terraform-operations.log  # Detailed log
~/.claude/logs/terraform-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-02-14 10:30:45] ACTION=ALLOW COMMAND="terraform plan" WORKSPACE=dev RESULT=read-only USER=donmccarty
[2026-02-14 10:31:02] ACTION=BLOCK COMMAND="terraform destroy" WORKSPACE=prod RESULT=production-destroy USER=donmccarty
[2026-02-14 10:32:15] ACTION=ALLOW COMMAND="terraform apply" WORKSPACE=dev RESULT=success CHANGES="+3 ~1 -0" USER=donmccarty
```

---

## References

- [Terraform Documentation](https://www.terraform.io/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [State Management](https://www.terraform.io/docs/language/state/index.html)
- [Workspaces](https://www.terraform.io/docs/language/state/workspaces.html)
- [S3 Backend](https://www.terraform.io/docs/language/settings/backends/s3.html)
- [Module Development](https://www.terraform.io/docs/language/modules/develop/index.html)
