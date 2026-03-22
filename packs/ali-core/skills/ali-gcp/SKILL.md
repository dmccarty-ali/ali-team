---
name: ali-gcp
description: |
  Google Cloud Platform services, architecture, and best practices. Use when:

  PLANNING: Designing GCP infrastructure, evaluating service choices, planning
  IAM strategy, architecting VPC networking, choosing compute options (GKE,
  Cloud Run, Compute Engine, Cloud Functions)

  IMPLEMENTATION: Writing gcloud commands, configuring IAM policies, setting up
  GKE clusters, deploying Cloud Run services, configuring Cloud Storage, writing
  Terraform for GCP resources

  GUIDANCE: Asking about GCP services, best practices for GCP architecture,
  cost optimization, how to configure GCP networking, security recommendations

  REVIEW: Auditing GCP IAM policies, reviewing infrastructure configs, validating
  security posture against GCP Well-Architected Framework, checking cost efficiency
---

# Google Cloud Platform

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing GCP infrastructure or choosing between GCP services
- Planning IAM strategy, service accounts, or org policies
- Architecting VPC networking, Private Google Access, or Cloud NAT
- Evaluating compute options (GKE vs Cloud Run vs Compute Engine vs Cloud Functions)
- Estimating cost implications of GCP design choices

**Implementation:**
- Writing gcloud CLI commands for any GCP resource
- Configuring IAM bindings, custom roles, or Workload Identity Federation
- Setting up GKE clusters, node pools, or Autopilot
- Deploying Cloud Run services or Cloud Functions
- Configuring Cloud Storage buckets, lifecycle rules, or access controls
- Writing Terraform HCL for GCP resources (google provider)

**Guidance/Best Practices:**
- Asking about GCP service selection for a use case
- Asking about cost optimization strategies on GCP
- Asking how to structure GCP project hierarchy
- Asking about GCP networking best practices or security configuration
- Asking about GCP Well-Architected Framework recommendations

**Review/Validation:**
- Auditing GCP IAM policies for least privilege
- Reviewing Terraform configs targeting GCP
- Checking Cloud Storage access configurations
- Validating security posture (VPC Service Controls, org policies, Binary Authorization)
- Reviewing cost efficiency of GCP resource choices

## When NOT to Use This Skill

- Executing gcloud commands operationally (delegate to ali-google-admin)
- Writing Terraform module structure or HCL syntax (use ali-terraform skill)
- Google Workspace admin operations (use ali-google-workspace skill)
- Google-specific developer APIs (Apps Script, Workspace Add-ins)

---

## Key Principles

- **Least privilege IAM**: Grant minimum required permissions; prefer predefined roles over basic (Owner/Editor/Viewer) roles
- **Project hierarchy first**: Org > Folder > Project structure enables policy inheritance and billing isolation
- **VPC is the network boundary**: All resources in a VPC; use Private Google Access to avoid public IPs
- **Managed services preferred**: Cloud Run over GCE, Cloud SQL over self-managed, Pub/Sub over self-managed queues
- **Workload Identity over service account keys**: Never download service account keys if Workload Identity can be used
- **Labels everywhere**: Apply labels for cost allocation, environment, and ownership to all resources
- **Encrypt with CMEK for regulated data**: Customer-Managed Encryption Keys for compliance requirements
- **Audit logs always on**: Admin Activity logs are always on; enable Data Access logs for sensitive APIs

---

## IAM and Security

### IAM Role Hierarchy

| Role Type | Example | When to Use |
|-----------|---------|-------------|
| Basic (Primitive) | roles/owner, roles/editor | Never in production — overprivileged |
| Predefined | roles/storage.objectViewer | Default choice — scoped to service |
| Custom | custom/myOrgRole | When predefined is too broad |

### Service Account Best Practices

```bash
# Create service account with minimal description
gcloud iam service-accounts create my-service \
  --display-name="My Service SA" \
  --project=my-project

# Grant only the specific role needed
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-service@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Never create keys unless absolutely required — use Workload Identity instead
# If key is required, rotate on a schedule and store in Secret Manager
```

