---
name: ali-aws-architecture
description: |
  AWS services and architecture patterns for Aliunde projects. Use when:

  PLANNING: Designing AWS infrastructure, evaluating service choices, planning
  VPC/networking, considering security architecture, estimating costs

  IMPLEMENTATION: Writing infrastructure as code, configuring AWS services,
  implementing S3/Lambda/ECS/Aurora integrations, setting up IAM policies

  GUIDANCE: Asking about AWS best practices, Well-Architected Framework,
  which service to use, how to structure AWS resources, cost optimization

  REVIEW: Checking AWS configurations, reviewing IAM policies, validating
  security settings, auditing infrastructure decisions

  Do NOT use for executing AWS CLI commands or operational tasks (use ali-aws-admin
  instead), or Terraform HCL syntax and module design (use ali-terraform instead)
---

# AWS Architecture

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing AWS infrastructure or service architecture
- Choosing between AWS services for a use case
- Planning VPC, networking, or security architecture
- Evaluating cost implications of design choices

**Implementation:**
- Writing Terraform, CloudFormation, or CDK
- Configuring S3, Lambda, ECS, Aurora, or other services
- Setting up IAM roles, policies, or cross-account access
- Implementing AWS SDK integrations (boto3, AWS SDK for Java)

**Guidance/Best Practices:**
- Asking about Well-Architected Framework principles
- Asking which AWS service fits a requirement
- Asking about cost optimization strategies
- Asking how to structure AWS resources

**Review/Validation:**
- Reviewing IAM policies for least privilege
- Checking security group configurations
- Validating encryption settings
- Auditing infrastructure for best practices

---

## Key Principles

- **Least privilege**: IAM policies grant minimum required permissions
- **Defense in depth**: Multiple security layers (VPC, SG, IAM, encryption)
- **Encrypt everything**: At-rest (KMS) and in-transit (TLS)
- **Automate everything**: Infrastructure as code, no manual console changes
- **Design for failure**: Assume components will fail; build resilience
- **Use managed services**: Less operational burden than self-managed
- **Tag everything**: Cost allocation, automation, and organization
- **Monitor and alert**: CloudWatch metrics, alarms, and logs

---

## S3 Patterns

### Bucket Naming and Organization

```
# Naming convention
{company}-{env}-{purpose}-{region}
aliunde-prod-documents-us-east-1
aliunde-dev-edi-files-us-east-1

# Prefix organization (not folders!)
raw/2024/01/15/file.edi          # Date-partitioned raw files
processed/claims/CLM001.parquet   # Processed by type
```

### Encryption

```python
import boto3

s3 = boto3.client('s3')

# Server-side encryption with KMS (SSE-KMS)
s3.put_object(
    Bucket='aliunde-prod-documents',
    Key='sensitive/file.pdf',
    Body=content,
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='alias/documents-key'  # Use alias, not key ID
)

# Bucket default encryption (set via Terraform/CloudFormation)
# All objects encrypted automatically
```

### Pre-signed URLs

```python
from datetime import datetime
import boto3

s3 = boto3.client('s3')

# Generate download URL (expires in 1 hour)
url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'aliunde-prod-documents',
        'Key': 'client/123/tax-return.pdf'
    },
    ExpiresIn=3600  # seconds
)

# Generate upload URL
upload_url = s3.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'aliunde-prod-uploads',
        'Key': f'pending/{uuid.uuid4()}.pdf',
        'ContentType': 'application/pdf'
    },
    ExpiresIn=900  # 15 minutes for upload
)
```

### Lifecycle Rules

```hcl
# Terraform example
resource "aws_s3_bucket_lifecycle_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    id     = "archive-old-files"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"  # Infrequent Access
    }

    transition {
      days          = 365
      storage_class = "GLACIER"
    }

    # Keep for 7 years (IRS requirement), then delete
    expiration {
      days = 2555  # ~7 years
    }
  }

  rule {
    id     = "clean-incomplete-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

---

## Lambda Patterns

### Cold Start Mitigation

```python
# Initialize outside handler (runs once per container)
import boto3
from functools import lru_cache

# These run during cold start, not every invocation
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

@lru_cache(maxsize=100)
def get_config(key: str) -> str:
    """Cache config lookups within container lifetime."""
    return ssm.get_parameter(Name=key)['Parameter']['Value']

def handler(event, context):
    # Handler code uses pre-initialized clients
    item = table.get_item(Key={'id': event['id']})
    return item
```

### Error Handling

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    try:
        result = process_event(event)
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    except ValidationError as e:
        logger.warning(f"Validation failed: {e}")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }
    except Exception as e:
        logger.exception("Unexpected error")  # Logs full traceback
        # Re-raise for Lambda to handle (DLQ, retry, etc.)
        raise
```

