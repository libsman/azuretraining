# Modul 37 – Terraform mit Azure: Plattformunabhängiges IaC

## Lernziele

Nach diesem Modul kannst du:

- Terraform als plattformunabhängiges IaC-Tool beschreiben
- Die Terraform-Grundbefehle (`init`, `plan`, `apply`, `destroy`) anwenden
- Den Azure Provider konfigurieren und authentifizieren
- Einfache Azure-Ressourcen per Terraform deployen
- Terraform State erklären und Remote State in Azure konfigurieren
- Den Unterschied zwischen Terraform und Bicep/ARM abwägen

---

## Hintergrund: Warum Terraform?

**On-Prem-Vergleich:** Bicep ist Azure-spezifisch – perfekt für Azure, aber nutzlos auf AWS oder GCP. In vielen Unternehmen wird eine Multi-Cloud-Strategie gefahren. **Terraform** von HashiCorp ist Cloud-agnostisch: derselbe Workflow für Azure, AWS, GCP und hunderte weiterer Anbieter.

**Terraform vs. Bicep:**

| | Terraform | Bicep |
|--|-----------|-------|
| Azure-spezifisch | Nein (Multi-Cloud) | Ja |
| Syntax | HCL (HashiCorp Config Language) | Bicep |
| State-Management | Eigene State-Datei | ARM-seitig |
| Community | Sehr groß (Provider für alles) | Microsoft-gepflegt |
| Import bestehender Ressourcen | `terraform import` | `az bicep decompile` |
| Empfehlung | Multi-Cloud oder bestehende Terraform-Infrastruktur | Azure-Only |

---

## Terraform installieren

```bash
# Windows (winget)
winget install HashiCorp.Terraform

# Prüfen
terraform version
```

In der Cloud Shell ist Terraform bereits verfügbar.

---

## Terraform-Projekt aufbauen

Erstelle einen neuen Ordner `terraform-training` und darin `main.tf`:

```hcl
# Provider konfigurieren
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
  required_version = ">= 1.5.0"
}

provider "azurerm" {
  features {}
}

# Resource Group
resource "azurerm_resource_group" "training" {
  name     = "rg-terraform-training"
  location = "West Europe"
  tags = {
    environment = "training"
    managed_by  = "terraform"
  }
}

# Storage Account
resource "azurerm_storage_account" "training" {
  name                     = "sttf${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.training.name
  location                 = azurerm_resource_group.training.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  allow_nested_items_to_be_public = false
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}
```

Füge `random` als Provider hinzu – erweitere den `terraform`-Block:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}
```

---

## Terraform-Workflow: init → plan → apply

```bash
# 1. Provider herunterladen und initialisieren
terraform init

# 2. Vorschau: was wird erstellt/geändert/gelöscht?
terraform plan

# 3. Ausführen (bestätigen mit "yes")
terraform apply

# Mit automatischer Bestätigung:
terraform apply -auto-approve
```

Die Ausgabe von `plan` zeigt:
- `+` Ressource wird **erstellt**
- `~` Ressource wird **geändert**
- `-` Ressource wird **gelöscht**
- `-/+` Ressource wird **ersetzt** (gelöscht + neu erstellt)

---

## Variablen und Outputs

Erstelle `variables.tf`:

```hcl
variable "location" {
  description = "Azure-Region für alle Ressourcen"
  type        = string
  default     = "West Europe"
}

variable "environment" {
  description = "Umgebung: dev, staging oder prod"
  type        = string
  default     = "training"

  validation {
    condition     = contains(["dev", "staging", "prod", "training"], var.environment)
    error_message = "Erlaubte Werte: dev, staging, prod, training"
  }
}
```

Erstelle `outputs.tf`:

```hcl
output "resource_group_name" {
  description = "Name der Resource Group"
  value       = azurerm_resource_group.training.name
}

output "storage_account_name" {
  description = "Name des Storage Accounts"
  value       = azurerm_storage_account.training.name
}

