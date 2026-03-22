---
name: ali-bicep
description: |
  Azure Bicep and ARM template Infrastructure as Code patterns. Use when:

  PLANNING: Designing Bicep module structure, planning parameter file strategy,
  architecting resource dependencies, evaluating Bicep vs Terraform for Azure,
  planning what-if deployment workflows

  IMPLEMENTATION: Writing Bicep resource declarations, creating modules,
  configuring parameter files, implementing deployment scripts, writing
  Azure Verified Modules patterns, building CI/CD pipelines for Bicep

  GUIDANCE: Asking about Bicep syntax, best practices for ARM templates,
  how to structure Bicep projects, naming conventions, resource type reference

  REVIEW: Auditing Bicep files for security misconfigurations, reviewing
  module design, validating parameter constraints, checking resource naming,
  reviewing deployment pipelines for Bicep
---

# Azure Bicep / ARM Templates

## Key Principles

- Bicep over ARM JSON (Bicep is the authoring layer — always compile to ARM, never write raw ARM JSON for new work)
- Modules for reuse (one resource type or logical group per module)
- Parameter files per environment (main.bicepparam for Bicep, .parameters.json for ARM compatibility)
- what-if before every deployment (treat like terraform plan)
- Azure Verified Modules (AVM) as the baseline for common resources

---

## Bicep Fundamentals

### Resource Declaration

```bicep
// Basic resource declaration
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stmyappprodeus001'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

### Parameters

```bicep
@description('Deployment environment')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Application name for resource naming')
@minLength(2)
@maxLength(16)
param appName string

@description('Database administrator password')
@secure()
param adminPassword string

// Optional with default
param location string = resourceGroup().location
```

### Variables, Outputs, Existing Resources

```bicep
var resourcePrefix = '${appName}-${environment}'

// Reference existing resource (not managed by this template)
resource existingVnet 'Microsoft.Network/virtualNetworks@2023-04-01' existing = {
  name: 'vnet-hub-prod-eastus-001'
  scope: resourceGroup('rg-networking-prod')
}

output storageAccountId string = storageAccount.id
output storageEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### String Interpolation, Conditionals, Loops

```bicep
// String interpolation — always use this, never concatenation
var storageName = 'st${appName}${environment}eus001'

// Conditional deployment
resource diagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (environment == 'prod') {
  name: 'diag-${appName}'
  // ...
}

// Loop over array
var subnetNames = ['frontend', 'backend', 'data']
resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-04-01' = [for name in subnetNames: {
  name: name
  parent: vnet
  properties: {
    addressPrefix: '10.0.${indexOf(subnetNames, name)}.0/24'
  }
}]
```

---

## Module Structure

### Flat Module (Local File)

```bicep
// main.bicep
module keyVault './modules/key-vault.bicep' = {
  name: 'keyVaultDeploy'
  params: {
    name: 'kv-${appName}-${environment}-eus-001'
    location: location
    tenantId: tenant().tenantId
  }
}

// Access module output
output kvUri string = keyVault.outputs.vaultUri
```

### Azure Verified Modules (Registry)

```bicep
// Use AVM for common resources — preferred over hand-rolled modules
module storageAccount 'br/public:avm/res/storage/storage-account:0.9.0' = {
  name: 'storageAccountDeploy'
  params: {
    name: 'stmyappprodeus001'
    location: location
    skuName: 'Standard_LRS'
  }
}
```

### Module Rules

- Explicit parameter passing — no implicit cross-module dependencies
- Outputs flow up — never have modules call each other sideways
- One resource type or logical group per module

---

## Parameter Files

### .bicepparam (Preferred — Native Bicep)

```bicep
// main.dev.bicepparam
using './main.bicep'

param environment = 'dev'
param appName = 'myapp'
param location = 'eastus'
```

### .parameters.json (ARM Compatibility)

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": { "value": "dev" },
    "appName": { "value": "myapp" }
  }
}
```

### Key Vault Secret Reference (Parameter Files)

```json
{
  "parameters": {
    "adminPassword": {
      "reference": {
        "keyVault": {
          "id": "/subscriptions/{sub}/resourceGroups/rg-secrets/providers/Microsoft.KeyVault/vaults/kv-myapp-prod"
        },
        "secretName": "admin-password"
      }
    }
  }
}
```

**Rule:** One params file per environment. Never hardcode environment-specific values in .bicep files.

---

## Deployment Patterns

### Scope Declarations

```bicep
// Resource group scope (default — omit or declare explicitly)
targetScope = 'resourceGroup'

// Subscription scope (for resource group creation, policy assignments)
targetScope = 'subscription'

// Management group scope (for org-wide policies)
targetScope = 'managementGroup'
```

### what-if and Deploy Commands

```bash
# what-if (required before every prod deployment)
az deployment group what-if \
  --resource-group rg-myapp-prod-eastus-001 \
  --template-file main.bicep \
  --parameters main.prod.bicepparam

# Deploy
az deployment group create \
  --resource-group rg-myapp-prod-eastus-001 \
  --template-file main.bicep \
  --parameters main.prod.bicepparam \
  --name "deploy-$(date +%Y%m%d%H%M%S)"

# Subscription-scope deploy (for resource group creation)
az deployment sub create \
  --location eastus \
  --template-file main.bicep \
  --parameters main.prod.bicepparam
```

### Deployment Stacks (Preview)

```bash
# Track and manage all resources as a unit
az stack group create \
  --name myapp-prod-stack \
  --resource-group rg-myapp-prod-eastus-001 \
  --template-file main.bicep \
  --parameters main.prod.bicepparam \
  --deny-settings-mode none