### Lambda Layers

```yaml
# SAM template
Resources:
  CommonLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: common-dependencies
      ContentUri: layers/common/
      CompatibleRuntimes:
        - python3.11
      RetentionPolicy: Retain

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.11
      Layers:
        - !Ref CommonLayer
```

---

## ECS/Fargate Patterns

### Task Definition

```json
{
  "family": "edi-parser",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/edi-parser-task-role",
  "containerDefinitions": [
    {
      "name": "parser",
      "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/edi-parser:latest",
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/edi-parser",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:db-password"
        }
      ],
      "environment": [
        {"name": "ENV", "value": "production"},
        {"name": "AWS_REGION", "value": "us-east-1"}
      ]
    }
  ]
}
```

### Service Auto-Scaling

```hcl
# Terraform - Scale based on CPU
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.api.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Aurora PostgreSQL Patterns

### Connection Pooling

```python
# Use RDS Proxy or application-level pooling
from psycopg2 import pool

# Application-level pool
connection_pool = pool.ThreadedConnectionPool(
    minconn=5,
    maxconn=20,
    host=os.environ['DB_HOST'],
    database=os.environ['DB_NAME'],
    user=os.environ['DB_USER'],
    password=os.environ['DB_PASSWORD']
)

def get_connection():
    return connection_pool.getconn()

def release_connection(conn):
    connection_pool.putconn(conn)
```

### Read Replica Usage

```python
import os

# Use environment variables to switch endpoints
WRITER_HOST = os.environ['DB_WRITER_HOST']  # For writes
READER_HOST = os.environ['DB_READER_HOST']  # For reads

def get_read_connection():
    """Use reader endpoint for SELECT queries."""
    return psycopg2.connect(host=READER_HOST, ...)

def get_write_connection():
    """Use writer endpoint for INSERT/UPDATE/DELETE."""
    return psycopg2.connect(host=WRITER_HOST, ...)
```

---

## Bedrock / Claude API Patterns

### Model Selection

| Model | Use Case | Cost |
|-------|----------|------|
| Claude 3 Haiku | High-volume, simple tasks | Lowest |
| Claude 3.5 Sonnet | Balanced capability/cost | Medium |
| Claude 3 Opus | Complex reasoning | Highest |

### Bedrock Integration

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

def invoke_claude(prompt: str, max_tokens: int = 1000) -> str:
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': max_tokens,
            'messages': [
                {'role': 'user', 'content': prompt}
            ]
        })
    )
    result = json.loads(response['body'].read())
    return result['content'][0]['text']
```

---

## KMS Patterns

### Key Hierarchy

```
# Account-level structure
aws/s3           # AWS-managed key for S3 (default)
alias/documents  # Customer-managed for sensitive documents
alias/database   # Customer-managed for RDS encryption
alias/secrets    # Customer-managed for Secrets Manager
```

### Key Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM policies",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ACCOUNT:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow service use",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::ACCOUNT:role/edi-parser-role"},
      "Action": ["kms:Decrypt", "kms:GenerateDataKey"],
      "Resource": "*"
    }
  ]
}
```

---

## Secrets Manager Patterns

### Retrieving Secrets

```python
import boto3
import json
from functools import lru_cache

secrets_client = boto3.client('secretsmanager')

@lru_cache(maxsize=10)
def get_secret(secret_name: str) -> dict:
    """Retrieve and cache secret (cache per Lambda container)."""
    response = secrets_client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# Usage
db_creds = get_secret('prod/database/credentials')
connection = psycopg2.connect(
    host=db_creds['host'],
    user=db_creds['username'],
    password=db_creds['password']
)
```

### Secret Rotation

```hcl
# Terraform - Enable rotation
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

---

## VPC Patterns

### Standard Layout

```
VPC: 10.0.0.0/16

Public Subnets (NAT, ALB):
  10.0.1.0/24 (us-east-1a)
  10.0.2.0/24 (us-east-1b)

Private Subnets (ECS, Lambda):
  10.0.10.0/24 (us-east-1a)
  10.0.11.0/24 (us-east-1b)

Database Subnets (RDS):
  10.0.20.0/24 (us-east-1a)
  10.0.21.0/24 (us-east-1b)
```

### VPC Endpoints (Cost Savings)

```hcl
# Gateway endpoints (free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
}

# Interface endpoints (cost per hour + data)
resource "aws_vpc_endpoint" "secrets" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## IAM Patterns

### Least Privilege Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSpecificBucket",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::aliunde-prod-edi-files",
        "arn:aws:s3:::aliunde-prod-edi-files/*"
      ]
    },
    {
      "Sid": "WriteProcessedFiles",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::aliunde-prod-processed/*"
    },
    {
      "Sid": "DecryptWithSpecificKey",
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:us-east-1:ACCOUNT:key/KEY-ID"
    }
  ]
}
```

