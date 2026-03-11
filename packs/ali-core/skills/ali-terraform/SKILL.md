---
name: ali-terraform
description: |
  Terraform and Infrastructure as Code patterns for Aliunde projects. Use when:

  PLANNING: Designing Terraform module structure, evaluating state management
  strategies, planning multi-environment deployments, considering provider patterns

  IMPLEMENTATION: Writing HCL modules, configuring backends, defining variables
  and outputs, implementing resource configurations, setting up CI/CD pipelines

  GUIDANCE: Asking about Terraform best practices, module design patterns,
  state management approaches, how to structure IaC projects

  REVIEW: Checking Terraform configurations, reviewing module design, validating
  security settings, auditing state management and variable handling

  Do NOT use for AWS architecture design decisions or service selection (use
  ali-aws-architecture instead), or executing AWS CLI commands (use ali-aws-admin instead)
---

# Terraform / Infrastructure as Code

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Terraform module structure or hierarchy
- Choosing state management strategy (workspaces vs directories)
- Planning multi-environment or multi-account deployments
- Evaluating provider configuration patterns

**Implementation:**
- Writing HCL modules, variables, outputs
- Configuring remote backends (S3, Terraform Cloud)
- Implementing resource configurations with lifecycle rules
- Setting up CI/CD pipelines for infrastructure

**Guidance/Best Practices:**
- Asking about Terraform module design patterns
- Asking about state management approaches
- Asking how to structure variables and outputs
- Asking about testing strategies (terratest, terraform test)

**Review/Validation:**
- Reviewing Terraform configurations for best practices
- Checking security settings in IaC
- Validating module design and reusability
- Auditing state management and sensitive data handling

---

## Key Principles

- **Modules for reuse**: Extract reusable patterns into modules
- **Variables with validation**: Type constraints and validation blocks
- **Remote state always**: Never local state in team environments
- **State locking required**: DynamoDB for S3 backend
- **Sensitive data protection**: Use `sensitive = true`, never commit secrets
- **Version constraints**: Pin providers with `~>` not `>=`
- **Consistent naming**: `{project}-{env}-{resource}` convention
- **Tags everywhere**: Cost allocation, automation, organization
- **Plan before apply**: Always review plan output
- **Test infrastructure**: Checkov, tfsec, terratest

---

## Module Structure

### Standard Layout

```
modules/
  vpc/
    main.tf           # Resources
    variables.tf      # Input variables
    outputs.tf        # Output values
    versions.tf       # Provider/Terraform versions
    README.md         # Usage documentation

environments/
  dev/
    main.tf           # Module calls
    terraform.tfvars  # Environment values
    backend.tf        # State configuration
  prod/
    main.tf
    terraform.tfvars
    backend.tf
```

### Module Call Pattern

```hcl
# environments/prod/main.tf

# Module with explicit dependencies and versioning
module "vpc" {
  source = "../../modules/vpc"

  # Required variables - no defaults
  environment = "prod"
  cidr_block  = "10.0.0.0/16"

  # Optional with sensible defaults
  enable_nat_gateway = true
  single_nat_gateway = false  # HA for prod

  # Tags passed through
  tags = local.common_tags
}

# Reference module outputs
resource "aws_security_group" "app" {
  vpc_id = module.vpc.vpc_id
  # ...
}
```

---

## Variable Patterns

### Variable Definition with Validation

```hcl
# variables.tf

# Required variable - no default
variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Optional with default
variable "instance_type" {
  description = "EC2 instance type for application servers"
  type        = string
  default     = "t3.medium"

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "Only t3 instance types are allowed."
  }
}

# Complex object variable
variable "database_config" {
  description = "Database configuration settings"
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    backup_retention  = optional(number, 7)  # TF 1.3+
  })
}

# Sensitive variable
variable "database_password" {
  description = "Master password for RDS instance"
  type        = string
  sensitive   = true

  validation {
    condition     = length(var.database_password) >= 16
    error_message = "Password must be at least 16 characters."
  }
}
```

### Variable Organization

```hcl
# variables.tf - organized by purpose

# ===========================
# Required Variables
# ===========================

variable "project_name" {
  description = "Project identifier for resource naming"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

# ===========================
# Optional Variables
# ===========================

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = true
}

# ===========================
# Feature Flags
# ===========================

variable "enable_nat_gateway" {
  description = "Create NAT gateway for private subnets"
  type        = bool
  default     = true
}
```

---

## State Management

### S3 Backend Configuration

```hcl
# backend.tf

terraform {
  backend "s3" {
    bucket         = "aliunde-terraform-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"

    # Optional: assume role for cross-account
    # role_arn = "arn:aws:iam::ACCOUNT:role/terraform-backend"
  }
}
```

### State Key Convention

```
# Pattern: {env}/{component}/terraform.tfstate

prod/vpc/terraform.tfstate
prod/database/terraform.tfstate
prod/application/terraform.tfstate

dev/vpc/terraform.tfstate
dev/database/terraform.tfstate
```

### State Locking Table

```hcl
# One-time setup for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Purpose = "Terraform state locking"
  }
}
```

---

## Provider Configuration

### Version Constraints

```hcl
# versions.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Allows 5.x, not 6.x
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}
```

### Provider Aliases for Multi-Region

