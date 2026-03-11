---
name: ali-aws-admin
description: |
  AWS operations with safety, cost, and security enforcement. Use when:

  PLANNING: Planning AWS infrastructure, evaluating service choices, designing
  security architecture, estimating costs, considering SDLC environments

  IMPLEMENTATION: Executing AWS CLI commands, creating/modifying resources,
  configuring services, managing IAM, deploying applications

  GUIDANCE: Asking about AWS best practices, cost optimization, security
  hardening, Well-Architected Framework, environment separation

  REVIEW: Reviewing AWS configurations, auditing security settings, checking
  cost implications, validating infrastructure decisions

  Do NOT use for AWS architecture design decisions or service selection guidance
  (use ali-aws-architecture instead), or Terraform/IaC code (use ali-terraform instead)
---

# AWS Admin

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing AWS infrastructure
- Evaluating service choices (Lambda vs ECS vs EC2, etc.)
- Planning VPC/networking architecture
- Designing security architecture
- Estimating costs and rightsizing

**Implementation:**
- Executing AWS CLI commands
- Creating or modifying AWS resources
- Configuring IAM policies and roles
- Deploying applications
- Managing secrets and parameters

**Guidance/Best Practices:**
- Cost optimization strategies
- Security hardening
- Well-Architected Framework
- Environment separation (dev/staging/prod)
- Disaster recovery planning

**Review/Validation:**
- Auditing security configurations
- Reviewing IAM policies
- Checking cost implications
- Validating infrastructure decisions

---

## Core Principles

### 1. Security First (Non-Negotiable)

| Rule | Rationale |
|------|-----------|
| **Never make databases public** | RDS/Aurora public access = data breach waiting to happen |
| **Never open 0.0.0.0/0 ingress** | Except ALB/CloudFront on 80/443 only |
| **Always encrypt at rest** | S3, RDS, EBS, Secrets Manager - no exceptions |
| **Always encrypt in transit** | TLS 1.2+ everywhere |
| **Least privilege IAM** | Start with zero permissions, add only what's needed |
| **No hardcoded credentials** | Use IAM roles, Secrets Manager, Parameter Store |
| **Enable CloudTrail** | Audit logging for all API calls |
| **Enable VPC Flow Logs** | Network traffic visibility |

### 2. Cost Consciousness

| Practice | Impact |
|----------|--------|
| **Right-size instances** | Don't start with xlarge - scale up when needed |
| **Use Reserved/Savings Plans** | 30-70% savings for predictable workloads |
| **Delete unused resources** | Orphaned EBS, old snapshots, idle NAT Gateways |
| **Set billing alerts** | Know before you're surprised |
| **Prefer serverless** | Pay only for what you use |
| **Use spot instances** | For fault-tolerant workloads (80% savings) |
| **Review Cost Explorer weekly** | Catch anomalies early |

### 3. SDLC Awareness

| Environment | Constraints |
|-------------|-------------|
| **dev** | Smallest instances, may be ephemeral, auto-shutdown OK |
| **staging** | Production-like config, smaller scale, limited retention |
| **pilot** | Production-like, real users, full monitoring |
| **prod** | Full redundancy, no shortcuts, change management required |

**Environment Indicators:**
- Resource names contain environment (e.g., `tax-practice-prod-db`)
- Tags: `Environment=dev|staging|pilot|prod`
- Separate AWS accounts preferred (at minimum, separate VPCs)

---

## Dangerous Operations (REQUIRE CONFIRMATION)

### Critical Risk (Production Impact)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `aws rds delete-db-instance` | Data loss | "Yes, delete database [name]" |
| `aws rds delete-db-cluster` | Data loss | "Yes, delete database cluster [name]" |
| `aws s3 rb --force` | Data loss | "Yes, delete bucket [name]" |
| `aws s3 rm --recursive` | Data loss | "Yes, delete all objects in [bucket]" |
| `aws ec2 terminate-instances` | Service disruption | "Yes, terminate instance [id]" |
| `aws ecs delete-service` | Service disruption | "Yes, delete service [name]" |
| `aws ecs delete-cluster` | Service disruption | "Yes, delete cluster [name]" |
| `aws lambda delete-function` | Service disruption | "Yes, delete function [name]" |

### High Risk (Security/Cost)

| Operation | Risk | Confirmation Required |
|-----------|------|----------------------|
| `--publicly-accessible` | Security breach | "Yes, make [resource] publicly accessible" |
| `--cidr 0.0.0.0/0` | Security breach | "Yes, open to all IPs" |
| `aws iam create-policy` | Privilege escalation | Review policy document first |
| `aws iam attach-*-policy` | Privilege escalation | Review what's being attached |
| `aws iam put-*-policy` | Privilege escalation | Review policy document first |
| Large instance types | Cost explosion | "Yes, create [type] instance (~$X/month)" |
| `aws secretsmanager delete-secret` | Credential loss | "Yes, delete secret [name]" |

### Medium Risk (Requires Review)

| Operation | Risk | Action |
|-----------|------|--------|
| Security group modifications | Network exposure | Show before/after rules |
| VPC/subnet changes | Connectivity impact | Explain impact |
| Route table modifications | Traffic routing | Explain impact |
| NAT Gateway creation | Cost ($45/month + data) | Confirm cost |

---

## Safe Operations (No Confirmation Needed)