### Cross-Account Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::OTHER-ACCOUNT:role/their-role"},
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {"sts:ExternalId": "shared-secret-id"}
      }
    }
  ]
}
```

---

## SES Patterns

### Sending Email

```python
import boto3

ses = boto3.client('ses', region_name='us-east-1')

def send_email(to: str, subject: str, body_html: str, body_text: str):
    ses.send_email(
        Source='noreply@aliunde.com',
        Destination={'ToAddresses': [to]},
        Message={
            'Subject': {'Data': subject},
            'Body': {
                'Html': {'Data': body_html},
                'Text': {'Data': body_text}
            }
        },
        ConfigurationSetName='tracking'  # For open/click tracking
    )
```

### Bounce Handling

```python
# SNS notification handler for bounces
def handle_ses_notification(event):
    message = json.loads(event['Records'][0]['Sns']['Message'])

    if message['notificationType'] == 'Bounce':
        bounce = message['bounce']
        for recipient in bounce['bouncedRecipients']:
            email = recipient['emailAddress']
            # Mark email as undeliverable in your database
            mark_email_invalid(email, reason='bounce')

    elif message['notificationType'] == 'Complaint':
        # User marked as spam - stop sending
        for recipient in message['complaint']['complainedRecipients']:
            unsubscribe_email(recipient['emailAddress'])
```

---

## EventBridge Patterns

### Event Rule

```hcl
resource "aws_cloudwatch_event_rule" "s3_upload" {
  name        = "edi-file-uploaded"
  description = "Trigger on new EDI file upload"

  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = { name = ["aliunde-prod-edi-files"] }
      object = { key = [{ prefix = "incoming/" }] }
    }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.s3_upload.name
  target_id = "process-edi"
  arn       = aws_lambda_function.process_edi.arn
}
```

---

## Well-Architected Framework

### Five Pillars Summary

| Pillar | Key Questions |
|--------|---------------|
| **Operational Excellence** | How do you deploy? How do you respond to incidents? |
| **Security** | How do you protect data? How do you manage permissions? |
| **Reliability** | How do you handle failure? How do you recover? |
| **Performance** | How do you select resources? How do you monitor? |
| **Cost Optimization** | How do you manage spend? How do you right-size? |

---

## Cost Optimization

### Quick Wins

| Service | Optimization |
|---------|--------------|
| S3 | Lifecycle rules → IA → Glacier; Intelligent Tiering |
| Lambda | Right-size memory; ARM64 (Graviton) = 20% cheaper |
| ECS | Spot for non-critical; right-size CPU/memory |
| RDS | Reserved instances for production; stop dev instances |
| NAT Gateway | Use VPC endpoints for S3/DynamoDB (free) |
| Data Transfer | Keep traffic in same AZ when possible |

### Cost Allocation Tags

```hcl
# Apply to all resources
default_tags {
  tags = {
    Project     = "ingestion-engine"
    Environment = "production"
    Owner       = "platform-team"
    CostCenter  = "engineering"
  }
}
```

---

## CloudWatch Metrics and Alarms

### Custom Metrics

```python
import boto3
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

def put_custom_metric(
    metric_name: str,
    value: float,
    unit: str = 'Count',
    dimensions: dict = None
):
    """Publish custom metric to CloudWatch."""
    metric_data = {
        'MetricName': metric_name,
        'Value': value,
        'Unit': unit,
        'Timestamp': datetime.utcnow()
    }

    if dimensions:
        metric_data['Dimensions'] = [
            {'Name': k, 'Value': v} for k, v in dimensions.items()
        ]

    cloudwatch.put_metric_data(
        Namespace='Aliunde/Application',
        MetricData=[metric_data]
    )

# Usage
put_custom_metric(
    'EDIFilesProcessed',
    value=150,
    dimensions={'Environment': 'production', 'FileType': '835'}
)