### Workload Identity Federation (No Keys)

Preferred pattern for GKE workloads — eliminates service account key files:

```bash
# Create Kubernetes service account
kubectl create serviceaccount my-ksa --namespace=my-namespace

# Bind KSA to GCP SA via Workload Identity
gcloud iam service-accounts add-iam-policy-binding \
  my-gcp-sa@my-project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:my-project.svc.id.goog[my-namespace/my-ksa]"

# Annotate the KSA
kubectl annotate serviceaccount my-ksa \
  --namespace=my-namespace \
  iam.gke.io/gcp-service-account=my-gcp-sa@my-project.iam.gserviceaccount.com
```

### Org Policies (Guardrails)

Key org policies to enforce:

| Policy | Constraint | Effect |
|--------|-----------|--------|
| Block public Cloud Storage | constraints/storage.publicAccessPrevention | Prevent accidental public buckets |
| Restrict service account key creation | constraints/iam.disableServiceAccountKeyCreation | Force Workload Identity |
| Restrict resource locations | constraints/gcp.resourceLocations | Keep data in specific regions |
| Require OS login on VMs | constraints/compute.requireOsLogin | Centralize SSH access |
| Restrict allowed external IPs | constraints/compute.restrictCloudSQLInstances | No public Cloud SQL |

---

## Compute Options: Decision Matrix

| Compute Option | Best For | Avoid When |
|----------------|----------|------------|
| **Cloud Run** | Stateless HTTP services, event-driven, variable traffic | Long-running jobs, stateful workloads |
| **Cloud Functions (gen 2)** | Event triggers, lightweight handlers, < 9 minutes | Complex services, > 32GB memory needed |
| **GKE Standard** | Existing K8s workloads, custom node configs, GPU/TPU | Small teams without K8s expertise |
| **GKE Autopilot** | K8s without node management, cost optimization | Workloads needing node-level customization |
| **Compute Engine** | Custom OS, GPU workloads, lift-and-shift VMs | New greenfield services (prefer managed) |
| **Cloud Batch** | Large-scale batch jobs, HPC workloads | Real-time processing |

### Cloud Run vs GKE

Choose **Cloud Run** when:
- HTTP/gRPC request-response pattern
- Stateless service with variable traffic (scales to zero)
- Deployment simplicity is priority
- Cost scales with usage (not always-on baseline)

Choose **GKE** when:
- Existing Kubernetes manifests/Helm charts
- WebSocket or long-lived connections needed
- Multiple containers per pod
- GPU/TPU requirements
- Fine-grained node pool control

---

## Networking

### VPC Design

```
# Standard multi-tier VPC layout
VPC: 10.0.0.0/16

Public (Load Balancers only — no direct VM internet access):
  10.0.1.0/24 (us-central1)

Private (Application tier — Cloud Run, GKE, GCE):
  10.0.10.0/24 (us-central1)

Data (Managed services — Cloud SQL, Memorystore):
  10.0.20.0/24 (us-central1)
```

### Private Google Access

Enable on subnets so VMs/pods without public IPs can reach Google APIs:

```bash
ALIUNDE_GOOGLE_ADMIN=1 gcloud compute networks subnets update [subnet-name] \
  --region=[region] \
  --enable-private-ip-google-access
```

### Cloud NAT (Outbound from Private Subnets)

```bash
# Create Cloud Router first
ALIUNDE_GOOGLE_ADMIN=1 gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# Add NAT config to router
ALIUNDE_GOOGLE_ADMIN=1 gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```

### VPC Service Controls

Use for highly sensitive data — creates a security perimeter around GCP services:

```bash
# List access policies
gcloud access-context-manager policies list --organization=[org-id]

# Describe a service perimeter
gcloud access-context-manager perimeters describe [perimeter] \
  --policy=[policy-id]
```

---

## Storage

### Cloud Storage Class Selection