```bash
# Read-only operations
aws sts get-caller-identity
aws s3 ls
aws ec2 describe-*
aws rds describe-*
aws ecs describe-*
aws ecs list-*
aws lambda list-functions
aws lambda get-function
aws cloudwatch get-metric-*
aws logs describe-log-groups
aws logs get-log-events
aws iam list-*
aws iam get-*

# Low-risk operations
aws s3 cp (to S3, not destructive)
aws logs tail
aws ecs update-service --force-new-deployment
aws ssm get-parameter
```

---

## Instance Type Cost Reference

Always consider cost when creating instances:

| Type | vCPU | Memory | ~Monthly Cost |
|------|------|--------|---------------|
| t3.micro | 2 | 1 GB | $8 |
| t3.small | 2 | 2 GB | $16 |
| t3.medium | 2 | 4 GB | $32 |
| t3.large | 2 | 8 GB | $65 |
| t3.xlarge | 4 | 16 GB | $130 |
| m6i.large | 2 | 8 GB | $77 |
| m6i.xlarge | 4 | 16 GB | $154 |
| r6i.large | 2 | 16 GB | $101 |
| r6i.xlarge | 4 | 32 GB | $202 |

**RDS Instance Costs (additional):**
| Type | ~Monthly Cost |
|------|---------------|
| db.t3.micro | $15 |
| db.t3.small | $29 |
| db.t3.medium | $58 |
| db.t3.large | $116 |
| db.r6g.large | $175 |

**Always ask:** "Is this the smallest instance that will work?"

---

## Security Group Best Practices

### Allowed Patterns

```bash
# ALB/CloudFront - public HTTPS only
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Internal services - VPC CIDR only
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 5432 \
  --cidr 10.0.0.0/16  # VPC CIDR
```

### Forbidden Patterns

```bash
# NEVER - Database open to internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 5432 \
  --cidr 0.0.0.0/0  # BLOCKED

# NEVER - SSH open to internet
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0  # BLOCKED - use SSM Session Manager

# NEVER - All ports open
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxx \
  --protocol -1 \
  --cidr 0.0.0.0/0  # BLOCKED
```

---

## IAM Best Practices

### Policy Design

1. **Start with AWS managed policies** when possible
2. **Use conditions** to restrict scope
3. **Never use `*` for resources** in production
4. **Require MFA** for sensitive operations
5. **Use service-linked roles** when available

### Example: Minimal ECS Task Role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/my-prefix/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:*:secret:my-app/*"
    }
  ]
}
```

### Anti-Patterns

```json
// NEVER - God mode
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// NEVER - Unrestricted S3
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

---

## Environment-Specific Rules

### Production Resources

Before ANY modification to production:

1. **Verify environment** - Check resource tags/names
2. **Check change window** - Is this an approved maintenance window?
3. **Confirm backup exists** - Recent snapshot/backup available?
4. **Have rollback plan** - How to undo if something goes wrong?
5. **Notify stakeholders** - Who needs to know?

### Development Resources

More flexibility, but still:
- Don't leave expensive resources running overnight
- Don't create public endpoints
- Don't use production credentials
- Do tag with owner for cleanup

---

## Cost Optimization Checklist

Before creating any resource:

- [ ] Is this the smallest instance type that will work?
- [ ] Can this be serverless instead?
- [ ] Is there an existing resource we can reuse?
- [ ] Is this tagged for cost allocation?
- [ ] Is there an auto-shutdown schedule for dev/staging?
- [ ] Have we set billing alerts?

After creating resources:
- [ ] Add to Cost Explorer report
- [ ] Set up CloudWatch alarms for anomalies
- [ ] Schedule review in 2 weeks to rightsize

---

## Pre-Execution Checklist

Before ANY AWS operation:

1. **Verify identity**
   ```bash
   aws sts get-caller-identity
   ```

2. **Verify region**
   ```bash
   aws configure get region
   ```

3. **Verify environment** (from resource name/tags)
   - Is this dev, staging, or prod?
   - Are you sure?

4. **Estimate cost impact** (for create operations)

5. **Check for dependencies** (for delete operations)

---

## Audit Trail

All AWS CLI operations are logged by aws-guard to:
```
~/.claude/logs/aws-operations.log  # Detailed log
~/.claude/logs/aws-audit.log       # Structured audit trail
```

### Audit Log Format

```
[2026-01-15 10:30:45] ACTION=ALLOW COMMAND="aws s3 ls" RESULT=read-only USER=donmccarty
[2026-01-15 10:31:02] ACTION=BLOCK COMMAND="aws rds modify-db-instance --publicly-accessible" RESULT=security-violation USER=donmccarty
```

---

## Quick Reference

### Daily Commands (Safe)

```bash
aws sts get-caller-identity        # Who am I?
aws s3 ls s3://bucket/             # List S3 contents
aws ecs list-services --cluster X  # List services
aws ecs describe-services ...      # Service details
aws logs tail /aws/ecs/...         # Tail logs
aws ssm get-parameter --name X     # Get config
```

### Deployment Commands (Review Required)

```bash
aws ecs update-service \
  --cluster X \
  --service Y \
  --force-new-deployment           # Rolling deploy

aws lambda update-function-code \
  --function-name X \
  --image-uri Y                    # Lambda deploy
```

### Dangerous Commands (Confirmation Required)

```bash
aws rds delete-db-instance ...     # REQUIRES CONFIRMATION
aws s3 rb s3://bucket --force      # REQUIRES CONFIRMATION
aws ec2 terminate-instances ...    # REQUIRES CONFIRMATION
```

---

## References

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Security Best Practices](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
- [AWS Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
