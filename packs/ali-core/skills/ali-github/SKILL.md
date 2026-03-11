---
name: ali-github
description: |
  GitHub platform features including Actions, OIDC, repositories, and CI/CD patterns. Use when:

  PLANNING: Designing CI/CD pipelines, planning GitHub Actions workflows, evaluating
  branch protection strategies, architecting multi-environment deployments

  IMPLEMENTATION: Writing GitHub Actions workflows, configuring OIDC for AWS/cloud,
  setting up repository secrets, implementing branch protection rules

  GUIDANCE: Asking about GitHub Actions best practices, workflow optimization,
  security hardening, how to structure CI/CD pipelines

  REVIEW: Checking workflow configurations, reviewing OIDC trust policies,
  validating repository settings, auditing CI/CD security
---

# GitHub

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing CI/CD pipelines with GitHub Actions
- Planning multi-environment deployment workflows
- Evaluating branch protection strategies
- Architecting repository structure and permissions

**Implementation:**
- Writing GitHub Actions workflow YAML files
- Configuring OIDC for AWS, Azure, or GCP
- Setting up repository secrets and variables
- Implementing reusable workflows and composite actions

**Guidance/Best Practices:**
- Asking about workflow optimization and caching
- Asking how to structure CI/CD pipelines
- Asking about GitHub Actions security best practices
- Asking about runner selection and performance

**Review/Validation:**
- Reviewing workflow configurations for issues
- Checking OIDC trust policies for security
- Validating repository settings and permissions
- Auditing CI/CD for supply chain security

---

## Key Principles

- **OIDC over static credentials**: Never store AWS/cloud credentials as secrets; use OIDC
- **Least privilege**: Workflows and tokens should have minimum required permissions
- **Pin action versions**: Use SHA hashes, not tags (supply chain security)
- **Cache aggressively**: Cache dependencies, build artifacts, and Docker layers
- **Fail fast**: Run lint and unit tests before expensive integration tests
- **Environments for gates**: Use GitHub Environments for deployment approvals
- **Secrets are not env vars**: Use `secrets` context, not environment variables for sensitive data
- **Reusable workflows**: Extract common patterns into reusable workflows

---

## GitHub Actions Workflow Patterns

### Basic CI Workflow Structure

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Limit concurrent runs to save resources
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
      - run: pip install ruff
      - run: ruff check src/ tests/
      - run: ruff format --check src/ tests/

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
      - run: pip install -e ".[dev]"
      - run: pytest tests/ -v --tb=short
```

### OIDC Authentication for AWS

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    # REQUIRED: Must declare permissions for OIDC
    permissions:
      id-token: write   # Required for OIDC token
      contents: read    # Required to checkout code

    steps:
      - uses: actions/checkout@v4

      # Configure AWS credentials via OIDC (no static secrets!)
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
          # Optional: Role session name for CloudTrail
          role-session-name: github-actions-${{ github.run_id }}

      - run: aws sts get-caller-identity  # Verify authentication
```

### AWS IAM Role Trust Policy for OIDC

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:ORG/REPO:*"
        }
      }
    }
  ]
}
```

**Trust Policy Conditions:**

| Condition | Example | Use Case |
|-----------|---------|----------|
| Any branch | `repo:org/repo:*` | Development/staging |
| Main only | `repo:org/repo:ref:refs/heads/main` | Production deploys |
| Tagged releases | `repo:org/repo:ref:refs/tags/v*` | Release deploys |
| Pull requests | `repo:org/repo:pull_request` | PR preview environments |
| Specific env | `repo:org/repo:environment:production` | Environment-gated |

### Docker Build and ECR Push

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2
        id: login-ecr

      # Build with cache for faster builds
      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/my-app:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/my-app:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### ECS Deployment

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment: production  # Requires approval
    permissions:
      id-token: write
      contents: read

    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      # Force new deployment with latest image
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-service \
            --force-new-deployment

      # Wait for deployment to stabilize
      - name: Wait for stability
        run: |
          aws ecs wait services-stable \
            --cluster my-cluster \
            --services my-service
```

### Service Container for Integration Tests

```yaml
jobs:
  test-integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
      - run: pip install -e ".[dev]"
      - run: pytest tests/integration -v
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
```

### Matrix Builds

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false  # Don't cancel other jobs if one fails
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, macos-latest]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -e ".[dev]"
      - run: pytest tests/
```

### Caching Dependencies

```yaml
steps:
  # Python with pip cache
  - uses: actions/setup-python@v5
    with:
      python-version: "3.11"
      cache: pip
      cache-dependency-path: requirements*.txt

  # Node with npm cache
  - uses: actions/setup-node@v4
    with:
      node-version: "20"
      cache: npm

  # Custom cache
  - uses: actions/cache@v4
    with:
      path: ~/.cache/pre-commit
      key: pre-commit-${{ hashFiles('.pre-commit-config.yaml') }}
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      cluster:
        required: true
        type: string
    secrets:
      AWS_ROLE_ARN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - run: |
          aws ecs update-service \
            --cluster ${{ inputs.cluster }} \
            --service my-service \
            --force-new-deployment