| Class | Use Case | Retrieval Cost |
|-------|----------|---------------|
| Standard | Frequently accessed data | None |
| Nearline | Accessed < once/month | Yes |
| Coldline | Accessed < once/quarter | Yes |
| Archive | Accessed < once/year | Yes (highest) |

### Cloud Storage Lifecycle Rules

```json
{
  "rule": [
    {
      "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
      "condition": {"age": 30}
    },
    {
      "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
      "condition": {"age": 90}
    },
    {
      "action": {"type": "Delete"},
      "condition": {"age": 365}
    }
  ]
}
```

### Database Selection

| Service | Best For | Avoid When |
|---------|----------|------------|
| **Cloud SQL (PostgreSQL)** | Relational data, existing PostgreSQL workloads | Multi-region writes needed |
| **AlloyDB** | High-performance PostgreSQL, analytics + OLTP | Simple workloads (Cloud SQL sufficient) |
| **Cloud Spanner** | Global relational data, multi-region writes | Cost-sensitive workloads |
| **Firestore** | Document data, mobile/web backends, offline sync | Complex joins or transactions |
| **Bigtable** | Time-series, IoT, wide-column at scale | Small datasets, complex queries |
| **Memorystore (Redis)** | Session cache, Pub/Sub, rate limiting | Persistent primary storage |

---

## Cost Optimization

### Machine Type Guidance

| Tier | Machine Types | Monthly Cost (approx) | Use Case |
|------|--------------|----------------------|----------|
| Micro | e2-micro, e2-small | $5-$15 | Dev, CI runners |
| Standard | e2-standard-2, e2-standard-4 | $50-$100 | App servers |
| High-perf | n2-standard-8, c3-standard-8 | $200-$400 | DB servers, compute-heavy |
| Large | n2-standard-16+ | $400+ | Requires confirmation |

### Committed Use Discounts (CUDs)

- 1-year CUD: ~37% off on-demand pricing
- 3-year CUD: ~55% off on-demand pricing
- Apply to: Compute Engine, Cloud SQL, GKE nodes
- Do not commit until workload is proven stable

### Preemptible / Spot VMs

- Up to 91% cheaper than on-demand
- May be terminated at any time (24-hour max lifetime for preemptible)
- Use for: batch jobs, stateless workers, CI/CD, fault-tolerant workloads
- Avoid for: databases, primary app servers

### Billing Alerts

Always configure budget alerts for any new project:

```bash
# Budget alerts are configured via the Billing console or API
# Recommend: 50%, 90%, 100% of monthly budget as alert thresholds
```

---

## Security Best Practices

### Secret Manager

Store all credentials, API keys, and sensitive config in Secret Manager — never in environment variables or code:

```bash
# Create secret
ALIUNDE_GOOGLE_ADMIN=1 gcloud secrets create my-secret \
  --replication-policy=automatic

# Add secret version
echo -n "my-secret-value" | \
  ALIUNDE_GOOGLE_ADMIN=1 gcloud secrets versions add my-secret --data-file=-

# Access in application (Python example)
# from google.cloud import secretmanager
# client = secretmanager.SecretManagerServiceClient()
# name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
# response = client.access_secret_version(request={"name": name})
# secret_value = response.payload.data.decode("UTF-8")
```

### Binary Authorization

Enforce that only trusted container images run in GKE:

```bash
# Check Binary Authorization policy
gcloud container binauthz policy export
```

### Cloud Armor (WAF / DDoS)

For external HTTPS Load Balancers:
- Apply security policies with preconfigured WAF rules
- Recommended rule sets: OWASP Top 10, RCE, XSS, SQLi
- Enable rate limiting at the edge

---

## Common gcloud Patterns

### Auth and Project Management

```bash
# List active accounts
gcloud auth list

# Set active account
gcloud config set account [email]

# List projects
gcloud projects list

# Switch project
gcloud config set project [project-id]

# Create project (confirm first)
ALIUNDE_GOOGLE_ADMIN=1 gcloud projects create [project-id] \
  --name="[display-name]" \
  --folder=[folder-id]
```