put_custom_metric(
    'ProcessingLatencyMs',
    value=1250.5,
    unit='Milliseconds',
    dimensions={'Pipeline': 'edi-835-ingestion'}
)
```

### CloudWatch Alarms (Terraform)

```hcl
# High error rate alarm
resource "aws_cloudwatch_metric_alarm" "error_rate" {
  alarm_name          = "edi-parser-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = 300  # 5 minutes
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "EDI parser error rate too high"

  dimensions = {
    FunctionName = aws_lambda_function.edi_parser.function_name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
}

# Database connections alarm
resource "aws_cloudwatch_metric_alarm" "db_connections" {
  alarm_name          = "aurora-connection-limit"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 60
  statistic           = "Average"
  threshold           = 80  # 80% of max connections

  dimensions = {
    DBClusterIdentifier = aws_rds_cluster.main.cluster_identifier
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

### CloudWatch Insights Queries

```sql
-- Find Lambda errors with context
fields @timestamp, @message, @requestId
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50

-- Analyze Lambda duration percentiles
stats avg(duration), percentile(duration, 95), percentile(duration, 99) by bin(5m)
| sort @timestamp desc

-- Find slow API requests
fields @timestamp, path, responseTime
| filter responseTime > 1000
| sort responseTime desc
| limit 100

-- Count events by type
stats count(*) as count by eventType
| sort count desc
```

---

## AWS Batch

For large-scale batch processing jobs:

### Job Definition

```hcl
resource "aws_batch_job_definition" "edi_processor" {
  name = "edi-processor"
  type = "container"

  container_properties = jsonencode({
    image      = "${aws_ecr_repository.edi_processor.repository_url}:latest"
    vcpus      = 4
    memory     = 8192

    jobRoleArn = aws_iam_role.batch_job_role.arn

    environment = [
      { name = "ENV", value = "production" },
      { name = "S3_BUCKET", value = aws_s3_bucket.data.bucket }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.batch.name
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "edi-processor"
      }
    }

    mountPoints = []
    volumes     = []
  })

  retry_strategy {
    attempts = 3
  }

  timeout {
    attempt_duration_seconds = 3600  # 1 hour max
  }
}
```

### Submit Batch Job

```python
import boto3

batch = boto3.client('batch')

def submit_batch_job(
    job_name: str,
    file_key: str,
    job_queue: str = 'edi-processing-queue'
) -> str:
    """Submit AWS Batch job for EDI processing."""
    response = batch.submit_job(
        jobName=job_name,
        jobQueue=job_queue,
        jobDefinition='edi-processor',
        containerOverrides={
            'environment': [
                {'name': 'INPUT_FILE', 'value': file_key},
                {'name': 'JOB_ID', 'value': job_name}
            ]
        },
        retryStrategy={
            'attempts': 3
        }
    )
    return response['jobId']

# Array job for parallel processing
def submit_array_job(file_keys: list[str]) -> str:
    """Submit array job to process multiple files in parallel."""
    response = batch.submit_job(
        jobName='edi-batch-processing',
        jobQueue='edi-processing-queue',
        jobDefinition='edi-processor',
        arrayProperties={
            'size': len(file_keys)  # Creates N parallel jobs
        },
        containerOverrides={
            'environment': [
                {'name': 'FILE_MANIFEST', 'value': ','.join(file_keys)}
            ]
        }
    )
    return response['jobId']
```

### Compute Environment

```hcl
resource "aws_batch_compute_environment" "edi_processing" {
  compute_environment_name = "edi-processing"
  type                     = "MANAGED"

  compute_resources {
    type      = "FARGATE_SPOT"  # Cost-effective for batch
    max_vcpus = 256

    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.batch.id]
  }

  service_role = aws_iam_role.batch_service_role.arn
}

resource "aws_batch_job_queue" "edi_processing" {
  name     = "edi-processing-queue"
  state    = "ENABLED"
  priority = 1

  compute_environment_order {
    order               = 1
    compute_environment = aws_batch_compute_environment.edi_processing.arn
  }
}
```

---

## Parameter Store vs Secrets Manager

| Feature | Parameter Store | Secrets Manager |
|---------|-----------------|-----------------|
| **Cost** | Free (standard), $0.05/param (advanced) | $0.40/secret/month |
| **Rotation** | Manual | Automatic with Lambda |
| **Size** | 8KB max (advanced) | 64KB max |
| **Versioning** | Yes | Yes |
| **Cross-account** | Limited | Yes |
| **Use for** | Config, feature flags | Credentials, API keys |

```python
import boto3

ssm = boto3.client('ssm')
secrets = boto3.client('secretsmanager')

# Parameter Store - for configuration
def get_parameter(name: str) -> str:
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response['Parameter']['Value']

feature_flag = get_parameter('/app/features/new-ui-enabled')
api_endpoint = get_parameter('/app/config/api-endpoint')

# Secrets Manager - for credentials
def get_secret(name: str) -> dict:
    response = secrets.get_secret_value(SecretId=name)
    return json.loads(response['SecretString'])

db_creds = get_secret('prod/database/aurora')
stripe_key = get_secret('prod/stripe/api-key')
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Terraform modules | `ali-ai-infra/terraform/` |
| Lambda functions | `ali-ai-acctg/lambdas/` |
| ECS task definitions | `ali-ai-infra/ecs/` |
| IAM policies | `ali-ai-infra/iam/` |
| boto3 usage examples | `ali-ai-ingestion-engine/system/` |

---

## References

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [Boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Pricing Calculator](https://calculator.aws/)
