# Universal Azure Bicep Infrastructure Patterns

These patterns apply to ALL project types. Use them when generating `infra/main.bicep` and the module files in `infra/modules/`.

---

## Design Principles

1. **Subscription-scoped**: `targetScope = 'subscription'` to create the resource group
2. **Modular**: All resources in `infra/modules/`, called from `main.bicep`
3. **Tagged**: Every resource gets `tags` with `azd-env-name` and the solution identifier
4. **Unique names**: Use `uniqueString()` for resource name tokens to avoid collisions
5. **Secure outputs**: All secrets output as `@secure()` — never output plain text secrets
6. **Consistent API versions**: Use latest stable API versions (2024-XX-XX or 2025-XX-XX)

---

## `infra/main.bicep` — Template

```bicep
targetScope = 'subscription'

@minLength(1)
@maxLength(64)
param environmentName string          // from azd

@minLength(1)
param location string                 // from azd, constrained by model availability

// --- AI Model Parameters (include only if project uses Azure OpenAI) ---
param modelDeploymentName string = '<default-model>'   // e.g., gpt-4o, gpt-5.4
param modelVersion string = '<version>'
param deploymentSkuName string = 'GlobalStandard'

// --- Additional parameters based on U7 answers ---
// Add params for each optional Azure service the user selected

var abbrs = loadJsonContent('./abbreviations.json')
var resourceToken = uniqueString(subscription().id, environmentName, location)
var tags = {
  'azd-env-name': environmentName
  'solution': '<project-slug>'
}

// Resource Group
resource rg 'Microsoft.Resources/resourceGroups@2024-03-01' = {
  name: '${abbrs.resourcesResourceGroups}${environmentName}'
  location: location
  tags: tags
}

// --- Module calls in dependency order ---

module monitoring 'modules/monitoring.bicep' = {
  name: 'monitoring'
  scope: rg
  params: {
    location: location
    tags: tags
    logAnalyticsName: '${abbrs.operationalInsightsWorkspaces}${resourceToken}'
    appInsightsName: '${abbrs.insightsComponents}${resourceToken}'
  }
}

// Add remaining modules based on project type and U7 answers
// Follow the dependency order specified below
```

---

## Module Dependency Order

Strictly enforce this order. Modules lower in the list may depend on outputs from modules above.

```
monitoring
    → ai-foundry (if U11=Yes OR Multi-Agent: any project with Foundry agent integration)
    → container-registry (if containerized)
        → container-apps-env
            → container-app (×N, one per service + one per Foundry agent if U11=Yes)
                → role-assignments
```

For optional services, insert them at the appropriate level:
- `cosmos.bicep` — after monitoring, before container-apps
- `storage.bicep` — after monitoring, before container-apps
- `keyvault.bicep` — after monitoring, before container-apps
- `ai-search.bicep` — after ai-foundry, before container-apps
- `servicebus.bicep` — after monitoring, before container-apps

---

## Base Modules (All Project Types)

### `infra/modules/monitoring.bicep`

```bicep
param location string
param tags object
param logAnalyticsName string
param appInsightsName string

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2023-09-01' = {
  name: logAnalyticsName
  location: location
  tags: tags
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  tags: tags
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: logAnalytics.id
  }
}

output logAnalyticsWorkspaceId string = logAnalytics.id
output appInsightsConnectionString string = appInsights.properties.ConnectionString
output appInsightsInstrumentationKey string = appInsights.properties.InstrumentationKey
```

### `infra/modules/container-registry.bicep`

```bicep
param location string
param tags object
param registryName string

resource acr 'Microsoft.ContainerRegistry/registries@2023-11-01-preview' = {
  name: registryName
  location: location
  tags: tags
  sku: { name: 'Standard' }
  properties: {
    adminUserEnabled: true
  }
}

output registryName string = acr.name
output registryLoginServer string = acr.properties.loginServer
```

### `infra/modules/container-apps-env.bicep`

```bicep
param location string
param tags object
param envName string
param logAnalyticsWorkspaceId string

resource env 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: envName
  location: location
  tags: tags
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: reference(logAnalyticsWorkspaceId, '2023-09-01').customerId
        sharedKey: listKeys(logAnalyticsWorkspaceId, '2023-09-01').primarySharedKey
      }
    }
  }
}

output envId string = env.id
output envName string = env.name
```

### `infra/modules/container-app.bicep` (Reusable Per-Service)