```hcl
# providers.tf

provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Alias for secondary region
provider "aws" {
  alias  = "west"
  region = "us-west-2"

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Use aliased provider
resource "aws_s3_bucket" "replica" {
  provider = aws.west
  bucket   = "${var.project_name}-replica-${var.environment}"
}
```

---

## Resource Patterns

### Lifecycle Rules

```hcl
resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-${var.environment}"
  # ... other config

  lifecycle {
    # Prevent accidental deletion
    prevent_destroy = true

    # Ignore changes made outside Terraform
    ignore_changes = [
      password,  # Rotated externally
    ]
  }
}

resource "aws_instance" "app" {
  # ... config

  lifecycle {
    # Create new before destroying old (blue-green)
    create_before_destroy = true
  }
}
```

### Count vs For_Each

```hcl
# count - for conditional creation
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

# for_each - for multiple similar resources
resource "aws_subnet" "private" {
  for_each = {
    "a" = "10.0.10.0/24"
    "b" = "10.0.11.0/24"
  }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = "${var.region}${each.key}"

  tags = {
    Name = "${var.project_name}-private-${each.key}"
  }
}

# Reference for_each resources
output "private_subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}
```

### Dynamic Blocks

```hcl
resource "aws_security_group" "app" {
  name   = "${var.project_name}-app"
  vpc_id = var.vpc_id

  # Dynamic ingress rules
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

### Data Sources

```hcl
# Look up existing resources - don't hardcode ARNs
data "aws_caller_identity" "current" {}

data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["${var.project_name}-${var.environment}"]
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use data sources
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  subnet_id     = data.aws_vpc.existing.id
  # ...
}
```

---

## Output Patterns

```hcl
# outputs.tf

# Simple output
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

# Sensitive output
output "database_endpoint" {
  description = "RDS endpoint for application connection"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

# Complex output
output "subnet_info" {
  description = "Map of subnet details"
  value = {
    for k, v in aws_subnet.private : k => {
      id         = v.id
      cidr_block = v.cidr_block
      az         = v.availability_zone
    }
  }
}
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Hardcoded ARNs | Breaks across accounts/regions | Use data sources |
| Local state | No locking, no sharing | S3 + DynamoDB backend |
| `>=` version constraints | Breaking changes | Use `~>` for minor versions |
| Secrets in tfvars | Committed to git | Secrets Manager, environment vars |
| Monolithic main.tf | Hard to maintain | Split by resource type |
| No variable validation | Runtime errors | Add validation blocks |
| Overuse of depends_on | Poor design signal | Fix implicit dependencies |
| Missing lifecycle rules | Accidental deletion | Add prevent_destroy |
| count for named resources | Fragile indexing | Use for_each with keys |
| No default_tags | Inconsistent tagging | Provider default_tags |

---

## Testing & Validation

### Pre-Commit Checks

```bash
# Format check
terraform fmt -check -recursive

# Validation
terraform validate

# Security scan
checkov -d .
tfsec .
```

### Terraform Test (1.6+)

```hcl
# tests/vpc.tftest.hcl

run "creates_vpc_with_correct_cidr" {
  command = plan

  variables {
    cidr_block  = "10.0.0.0/16"
    environment = "test"
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block does not match input"
  }
}
```

### Terratest (Go)

```go
func TestVpcModule(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "environment": "test",
            "cidr_block":  "10.0.0.0/16",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

---

## CI/CD Pipeline

### GitHub Actions Example

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform Init
        run: terraform init
        working-directory: infrastructure/environments/prod

      - name: Terraform Format Check
        run: terraform fmt -check -recursive

      - name: Terraform Validate
        run: terraform validate

      - name: Checkov Scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/

      - name: Terraform Plan
        run: terraform plan -out=tfplan
        working-directory: infrastructure/environments/prod

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: infrastructure/environments/prod/tfplan

  apply:
    needs: plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production  # Requires approval
    steps:
      - uses: actions/checkout@v4

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan
          path: infrastructure/environments/prod

      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
        working-directory: infrastructure/environments/prod
```

---

## Quick Reference

### Common Commands

```bash
# Initialize
terraform init
terraform init -upgrade  # Update providers

# Plan
terraform plan
terraform plan -out=tfplan
terraform plan -target=module.vpc

# Apply
terraform apply
terraform apply tfplan
terraform apply -auto-approve  # CI only

# State operations
terraform state list
terraform state show aws_vpc.main
terraform state mv aws_vpc.old aws_vpc.new
terraform import aws_vpc.main vpc-123456

# Workspace (if using)
terraform workspace list
terraform workspace select prod
terraform workspace new staging
```

### Import Block (TF 1.5+)

```hcl
# Import existing resources declaratively
import {
  to = aws_vpc.main
  id = "vpc-123456789"
}

# Then run: terraform plan -generate-config-out=generated.tf
```

### Moved Block (TF 1.3+)

```hcl
# Refactor without state surgery
moved {
  from = aws_instance.app
  to   = module.compute.aws_instance.app
}
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Terraform modules | `infrastructure/terraform/modules/` |
| Environment configs | `infrastructure/terraform/environments/` |
| Backend setup | `infrastructure/terraform/bootstrap/` |
| CI/CD workflows | `.github/workflows/terraform.yml` |

---

## References

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [AWS Provider Registry](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [Checkov](https://www.checkov.io/)
- [tfsec](https://aquasecurity.github.io/tfsec/)