output "storage_primary_endpoint" {
  description = "Primärer Blob-Endpunkt"
  value       = azurerm_storage_account.training.primary_blob_endpoint
}
```

Variablen beim Apply übergeben:

```bash
terraform apply -var="environment=dev"

# Oder in einer tfvars-Datei:
# terraform.tfvars:
# environment = "dev"
# location    = "North Europe"
terraform apply
```

---

## Terraform State

Terraform speichert den **aktuellen Zustand** der Infrastruktur in einer `terraform.tfstate`-Datei. So weiß Terraform was bereits existiert und was geändert werden muss.

!!! warning "State-Datei niemals in Git committen"
    `terraform.tfstate` enthält sensible Informationen (IDs, ggf. Passwörter). Füge sie zu `.gitignore` hinzu:
    ```
    .terraform/
    terraform.tfstate
    terraform.tfstate.backup
    *.tfvars
    ```

### Remote State in Azure Storage

In Teams muss der State zentral gespeichert werden:

```bash
# Storage Account für State erstellen
az storage account create \
  --name sttfstate$RANDOM \
  --resource-group rg-devops \
  --sku Standard_LRS \
  --allow-blob-public-access false

# Container erstellen
az storage container create \
  --name tfstate \
  --account-name STORAGE-NAME
```

In `main.tf` den Backend-Block hinzufügen:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-devops"
    storage_account_name = "STORAGE-NAME"
    container_name       = "tfstate"
    key                  = "training.tfstate"
  }
  required_providers { ... }
}
```

```bash
# Backend neu initialisieren (State migrieren)
terraform init -migrate-state
```

---

## App Service mit Terraform

Erweitere `main.tf` um einen App Service:

```hcl
resource "azurerm_service_plan" "training" {
  name                = "plan-terraform-training"
  resource_group_name = azurerm_resource_group.training.name
  location            = azurerm_resource_group.training.location
  os_type             = "Linux"
  sku_name            = "F1"
}

resource "azurerm_linux_web_app" "training" {
  name                = "app-tf-${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.training.name
  location            = azurerm_resource_group.training.location
  service_plan_id     = azurerm_service_plan.training.id

  site_config {
    application_stack {
      python_version = "3.12"
    }
  }

  identity {
    type = "SystemAssigned"
  }

  tags = {
    environment = var.environment
    managed_by  = "terraform"
  }
}

output "app_url" {
  value = "https://${azurerm_linux_web_app.training.default_hostname}"
}
```

---

## Ressourcen löschen

```bash
# Alle Terraform-verwalteten Ressourcen löschen
terraform destroy

# Mit automatischer Bestätigung:
terraform destroy -auto-approve
```

!!! warning "destroy löscht alles"
    `terraform destroy` löscht alle Ressourcen die Terraform verwaltet – irreversibel. In Produktion immer zuerst `terraform plan -destroy` ausführen um zu sehen was gelöscht wird.

---

## Challenge

!!! question "Challenge: Key Vault per Terraform"
    Füge zu deinem Terraform-Projekt einen Key Vault hinzu:
    
    - `azurerm_key_vault`-Ressource mit `enable_rbac_authorization = true`
    - RBAC-Zuweisung: App Service Managed Identity bekommt `Key Vault Secrets User`
    - Output: Key Vault URI

??? success "Hinweis"
    ```hcl
    data "azurerm_client_config" "current" {}
    
    resource "azurerm_key_vault" "training" {
      name                = "kv-tf-${random_string.suffix.result}"
      location            = azurerm_resource_group.training.location
      resource_group_name = azurerm_resource_group.training.name
      tenant_id           = data.azurerm_client_config.current.tenant_id
      sku_name            = "standard"
      enable_rbac_authorization = true
    }
    
    resource "azurerm_role_assignment" "kv_secrets_user" {
      scope                = azurerm_key_vault.training.id
      role_definition_name = "Key Vault Secrets User"
      principal_id         = azurerm_linux_web_app.training.identity[0].principal_id
    }
    ```

---

Weiter zu [Modul 38 – Azure Policy & Resource Locks: Compliance automatisieren](modul-38-policy.md) →