```bicep
param location string
param tags object
param appName string
param envId string
param registryLoginServer string
param registryName string
param imageName string
param imageTag string
param targetPort int = 8000
param cpu string = '1'
param memory string = '2Gi'
param minReplicas int = 0
param maxReplicas int = 3
param env array = []

resource app 'Microsoft.App/containerApps@2024-03-01' = {
  name: appName
  location: location
  tags: tags
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    managedEnvironmentId: envId
    configuration: {
      ingress: {
        external: true
        targetPort: targetPort
        transport: 'auto'
      }
      registries: [
        {
          server: registryLoginServer
          identity: 'system'
        }
      ]
    }
    template: {
      containers: [
        {
          name: appName
          image: '${registryLoginServer}/${imageName}:${imageTag}'
          resources: {
            cpu: json(cpu)
            memory: memory
          }
          env: env
        }
      ]
      scale: {
        minReplicas: minReplicas
        maxReplicas: maxReplicas
        rules: [
          {
            name: 'http-rule'
            http: { metadata: { concurrentRequests: '50' } }
          }
        ]
      }
    }
  }
}

output appId string = app.id
output appFqdn string = app.properties.configuration.ingress.fqdn
output principalId string = app.identity.principalId
```

### `infra/modules/role-assignments.bicep`

```bicep
param principalIds array        // managed identity principal IDs from container apps
param scopeResourceId string    // the resource to assign roles on (e.g., Foundry account)
param roleDefinitionIds array   // list of role definition GUIDs

resource roleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for combo in flatten([for pid in principalIds: [for rid in roleDefinitionIds: { pid: pid, rid: rid }]]): {
  name: guid(scopeResourceId, combo.pid, combo.rid)
  scope: scopeResourceId
  properties: {
    principalId: combo.pid
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', combo.rid)
    principalType: 'ServicePrincipal'
  }
}]
```

---

## AI-Specific Modules (RAG, Multi-Agent, any AI project)

### `infra/modules/ai-foundry.bicep`

```bicep
param location string
param tags object
param accountName string
param projectName string
param deploymentName string
param modelName string = 'gpt-4o'
param modelVersion string = '2024-08-06'
param deploymentSkuName string = 'GlobalStandard'
param capacityK int = 100

// Foundry account: kind='AIServices', sku='S0'
resource foundryAccount 'Microsoft.CognitiveServices/accounts@2025-06-01' = {
  name: accountName
  location: location
  tags: tags
  kind: 'AIServices'
  sku: { name: 'S0' }
  properties: {
    customSubDomainName: accountName   // must be globally unique
    publicNetworkAccess: 'Enabled'
  }
}

// Foundry project is a child resource of the account
resource foundryProject 'Microsoft.CognitiveServices/accounts/projects@2025-06-01' = {
  parent: foundryAccount
  name: projectName
  location: location
  tags: tags
}

// Model deployment lives under the ACCOUNT (not the project)
resource modelDeployment 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundryAccount
  name: deploymentName
  sku: {
    name: deploymentSkuName
    capacity: capacityK
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: modelName
      version: modelVersion
    }
  }
}

// CRITICAL: Project endpoint must use this exact format
output projectEndpoint string = 'https://${foundryAccount.properties.customSubDomainName}.services.ai.azure.com/api/projects/${foundryProject.name}'
output accountId string = foundryAccount.id
output accountName string = foundryAccount.name
```

### Multiple Model Deployments (RAG + Foundry)

For projects that need both a chat/reasoning model AND an embedding model (e.g., RAG with Foundry), add a second deployment resource:

```bicep
// Embedding model deployment (in addition to chat model above)
param embeddingDeploymentName string = 'text-embedding-3-large'
param embeddingModelName string = 'text-embedding-3-large'
param embeddingModelVersion string = '1'
param embeddingCapacity int = 120

resource embeddingDeployment 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundryAccount
  name: embeddingDeploymentName
  sku: {
    name: 'GlobalStandard'
    capacity: embeddingCapacity
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: embeddingModelName
      version: embeddingModelVersion
    }
  }
  dependsOn: [modelDeployment]  // Deployments must be sequential
}
```

### RBAC Note for Foundry Agent Projects (U11 = Yes)

When U11 = Yes, all container app managed identities (backend + any Foundry agent containers) must have **both** roles on the Foundry account scope:
- `Cognitive Services OpenAI User` (`5e0bd9bd-7b93-4f28-af87-19fc36ad61bd`)
- `Azure AI User` (`53ca9b11-8b9d-4b51-acae-26b3df39f6f0`)

Pass both role GUIDs to `role-assignments.bicep` and include principal IDs from all container apps that need Foundry access.

### `infra/modules/ai-search.bicep` (for RAG projects)

```bicep
param location string
param tags object
param searchServiceName string
param sku string = 'basic'

resource searchService 'Microsoft.Search/searchServices@2024-06-01-preview' = {
  name: searchServiceName
  location: location
  tags: tags
  sku: { name: sku }
  properties: {
    replicaCount: 1
    partitionCount: 1
    hostingMode: 'default'
    publicNetworkAccess: 'Enabled'
  }
}

output searchServiceId string = searchService.id
output searchServiceName string = searchService.name
output searchServiceEndpoint string = 'https://${searchService.name}.search.windows.net'
```

---

## Optional Service Modules (based on U7 answers)

### `infra/modules/cosmos.bicep`