```

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      cluster: prod-cluster
    secrets:
      AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

---

## GitHub Environments

### Environment Configuration

Environments provide:
- **Required reviewers**: Approval gates before deployment
- **Wait timer**: Delay before deployment proceeds
- **Branch restrictions**: Only certain branches can deploy
- **Secrets**: Environment-specific secrets

```yaml
jobs:
  deploy-staging:
    environment: staging
    # No approval needed

  deploy-production:
    environment: production
    # Requires approval from designated reviewers
    needs: deploy-staging
```

### Environment Secrets vs Repository Secrets

| Type | Scope | Use Case |
|------|-------|----------|
| Repository secrets | All workflows | Shared config (AWS account ID) |
| Environment secrets | Specific environment | Environment-specific (prod DB password) |

```yaml
# Access environment-specific secret
- run: echo "Deploying to ${{ vars.ENVIRONMENT_NAME }}"
  env:
    DB_PASSWORD: ${{ secrets.DB_PASSWORD }}  # From environment
```

---

## Branch Protection Rules

### Recommended Settings for Main Branch

| Setting | Value | Why |
|---------|-------|-----|
| Require pull request | Yes | No direct pushes |
| Required reviews | 1+ | Code review gate |
| Dismiss stale reviews | Yes | Re-review after changes |
| Require status checks | Yes | CI must pass |
| Require branches up to date | Yes | Merge conflicts caught early |
| Require signed commits | Optional | For high-security repos |
| Include administrators | Yes | No exceptions |

### Required Status Checks

Select which workflow jobs must pass:
- `lint` - Code formatting and style
- `test` - Unit tests
- `security` - Security scanning (Dependabot, CodeQL)

---

## Security Best Practices

### Pin Action Versions by SHA

```yaml
# BAD - Tag can be moved to malicious code
- uses: actions/checkout@v4

# GOOD - SHA is immutable
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# ACCEPTABLE - Major version for trusted actions
- uses: actions/checkout@v4  # OK for official actions
```

### Limit Workflow Permissions

```yaml
# Global permission restriction
permissions:
  contents: read  # Default: read-only

jobs:
  deploy:
    permissions:
      id-token: write  # Only what's needed
      contents: read
```

### Secrets Handling

```yaml
# NEVER echo secrets
- run: echo ${{ secrets.API_KEY }}  # BAD - Visible in logs

# Use masking for derived values
- run: |
    TOKEN=$(get-token)
    echo "::add-mask::$TOKEN"
    use-token "$TOKEN"

# Pass secrets via environment
- run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

### Dependabot Security Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: pip
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 10

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Static AWS credentials | Can be leaked, no rotation | Use OIDC authentication |
| `permissions: write-all` | Excessive access | Declare minimal permissions |
| Unpinned action versions | Supply chain risk | Pin to SHA or major version |
| Secrets in workflow files | Exposed in repo | Use GitHub Secrets |
| No concurrency limits | Wasted resources | Use `concurrency` key |
| No caching | Slow builds | Cache dependencies and layers |
| No `fail-fast: false` | Hides failures | See all matrix failures |
| Hard-coded values | Not reusable | Use inputs and environment |
| Missing health checks | Stale service containers | Add health check options |
| No deployment validation | Deploy failures undetected | Add smoke tests after deploy |

---

## Quick Reference

### Workflow Triggers

```yaml
on:
  push:
    branches: [main, develop]
    paths: ['src/**', 'tests/**']  # Only on certain file changes
    paths-ignore: ['docs/**']

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM UTC

  workflow_dispatch:  # Manual trigger
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

  release:
    types: [published]
```

### Common Expressions

```yaml
# Conditionals
if: github.ref == 'refs/heads/main'
if: github.event_name == 'pull_request'
if: contains(github.event.head_commit.message, '[skip ci]')
if: always()  # Run even if previous steps failed
if: failure()  # Run only if previous steps failed

# String operations
${{ github.repository }}           # owner/repo
${{ github.sha }}                  # Full commit SHA
${{ github.ref_name }}             # Branch or tag name
${{ github.run_id }}               # Unique run identifier
${{ github.actor }}                # User who triggered

# Outputs between steps
- id: build
  run: echo "version=1.0.0" >> $GITHUB_OUTPUT
- run: echo "Version is ${{ steps.build.outputs.version }}"
```

### GitHub CLI in Workflows

```yaml
- name: Create issue on failure
  if: failure()
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    gh issue create \
      --title "CI Failed: ${{ github.workflow }}" \
      --body "Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}" \
      --label "bug,ci-failure"

- name: Comment on PR
  if: github.event_name == 'pull_request'
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    gh pr comment ${{ github.event.pull_request.number }} \
      --body "Build succeeded! Preview: https://preview.example.com/${{ github.sha }}"
```

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| CI/CD workflows | `.github/workflows/` |
| Dependabot config | `.github/dependabot.yml` |
| Branch protection | Repository Settings > Branches |
| Environments | Repository Settings > Environments |
| Secrets | Repository Settings > Secrets and variables |

---

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub OIDC with AWS](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- [Workflow Syntax Reference](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions)
- [Security Hardening Guide](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)
