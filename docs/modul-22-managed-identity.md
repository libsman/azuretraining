# Modul 22 – Managed Identity: Apps ohne Passwörter authentifizieren

## Lernziele

Nach diesem Modul kannst du:

- Erklären was Managed Identity ist und warum man sie braucht
- System-assigned und User-assigned Managed Identity unterscheiden
- Einer App Service Web-App eine Managed Identity geben
- Der Identity eine RBAC-Rolle auf eine Azure-Ressource zuweisen
- Aus einer Azure-App ohne Passwort auf Blob Storage und Key Vault zugreifen

---

## Hintergrund: Das Problem mit Passwörtern in Apps

Stell dir vor: deine Web-App muss auf einen Storage Account zugreifen. Die naheliegende Lösung:

```python
# ❌ So macht man es NICHT
STORAGE_KEY = "abcdef123456..."  # im Code gespeichert
```

Was passiert wenn dieser Code in Git landet? Der Key ist öffentlich. Was wenn der Key rotiert werden muss? Code ändern und neu deployen. Was wenn jemand den Code klaut? Alle Ressourcen mit diesem Key sind kompromittiert.

**Managed Identity löst dieses Problem grundlegend:**

```
App Service  →  "Wer bist du?"  →  Managed Identity  →  Microsoft Entra ID
                                                               ↓
                                                    Token ausstellen
                                                               ↓
App Service  →  Azure Storage  →  Token prüfen  →  Zugriff erlaubt
```

Die App hat eine **Identität in Entra ID** – genau wie ein Benutzer. Azure stellt automatisch kurzlebige Tokens aus, ohne dass du ein Passwort speichern musst. **Kein Secret im Code, kein Ablaufdatum verwalten, kein Rotations-Problem.**

---

## System-assigned vs. User-assigned

| | System-assigned | User-assigned |
|--|-----------------|---------------|
| Lebensdauer | An die Ressource gebunden | Unabhängige Ressource |
| Wird gelöscht wenn | Ressource gelöscht wird | Du sie löschst |
| Kann mehreren Ressourcen zugewiesen werden | ❌ Nur einer | ✅ Beliebig vielen |
| Typischer Use Case | Einfache 1:1-Beziehung | Mehrere Apps teilen dieselbe Identity |

Für den Anfang ist **System-assigned** einfacher – du aktivierst sie direkt auf der Ressource.

---

## Managed Identity aktivieren (App Service)

### Schritt 1: App Service öffnen

Navigiere zu deiner Web App `webapp-aztraining-XXXX` (aus Lernpfad 1, Modul 5).

Falls du die Ressourcen aus LP1 bereits gelöscht hast, erstelle eine neue Web App:

??? info "Neue Web App erstellen (optional)"
    **App Services** → **+ Create** → **Web App**
    
    | Feld | Wert |
    |------|------|
    | Resource group | `rg-identity` |
    | Name | `app-identity-XXXX` |
    | Runtime stack | `Python 3.12` |
    | OS | `Linux` |
    | Region | `West Europe` |
    | Plan | `Free F1` |

### Schritt 2: System-assigned Identity aktivieren

1. Links im App Service-Menü: **Settings** → **Identity**
2. Tab **System assigned**
3. Status von **Off** auf **On** setzen
4. Klicke **Save** → **Yes** bestätigen

Azure erstellt jetzt einen Service Principal in Entra ID. Die angezeigte **Object ID** ist die Identität deiner App.

!!! success "App hat jetzt eine Identität"
    Du kannst die neue Identität in Entra ID sehen: **Microsoft Entra ID** → **Enterprise applications** → Filter: `Managed Identities` → deine App erscheint in der Liste.

---

## Storage Account für Managed Identity vorbereiten

### Schritt 1: Storage Account erstellen

1. **Storage accounts** → **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-identity` (oder deine vorhandene) |
| Storage account name | `staztraining-XXXX` |
| Region | `West Europe` |
| Redundancy | `LRS` |

**Review + create** → **Create**

### Schritt 2: Blob Container erstellen

1. Gehe zu `staztraining-XXXX` → **Containers** → **+ Container**
2. Name: `daten`
3. Public access level: `Private`
4. **Create**

### Schritt 3: RBAC-Rolle für die Managed Identity vergeben

Jetzt gibst du der Identität deiner App Zugriff auf den Storage:

1. Gehe zu `staztraining-XXXX` → **Access control (IAM)**
2. Klicke oben auf **+ Add** → **Add role assignment**
3. Wähle die Rolle: **Storage Blob Data Contributor**
4. Klicke auf **Next**
5. Assign access to: **Managed identity**
6. Klicke auf **+ Select members**
7. Managed identity: **App Service**
8. Wähle deine App aus der Liste
9. **Select** → **Review + assign** → **Review + assign**

!!! info "Warum Storage Blob Data Contributor?"
    Die Rolle `Storage Blob Data Contributor` erlaubt Lesen, Schreiben und Löschen von Blobs. Es gibt auch:
    - `Storage Blob Data Reader`: nur Lesen
    - `Storage Blob Data Owner`: alles inkl. ACL-Management
    
    **Least Privilege**: immer die minimale Rolle wählen die ausreicht.

---

## Python-Code: Mit Managed Identity auf Storage zugreifen

### In der Cloud Shell

```bash
pip install azure-storage-blob azure-identity --quiet
code managed_identity_demo.py
```

```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