```bicep
param location string
param tags object
param accountName string
param databaseName string
param containerNames array = []
param useServerless bool = true

resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: accountName
  location: location
  tags: tags
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [{ locationName: location, failoverPriority: 0 }]
    capabilities: useServerless ? [{ name: 'EnableServerless' }] : []
    consistencyPolicy: { defaultConsistencyLevel: 'Session' }
  }
}

resource database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2024-05-15' = {
  parent: cosmosAccount
  name: databaseName
  properties: {
    resource: { id: databaseName }
  }
}

resource containers 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2024-05-15' = [for name in containerNames: {
  parent: database
  name: name
  properties: {
    resource: {
      id: name
      partitionKey: { paths: ['/id'], kind: 'Hash' }
    }
  }
}]

output cosmosAccountEndpoint string = cosmosAccount.properties.documentEndpoint
output cosmosAccountName string = cosmosAccount.name
```

### `infra/modules/storage.bicep`

```bicep
param location string
param tags object
param storageName string
param containerNames array = []

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  tags: tags
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
  }
}

resource blobService 'Microsoft.Storage/storageAccounts/blobServices@2023-05-01' = {
  parent: storageAccount
  name: 'default'
}

resource blobContainers 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-05-01' = [for name in containerNames: {
  parent: blobService
  name: name
}]

output storageAccountName string = storageAccount.name
output storageAccountEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### `infra/modules/keyvault.bicep`

```bicep
param location string
param tags object
param vaultName string

resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: vaultName
  location: location
  tags: tags
  properties: {
    sku: { family: 'A', name: 'standard' }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enabledForDeployment: false
    enabledForDiskEncryption: false
    enabledForTemplateDeployment: false
  }
}

output vaultUri string = keyVault.properties.vaultUri
output vaultName string = keyVault.name
```

### `infra/modules/servicebus.bicep`

```bicep
param location string
param tags object
param namespaceName string
param queueNames array = []
param topicNames array = []

resource sbNamespace 'Microsoft.ServiceBus/namespaces@2023-01-01-preview' = {
  name: namespaceName
  location: location
  tags: tags
  sku: { name: 'Standard', tier: 'Standard' }
}

resource queues 'Microsoft.ServiceBus/namespaces/queues@2023-01-01-preview' = [for name in queueNames: {
  parent: sbNamespace
  name: name
  properties: {
    maxDeliveryCount: 10
    deadLetteringOnMessageExpiration: true
  }
}]

resource topics 'Microsoft.ServiceBus/namespaces/topics@2023-01-01-preview' = [for name in topicNames: {
  parent: sbNamespace
  name: name
}]

output namespaceName string = sbNamespace.name
output namespaceEndpoint string = '${sbNamespace.name}.servicebus.windows.net'
```

---

## `infra/main.parameters.json` — Template

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environmentName":        { "value": "${AZURE_ENV_NAME}" },
    "location":               { "value": "${AZURE_LOCATION}" },
    "modelDeploymentName":    { "value": "${AZURE_OPENAI_DEPLOYMENT_NAME}" },
    "deploymentSkuName":      { "value": "${AZURE_OPENAI_DEPLOYMENT_SKU}" }
  }
}
```

Adjust parameters based on the project type and services selected. Only include AI-related parameters if the project uses Azure OpenAI.

---

## `infra/abbreviations.json` — Standard Prefixes

```json
{
  "resourcesResourceGroups": "rg-",
  "operationalInsightsWorkspaces": "log-",
  "insightsComponents": "appi-",
  "containerRegistryRegistries": "cr",
  "appManagedEnvironments": "cae-",
  "appContainerApps": "ca-",
  "cognitiveServicesAccounts": "ai-",
  "searchSearchServices": "srch-",
  "documentDBDatabaseAccounts": "cosmos-",
  "storageStorageAccounts": "st",
  "keyVaultVaults": "kv-",
  "serviceBusNamespaces": "sb-",
  "webSitesFunctions": "func-",
  "machineLearningServicesWorkspaces": "mlw-"
}
```

---

## RBAC Role Definition IDs — Common Roles

| Role | GUID | When Needed |
|---|---|---|
| Cognitive Services OpenAI User | `5e0bd9bd-7b93-4f28-af87-19fc36ad61bd` | Any project using Azure OpenAI |
| Azure AI User | `53ca9b11-8b9d-4b51-acae-26b3df39f6f0` | Any project using Azure AI Foundry |
| Storage Blob Data Contributor | `ba92f5b4-2d11-453d-a403-e96b0029c9fe` | Projects using Azure Storage |
| Cosmos DB Data Contributor | `00000000-0000-0000-0000-000000000002` | Projects using Cosmos DB |
| Key Vault Secrets User | `4633458b-17de-408a-b874-0445c86b69e6` | Projects using Key Vault |
| Service Bus Data Sender/Receiver | `69a216fc-b8fb-44d8-bc22-1f3c2cd27a39` / `4f6d3b9b-027b-4f4c-9142-0e5a2a2247e0` | Projects using Service Bus |

Assign roles based on which services the project uses. Every container app managed identity must have the appropriate roles on the services it accesses.
