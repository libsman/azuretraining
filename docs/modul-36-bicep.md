# Modul 36 – Bicep: Infrastruktur als Code – modern und lesbar

## Lernziele

Nach diesem Modul kannst du:

- Bicep als modernere Alternative zu ARM Templates beschreiben
- Die Bicep-Syntax für Ressourcen, Parameter, Variablen und Outputs schreiben
- Eine vollständige Infrastruktur (App Service + Storage + Key Vault) per Bicep deployen
- Module in Bicep für Wiederverwendbarkeit einsetzen
- Bicep und ARM gegenseitig konvertieren

---

## Hintergrund: Warum Bicep statt ARM?

**Direkter Vergleich** – dasselbe Storage Account in beiden Sprachen:

**ARM (JSON, 25 Zeilen):**
```json
{
  "$schema": "...",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": { "type": "string" }
  },
  "resources": [{
    "type": "Microsoft.Storage/storageAccounts",
    "apiVersion": "2023-01-01",
    "name": "[parameters('storageAccountName')]",
    "location": "[resourceGroup().location]",
    "sku": { "name": "Standard_LRS" },
    "kind": "StorageV2",
    "properties": {}
  }]
}
```

**Bicep (7 Zeilen):**
```bicep
param storageAccountName string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: resourceGroup().location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
```

Bicep kompiliert intern zu ARM – du schreibst weniger, Azure versteht dasselbe.

!!! info "Bicep ist Microsofts empfohlener IaC-Weg"
    Microsoft empfiehlt Bicep für alle neuen Azure-IaC-Projekte. ARM ist weiterhin gültig aber neu anfangen solltest du mit Bicep.

---

## Bicep installieren

Bicep ist in der Azure CLI integriert:

```bash
# Aktuelle Bicep-Version prüfen
az bicep version

# Bicep aktualisieren
az bicep upgrade
```

In der Cloud Shell ist Bicep bereits verfügbar.

---

## Erste Bicep-Datei

Erstelle `storage.bicep` in der Cloud Shell (`code storage.bicep`):

```bicep
@description('Name des Storage Accounts')
@minLength(3)
@maxLength(24)
param storageAccountName string

@description('Azure-Region')
param location string = resourceGroup().location

@allowed(['Standard_LRS', 'Standard_GRS', 'Premium_LRS'])
param sku string = 'Standard_LRS'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: sku
  }
  kind: 'StorageV2'
  properties: {
    accessTier: 'Hot'
    allowBlobPublicAccess: false
  }
}

output storageAccountId string = storageAccount.id
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### Deployen

```bash
az group create --name rg-devops --location westeurope

az deployment group create \
  --resource-group rg-devops \
  --template-file storage.bicep \
  --parameters storageAccountName=stbicep$RANDOM
```

---

## Komplette Infrastruktur: App Service + Key Vault

Erstelle `main.bicep`:

```bicep
@description('Prefix für alle Ressourcennamen')
param prefix string = 'bicep'

@description('Azure-Region')
param location string = resourceGroup().location

@description('Tenant-ID für Key Vault')
param tenantId string = tenant().tenantId

// Variables
var appServicePlanName = '${prefix}-plan'
var appName = '${prefix}-app-${uniqueString(resourceGroup().id)}'
var keyVaultName = 'kv-${prefix}-${uniqueString(resourceGroup().id)}'

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: 'F1'
    tier: 'Free'
  }
  properties: {
    reserved: true  // Linux
  }
}

// App Service
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'PYTHON|3.12'
      appSettings: [
        {
          name: 'KEY_VAULT_URI'
          value: keyVault.properties.vaultUri
        }
      ]
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

// Key Vault
resource keyVault 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: tenantId
    sku: {
      family: 'A'
      name: 'standard'
    }
    enableRbacAuthorization: true
  }
}

// RBAC: App Service darf Secrets lesen
resource kvSecretUserRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, appService.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4633458b-17de-408a-b874-0445c86b69e6'  // Key Vault Secrets User
    )
    principalId: appService.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

// Outputs
output appUrl string = 'https://${appService.properties.defaultHostName}'
output keyVaultUri string = keyVault.properties.vaultUri
output appIdentityPrincipalId string = appService.identity.principalId
```

```bash
az deployment group create \
  --resource-group rg-devops \
  --template-file main.bicep \
  --parameters prefix=training
```

!!! success "Managed Identity + Key Vault in einem Schritt"
    Das Template erstellt automatisch eine Managed Identity auf dem App Service und weist ihr die `Key Vault Secrets User`-Rolle zu – alles in einem einzigen Deployment.

---

## Bicep Module: Wiederverwendbarkeit

Bicep-Module sind eigenständige `.bicep`-Dateien die als Bausteine verwendet werden:

Erstelle `modules/storage.bicep`:

```bicep
param name string
param location string = resourceGroup().location

resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output id string = storage.id
```

In `main.bicep` verwenden:

```bicep
module storageModule 'modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    name: 'stmodule${uniqueString(resourceGroup().id)}'
  }
}

// Output des Moduls verwenden
output storageId string = storageModule.outputs.id
```

---

## ARM ↔ Bicep konvertieren

```bash
# ARM Template → Bicep (Decompile)
az bicep decompile --file storage-account.json
# Erstellt: storage-account.bicep

# Bicep → ARM Template (Build)
az bicep build --file main.bicep
# Erstellt: main.json (ARM Template)
```

---

## What-If mit Bicep

```bash
az deployment group what-if \
  --resource-group rg-devops \
  --template-file main.bicep \
  --parameters prefix=training
```

---

## Bicep in GitHub Actions

```yaml
- name: Bicep deployen
  uses: azure/login@v2
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Infrastruktur deployen
  run: |
    az deployment group create \
      --resource-group rg-devops \
      --template-file main.bicep \
      --parameters prefix=training
```

!!! tip "Azure Credentials für GitHub Actions"
    Erstelle einen Service Principal: 
    `az ad sp create-for-rbac --name "github-actions" --role Contributor --scopes /subscriptions/SUBSCRIPTION-ID --json-auth`
    Den JSON-Output als `AZURE_CREDENTIALS` Secret in GitHub hinterlegen.

---

## Challenge

!!! question "Challenge: Vollständige App deployen"
    Erstelle ein Bicep-Template das folgendes deployt:
    
    - Storage Account (Standard_LRS)
    - Container App Environment
    - Eine Container App die das `hello-world`-Image ausführt
    
    Nutze Parameter für den Prefix und Outputs für die App-URL.

??? success "Hinweis"
    Container App in Bicep:
    ```bicep
    resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
      name: '${prefix}-app'
      location: location
      properties: {
        managedEnvironmentId: environment.id
        configuration: {
          ingress: {
            external: true
            targetPort: 80
          }
        }
        template: {
          containers: [{
            name: 'hello'
            image: 'mcr.microsoft.com/azuredocs/containerapps-helloworld:latest'
          }]
        }
      }
    }
    ```

---

Weiter zu [Modul 37 – Terraform mit Azure: Plattformunabhängiges IaC](modul-37-terraform.md) →