# Kein Passwort, kein Key – nur die Identität der Umgebung
credential = DefaultAzureCredential()

# Storage Account URL (kein Key!)
account_url = "https://staztraining-XXXX.blob.core.windows.net"
client = BlobServiceClient(account_url=account_url, credential=credential)

container_name = "daten"

# Datei hochladen
inhalt = b"Hallo von Managed Identity! Kein Passwort im Code."
blob_client = client.get_blob_client(container=container_name, blob="test.txt")
blob_client.upload_blob(inhalt, overwrite=True)
print("✅ Datei hochgeladen")

# Datei herunterladen und anzeigen
download = blob_client.download_blob()
print(f"✅ Inhalt: {download.readall().decode()}")
```

```bash
python3 managed_identity_demo.py
```

!!! info "DefaultAzureCredential – wie das funktioniert"
    `DefaultAzureCredential` probiert der Reihe nach verschiedene Authentifizierungsmethoden:
    
    1. Umgebungsvariablen (für CI/CD-Pipelines)
    2. **Managed Identity** (wenn in Azure ausgeführt) ← das nutzen wir
    3. Azure CLI Login (lokal auf deinem PC)
    4. Visual Studio Code-Login
    5. Interactive Browser
    
    In der Cloud Shell und in einer App Service-App greift Methode 2 automatisch. Lokal auf deinem PC greift Methode 3 oder 4 – du brauchst keine Codeänderung für lokale Entwicklung!

---

## Managed Identity für Key Vault

Das kennst du schon aus Modul 12 und Modul 19. Hier die Kurzfassung als Referenz:

### Rolle vergeben

Auf dem Key Vault: **Access control (IAM)** → **+ Add** → **Add role assignment**
- Rolle: **Key Vault Secrets User**
- Managed identity: deine App

### Code (keine Passwörter!)

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(
    vault_url="https://kv-aztraining-XXXX.vault.azure.net",
    credential=credential
)

geheimnis = client.get_secret("mein-secret")
print(f"Secret: {geheimnis.value}")
```

---

## User-assigned Managed Identity

Für Szenarien wo mehrere Apps dieselbe Identity teilen sollen:

### Erstellen

1. Suche im Portal nach **`Managed Identities`** → **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-identity` |
| Region | `West Europe` |
| Name | `id-shared-apps` |

**Review + create** → **Create**

### Zuweisen

1. App Service → **Identity** → Tab **User assigned** → **+ Add**
2. Wähle `id-shared-apps`
3. **Add**

Jetzt können mehrere App Services dieselbe Identity nutzen – und RBAC-Rollen nur einmal vergeben werden.

---

## Challenge

!!! question "Challenge: Managed Identity für Cosmos DB"
    Azure Cosmos DB unterstützt ebenfalls Managed Identity (via RBAC auf Datenebene).
    
    1. Weise deiner App-Managed Identity die Cosmos DB-Rolle **Cosmos DB Built-in Data Reader** zu
    2. Schreibe ein Python-Script das mit `DefaultAzureCredential` auf einen Cosmos DB-Container zugreift und Dokumente liest
    
    Dokumentation: [Cosmos DB RBAC](https://learn.microsoft.com/de-de/azure/cosmos-db/how-to-setup-rbac)

??? success "Hinweis"
    Cosmos DB RBAC wird anders vergeben als normales Azure RBAC – über den **Data plane**-Bereich:
    
    ```bash
    # Rolle in Cosmos DB vergeben (CLI)
    az cosmosdb sql role assignment create \
      --account-name cosmos-aztraining-XXXX \
      --resource-group rg-datenbanken \
      --role-definition-name "Cosmos DB Built-in Data Reader" \
      --principal-id <Object-ID deiner Managed Identity> \
      --scope "/"
    ```
    
    Im Python-Code:
    ```python
    from azure.identity import DefaultAzureCredential
    from azure.cosmos import CosmosClient
    
    credential = DefaultAzureCredential()
    client = CosmosClient(
        url="https://cosmos-aztraining-XXXX.documents.azure.com:443/",
        credential=credential  # kein Key mehr!
    )
    ```

---

Weiter zu [Modul 23 – Azure RBAC vertieft: Eigene Rollen und Scopes](modul-23-rbac.md) →