### IAM Grant / Revoke

```bash
# Grant role
ALIUNDE_GOOGLE_ADMIN=1 gcloud projects add-iam-policy-binding [project-id] \
  --member=[principal-type]:[identifier] \
  --role=roles/[role-name]

# Revoke role
ALIUNDE_GOOGLE_ADMIN=1 gcloud projects remove-iam-policy-binding [project-id] \
  --member=[principal-type]:[identifier] \
  --role=roles/[role-name]

# Get current IAM policy
gcloud projects get-iam-policy [project-id] --format=yaml
```

### kubectl Context for GKE

```bash
# Get credentials and set kubectl context
ALIUNDE_GOOGLE_ADMIN=1 gcloud container clusters get-credentials [cluster-name] \
  --region=[region] \
  --project=[project-id]

# Verify context
kubectl config current-context
kubectl cluster-info
```

### Terraform for GCP

```hcl
# Provider setup
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Example: Cloud Run service
resource "google_cloud_run_v2_service" "my_service" {
  name     = "my-service"
  location = var.region

  template {
    containers {
      image = "gcr.io/${var.project_id}/my-app:latest"

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }
    }

    service_account = google_service_account.my_sa.email
  }
}
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Basic roles (Owner/Editor) at project scope | Massively over-privileged | Use predefined or custom roles |
| Public Cloud Storage buckets | Data exposure risk | Enable uniform bucket-level access + public access prevention |
| Hardcoded service account keys in code | Key leakage risk; hard to rotate | Use Workload Identity Federation or Secret Manager |
| No VPC for GCE instances | Exposed to public internet | Always deploy in VPC with private IPs |
| Compute Engine VMs with public IPs | Unnecessary attack surface | Use Cloud NAT + Internal LB pattern |
| No lifecycle rules on Cloud Storage | Storage costs grow unbounded | Apply Nearline/Coldline/Archive transitions |
| Ignoring Cloud SQL public IP warning | Direct internet exposure | Use Cloud SQL Auth Proxy or Private IP only |
| Default network (auto-mode VPC) in production | Overly broad firewall rules, no subnet control | Create custom-mode VPC with explicit subnets |
| No billing alerts | Budget overruns go undetected | Set budget alerts at 50/90/100% |
| Service accounts with project-level Editor | Over-privileged; blast radius too wide | Grant only the specific roles needed per service |

---

## Quick Reference: Essential gcloud Commands

```bash
# Account and project
gcloud config list                                  # Show current config
gcloud auth list                                    # List accounts
gcloud projects list                                # List projects
gcloud config set project [id]                      # Switch project

# IAM
gcloud projects get-iam-policy [project]            # Show IAM bindings
gcloud iam roles list --filter="name:roles/storage" # Find roles
gcloud iam service-accounts list                    # List SAs

# Compute
gcloud compute instances list                       # List VMs
gcloud compute machine-types list --filter="zone:us-central1-a" # List machine types

# GKE
gcloud container clusters list                      # List clusters
gcloud container clusters get-credentials [name] --region=[r]  # Get kubeconfig

# Cloud Run
gcloud run services list                            # List services
gcloud run services describe [name] --region=[r]    # Service details

# Cloud Storage
gcloud storage ls                                   # List buckets
gcloud storage ls gs://[bucket]/                    # List objects

# Cloud SQL
gcloud sql instances list                           # List instances
gcloud sql backups list --instance=[name]           # List backups

# Logs
gcloud logging read "resource.type=gce_instance" --limit=50
gcloud logging read "severity>=ERROR" --limit=20 --freshness=1h
```

---

## References

- [GCP Documentation](https://cloud.google.com/docs)
- [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)
- [GCP IAM Documentation](https://cloud.google.com/iam/docs)
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [GCP Well-Architected Framework](https://cloud.google.com/architecture/framework)
- [Terraform Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [GCP Security Best Practices](https://cloud.google.com/security/best-practices)