```

---

## Naming Conventions (Microsoft CAF)

**Format:** `{resource-abbreviation}-{workload}-{env}-{region}-{instance}`

| Resource | Abbreviation | Example |
|----------|-------------|---------|
| Resource Group | rg | rg-myapp-prod-eastus-001 |
| App Service | app | app-myapp-prod-eus-001 |
| Key Vault | kv | kv-myapp-prod-eus-001 |
| Storage Account | st | stmyappprodeus001 (no dashes — 24 char max) |
| AKS Cluster | aks | aks-myapp-prod-eus-001 |
| Virtual Network | vnet | vnet-myapp-prod-eastus-001 |
| Container Registry | cr | crmyappprodeus001 (no dashes) |
| Function App | func | func-myapp-prod-eus-001 |
| Service Bus | sb | sb-myapp-prod-eus-001 |

Full abbreviation list: https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations

---

## Security Patterns

### Secrets in Parameters

```bicep
// Always @secure() on any parameter containing passwords, keys, or connection strings
@secure()
param adminPassword string

@secure()
param connectionString string
```

### Role Assignments

```bicep
// Role assignment — use newGuid() for deterministic names, scope at resource level
resource storageRoleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(storageAccount.id, managedIdentity.id, roleDefinitionId)
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', roleDefinitionId)
    principalId: managedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### Managed Identity Over Service Principal

```bicep
// Assign system-assigned managed identity to App Service
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-${appName}-${environment}-eus-001'
  location: location
  identity: {
    type: 'SystemAssigned'  // prefer over service principal credentials
  }
  properties: {
    // ...
  }
}
```

---

## CI/CD Integration

### GitHub Actions Pattern (OIDC — No Client Secrets)

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC — Workload Identity Federation)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Bicep lint
        run: az bicep build --file main.bicep

      - name: what-if
        run: |
          az deployment group what-if \
            --resource-group ${{ vars.RESOURCE_GROUP }} \
            --template-file main.bicep \
            --parameters main.${{ vars.ENVIRONMENT }}.bicepparam

      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          az deployment group create \
            --resource-group ${{ vars.RESOURCE_GROUP }} \
            --template-file main.bicep \
            --parameters main.${{ vars.ENVIRONMENT }}.bicepparam
```

### Bicep Linter (bicepconfig.json)

```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-hardcoded-env-urls": { "level": "error" },
        "no-unnecessary-dependson": { "level": "warning" },
        "secure-parameter-default": { "level": "error" },
        "use-stable-vm-image": { "level": "warning" }
      }
    }
  }
}
```

**Pipeline pattern:** lint → what-if → manual approval → deploy

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Hardcoding resource IDs | Breaks across environments | Use `existing` keyword or outputs from other modules |
| Giant monolithic main.bicep | Hard to maintain, review, reuse | Split into focused modules |
| `@secure()` missing on secret parameters | Secrets exposed in deployment history | Add `@secure()` to every password/key/connection string |
| Skipping what-if in prod pipelines | Unexpected destructive changes | Always run what-if before prod deploy |
| Using `any()` type | Bypasses type validation | Use correct resource type or create a typed object |
| ARM JSON for new work | Verbose, error-prone | Use Bicep, compile to ARM with `az bicep build` |
| targetScope mismatch | Deployment fails at submission | Match targetScope to the az deployment command type |
| Modules calling each other sideways | Circular dependencies, unpredictable ordering | Outputs flow up to parent only |
| Hardcoded environment values in .bicep | Not reusable across envs | All env-specific values in .bicepparam files |
| Client secrets in app registrations | Rotation burden, leak risk | Use Managed Identity where workload is Azure-hosted |

---

## Quick Reference

### Bicep CLI Commands

```bash
# Install / upgrade
az bicep install
az bicep upgrade
az bicep version

# Compile Bicep to ARM JSON
az bicep build --file main.bicep

# Decompile ARM JSON to Bicep (starting point — always review output)
az bicep decompile --file azuredeploy.json

# Lint only (no compilation output)
az bicep build --file main.bicep --stdout > /dev/null
```

### Common Resource Type Strings

| Resource | Type String |
|----------|-------------|
| Virtual Machine | `Microsoft.Compute/virtualMachines@2023-09-01` |
| App Service | `Microsoft.Web/sites@2023-01-01` |
| AKS | `Microsoft.ContainerService/managedClusters@2024-01-01` |
| Key Vault | `Microsoft.KeyVault/vaults@2023-07-01` |
| Storage Account | `Microsoft.Storage/storageAccounts@2023-01-01` |
| Virtual Network | `Microsoft.Network/virtualNetworks@2023-04-01` |
| Service Bus Namespace | `Microsoft.ServiceBus/namespaces@2022-10-01-preview` |
| Container Registry | `Microsoft.ContainerRegistry/registries@2023-07-01` |
| Role Assignment | `Microsoft.Authorization/roleAssignments@2022-04-01` |

### VS Code Setup

Install extension: `ms-azuretools.vscode-bicep` — provides intellisense, linting, and what-if previews inline.

---

## References

- [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [CAF Naming Conventions](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/resource-abbreviations)
- [Bicep GitHub](https://github.com/Azure/bicep)
- [Workload Identity Federation](https://learn.microsoft.com/en-us/azure/active-directory/workload-identities/workload-identity-federation)
