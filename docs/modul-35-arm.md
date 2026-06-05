# Modul 35 – ARM Templates: Infrastruktur als JSON verstehen

## Lernziele

Nach diesem Modul kannst du:

- Das Konzept von Infrastructure as Code (IaC) erklären
- Den Aufbau eines ARM Templates (JSON) lesen und verstehen
- Eine Resource Group und einfache Ressourcen per ARM Template deployen
- Parameter und Variablen in ARM Templates nutzen
- Den Unterschied zwischen ARM (JSON) und Bicep (nächstes Modul) erklären

---

## Hintergrund: Was ist Infrastructure as Code?

**On-Prem-Vergleich:** Wenn du on-prem einen neuen Server brauchst, klickst du dich durch den Hyper-V-Manager, fügst CPU/RAM hinzu, installierst Windows, installierst Rollen. Das funktioniert für einen Server – aber was wenn du 50 Server mit identischer Konfiguration brauchst? Oder wenn du die Umgebung nach einem Disaster exakt wiederherstellen musst?

**Infrastructure as Code (IaC)** löst das: Infrastruktur wird als Code beschrieben, in Git gespeichert und kann beliebig oft deployed werden – immer identisch.

**Vorteile:**
- ✅ Versioniert (Git-History: wann hat wer was geändert?)
- ✅ Wiederholbar (Dev = Staging = Prod)
- ✅ Dokumentiert (der Code ist die Dokumentation)
- ✅ Testbar (Deployment kann in CI/CD validiert werden)

**ARM Templates** (Azure Resource Manager) sind die native IaC-Sprache für Azure – JSON-Dateien die Azure-Ressourcen beschreiben.

---

## ARM Template Aufbau

Ein ARM Template ist eine JSON-Datei mit festgelegter Struktur:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    // Eingabewerte die beim Deployment übergeben werden
  },
  "variables": {
    // Berechnete Werte für die Wiederverwendung
  },
  "resources": [
    // Azure-Ressourcen die erstellt werden
  ],
  "outputs": {
    // Werte die nach dem Deployment zurückgegeben werden
  }
}
```

---

## Erstes ARM Template: Storage Account

Erstelle `storage-account.json` in der Cloud Shell:

```bash
code storage-account.json
```

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "type": "string",
      "minLength": 3,
      "maxLength": 24,
      "metadata": {
        "description": "Name des Storage Accounts (global eindeutig, nur Kleinbuchstaben und Zahlen)"
      }
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]",
      "metadata": {
        "description": "Azure-Region (Standard: Region der Resource Group)"
      }
    }
  },
  "variables": {
    "storageSkuName": "Standard_LRS"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-01-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "[variables('storageSkuName')]"
      },
      "kind": "StorageV2",
      "properties": {
        "accessTier": "Hot"
      }
    }
  ],
  "outputs": {
    "storageAccountId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageAccountName'))]"
    }
  }
}
```

### Deployment

```bash
# Resource Group erstellen (falls noch nicht vorhanden)
az group create --name rg-devops --location westeurope

# ARM Template deployen
az deployment group create \
  --resource-group rg-devops \
  --template-file storage-account.json \
  --parameters storageAccountName=stdevops$RANDOM
```

---

## Parameter-Datei

Parameter können auch in einer separaten JSON-Datei stehen – das ermöglicht verschiedene Konfigurationen für Dev/Prod:

Erstelle `storage-account.parameters.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "stdevopstraining01"
    }
  }
}
```

```bash
az deployment group create \
  --resource-group rg-devops \
  --template-file storage-account.json \
  --parameters @storage-account.parameters.json
```

---

## What-If: Vorschau vor dem Deployment

Bevor du etwas deployst, kannst du dir anzeigen lassen was sich ändern würde:

```bash
az deployment group what-if \
  --resource-group rg-devops \
  --template-file storage-account.json \
  --parameters storageAccountName=stdevopstraining01
```

Die Ausgabe zeigt: `+ Create`, `~ Modify`, `- Delete` – du siehst genau was passiert, ohne es wirklich auszuführen.

---

## Komplexeres Template: App Service + Plan

Erstelle `app-service.json`:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "appName": {
      "type": "string"
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "planName": "[concat(parameters('appName'), '-plan')]"
  },
  "resources": [
    {
      "type": "Microsoft.Web/serverfarms",
      "apiVersion": "2022-09-01",
      "name": "[variables('planName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "F1",
        "tier": "Free"
      },
      "properties": {
        "reserved": true
      }
    },
    {
      "type": "Microsoft.Web/sites",
      "apiVersion": "2022-09-01",
      "name": "[parameters('appName')]",
      "location": "[parameters('location')]",
      "dependsOn": [
        "[resourceId('Microsoft.Web/serverfarms', variables('planName'))]"
      ],
      "properties": {
        "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('planName'))]",
        "siteConfig": {
          "linuxFxVersion": "PYTHON|3.12"
        }
      }
    }
  ],
  "outputs": {
    "appUrl": {
      "type": "string",
      "value": "[concat('https://', reference(parameters('appName')).defaultHostName)]"
    }
  }
}
```

```bash
az deployment group create \
  --resource-group rg-devops \
  --template-file app-service.json \
  --parameters appName=app-arm-$RANDOM
```

---

## ARM Templates aus dem Portal exportieren

Für bestehende Ressourcen kann Azure das ARM Template automatisch generieren:

1. Öffne eine Ressource im Azure Portal
2. Links im Menü: **Export template**
3. Du siehst das Template und kannst es herunterladen

Das ist nützlich um zu verstehen wie Ressourcen in ARM beschrieben werden.

!!! info "ARM vs. Bicep"
    ARM Templates sind mächtig aber verbose – JSON mit vielen verschachtelten Strukturen. **Bicep** (nächstes Modul) ist die modernere Alternative: deutlich lesbarer, weniger Codezeilen, kompiliert aber intern zu ARM. Du kannst heute neu mit Bicep starten – ARM-Kenntnisse helfen trotzdem das Konzept zu verstehen.

---

## Challenge

!!! question "Challenge: Key Vault per ARM Template"
    Erstelle ein ARM Template das folgendes deployt:
    
    - Einen Azure Key Vault
    - Parameter: `keyVaultName`, `tenantId` (deine Tenant-ID)
    - Property: `enableRbacAuthorization: true`
    
    Deine Tenant-ID findest du mit: `az account show --query tenantId -o tsv`

??? success "Hinweis"
    ```json
    {
      "type": "Microsoft.KeyVault/vaults",
      "apiVersion": "2023-07-01",
      "name": "[parameters('keyVaultName')]",
      "location": "[parameters('location')]",
      "properties": {
        "tenantId": "[parameters('tenantId')]",
        "sku": { "family": "A", "name": "standard" },
        "enableRbacAuthorization": true
      }
    }
    ```

---

Weiter zu [Modul 36 – Bicep: Infrastruktur als Code – modern und lesbar](modul-36-bicep.md) →
