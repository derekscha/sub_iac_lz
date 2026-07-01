# Context: Azure Terraform CI/CD Bootstrap – Bicep (Key Vault + Storage Account)

## Research Finding

Microsoft does not publish a dedicated Azure Quickstart Template or AVM pattern module
for the Terraform CI/CD bootstrap use case (state storage + secrets vault combined).
The AVM Quickstart guide defers remote state configuration to Microsoft Learn docs.

The correct approach is to compose two AVM Bicep resource modules:

- `br/public:avm/res/storage/storage-account:0.13.0` — state backend
- `br/public:avm/res/key-vault/vault:0.12.0` — secrets storage

Verify current versions before deploying:

- <https://azure.github.io/Azure-Verified-Modules/indexes/bicep/bicep-resource-modules/>

## Design Decisions

- `allowSharedKeyAccess = false` + `use_azuread_auth = true` in Terraform backend (Entra ID auth, no shared keys)
- `enableRbacAuthorization = true` on Key Vault (RBAC only, access policies disabled)
- CI/CD identity: `Storage Blob Data Contributor` + `Key Vault Secrets User` (least privilege)
- Admin identity: `Storage Blob Data Owner` + `Key Vault Administrator`
- `CanNotDelete` locks on both resources
- Storage replication: `Standard_GRS`
- Soft-delete + blob versioning retention: 30 days on both resources
- `uniqueSuffix` param (max 6 chars) handles global naming uniqueness — pass as pipeline variable

## Bicep Template

File: `terraform-bootstrap/main.bicep`

```bicep
targetScope = 'resourceGroup'

@description('Azure region for all resources.')
param location string = resourceGroup().location

@description('Environment token used for resource naming.')
@allowed(['dev', 'tst', 'prd'])
param environment string = 'dev'

@description('Workload token used for resource naming.')
param workload string = 'tfstate'

@description('Unique suffix for globally unique resource names (storage account, Key Vault).')
@maxLength(6)
param uniqueSuffix string

@description('Object ID of the CI/CD service principal or managed identity that will run Terraform.')
param cicdPrincipalObjectId string

@description('Object ID of the operator or AAD group that administers the vault.')
param adminObjectId string

@description('Current tenant ID.')
param tenantId string = tenant().tenantId

module stateStorage 'br/public:avm/res/storage/storage-account:0.13.0' = {
  name: 'deploy-tf-state-storage'
  params: {
    name: 'st${workload}${environment}${uniqueSuffix}'
    location: location
    skuName: 'Standard_GRS'
    kind: 'StorageV2'
    accessTier: 'Hot'
    allowBlobPublicAccess: false
    allowSharedKeyAccess: false
    minimumTlsVersion: 'TLS1_2'
    lock: {
      kind: 'CanNotDelete'
      name: 'tf-state-storage-lock'
    }
    blobServices: {
      containers: [
        {
          name: 'tfstate'
          publicAccess: 'None'
        }
      ]
      deleteRetentionPolicy: {
        enabled: true
        days: 30
      }
      containerDeleteRetentionPolicy: {
        enabled: true
        days: 30
      }
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: 'Storage Blob Data Contributor'
        principalId: cicdPrincipalObjectId
        principalType: 'ServicePrincipal'
      }
      {
        roleDefinitionIdOrName: 'Storage Blob Data Owner'
        principalId: adminObjectId
        principalType: 'User'
      }
    ]
    tags: {
      workload: workload
      environment: environment
      purpose: 'terraform-remote-state'
    }
  }
}

module secretsVault 'br/public:avm/res/key-vault/vault:0.12.0' = {
  name: 'deploy-tf-secrets-vault'
  params: {
    name: 'kv-${workload}-${environment}-${uniqueSuffix}'
    location: location
    enablePurgeProtection: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 30
    enableRbacAuthorization: true
    sku: 'standard'
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Allow'
    }
    lock: {
      kind: 'CanNotDelete'
      name: 'tf-secrets-vault-lock'
    }
    roleAssignments: [
      {
        roleDefinitionIdOrName: 'Key Vault Secrets User'
        principalId: cicdPrincipalObjectId
        principalType: 'ServicePrincipal'
      }
      {
        roleDefinitionIdOrName: 'Key Vault Administrator'
        principalId: adminObjectId
        principalType: 'User'
      }
    ]
    tags: {
      workload: workload
      environment: environment
      purpose: 'terraform-ci-secrets'
    }
  }
}

output storageAccountName string = stateStorage.outputs.name
output storageAccountId string = stateStorage.outputs.resourceId
output tfstateContainerName string = 'tfstate'
output keyVaultName string = secretsVault.outputs.name
output keyVaultUri string = secretsVault.outputs.uri
```

## Terraform Backend Block

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "<bootstrap-rg-name>"
    storage_account_name = "<output: storageAccountName>"
    container_name       = "tfstate"
    key                  = "workload/env/terraform.tfstate"
    use_azuread_auth     = true
  }
}
```

## Open Items / Next Steps

- Harden `networkAcls.defaultAction` to `Deny` + add `ipRules` for pipeline agent egress IPs in production
- Optionally add private endpoints for both resources if deploying into a private network
- Optionally add a GitHub Actions or Azure DevOps pipeline YAML as a one-time bootstrap step
- Pin AVM module versions after verifying latest stable at the AVM module index
