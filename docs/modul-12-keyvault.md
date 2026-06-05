# Modul 12 – Azure Key Vault: Secrets und API-Keys sicher speichern

## Lernziele

Nach diesem Modul kannst du:

- Einen Azure Key Vault erstellen und mit RBAC absichern
- Secrets, Schlüssel und Zertifikate im Key Vault anlegen
- Secrets über das Portal, die CLI und Python abfragen
- Erklären warum Key Vault besser ist als `.env`-Dateien oder hartcodierte Passwörter

---

## Hintergrund: Das Problem mit gespeicherten Passwörtern

**On-Prem-Vergleich:** Jeder kennt es: Passwörter in Batch-Skripten, Verbindungsstrings in `web.config`, API-Keys in `.env`-Dateien. Das ist Alltag – und ein Sicherheitsrisiko. Azure Key Vault ist das Cloud-Äquivalent zu einem **zentralen Passwort-Manager** (KeePass, CyberArk, HashiCorp Vault) – vollständig von Microsoft verwaltet.

**Das Problem in der Praxis:**

```python
# ❌ So nicht:
DB_PASSWORD = "MeinPasswort123"
API_KEY = "sk-abc123..."

# ✅ So:
DB_PASSWORD = get_from_keyvault("db-password")
```

Hartcodierte Secrets tauchen früher oder später in Git-Repositories auf, in Logfiles oder Screenshots. Ein einziger Leak kann teure Folgen haben.

**Was kann Key Vault speichern?**

| Typ | Beschreibung | Beispiel |
|-----|-------------|---------|
| **Secrets** | Beliebige Text-Strings | Passwörter, API-Keys, Verbindungsstrings |
| **Keys** | Kryptografische Schlüssel | RSA, EC – für Ver-/Entschlüsselung |
| **Certificates** | TLS/SSL-Zertifikate | Mit automatischer Erneuerung |

**Warum Key Vault statt Umgebungsvariablen?**

- Secrets leben **nie im Code** oder in Git
- Vollständiges **Audit Log**: Wer hat wann welches Secret abgerufen?
- **RBAC**: Nur autorisierte Identitäten können lesen
- **Versionierung**: Alte Versionen bleiben für Rollbacks erhalten
- **Managed Identity**: Apps holen Secrets ohne eigene Anmeldedaten (Lernpfad 4)

---

## Key Vault erstellen

### Schritt 1: Dienst öffnen

1. Tippe im Portal **`Key vaults`** → **+ Create**

### Schritt 2: Basics

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-netzwerk` |
| Key vault name | `kv-aztraining-XXXX` (XXXX = deine Initialen + Datum, z.B. `kv-aztraining-ms0605`) |
| Region | `West Europe` |
| Pricing tier | `Standard` |

!!! info "Name muss weltweit eindeutig sein"
    Key Vault Namen sind Teil der URL `https://kv-aztraining-XXXX.vault.azure.net` – sie müssen global eindeutig sein.

!!! tip "Standard vs. Premium"
    Standard: Software-geschützte Schlüssel – ausreichend für fast alle Anwendungen. Premium: Hardware Security Module (HSM) – für hochsensible, regulierte Umgebungen. Für dieses Training: Standard.

### Schritt 3: Access Configuration

1. Wechsle zum Tab **Access configuration**
2. Permission model: **Azure role-based access control (RBAC)** ✅ (Haken setzen)

!!! info "RBAC vs. Vault Access Policies"
    Key Vault unterstützt zwei Zugriffsmodelle:
    - **RBAC** (empfohlen): Berechtigungen über Azure-Rollen – konsistent mit dem Rest von Azure
    - **Access Policies** (legacy): Key-Vault-spezifisches Modell, wird noch unterstützt aber nicht empfohlen
    
    Neue Vaults sollten immer RBAC nutzen.

### Schritt 4: Review + Create

**Review + create** → **Create**

---

## RBAC: Dir selbst Zugriff gewähren

Mit RBAC-Modell musst du dir explizit die Berechtigung geben, Secrets zu verwalten.

1. Navigiere zu deinem Key Vault → **Access control (IAM)**
2. Klicke **+ Add** → **Add role assignment**
3. Suche und wähle die Rolle: **Key Vault Secrets Officer**
4. Tab **Members** → **+ Select members** → suche deinen Benutzernamen
5. **Review + assign** → **Assign**

!!! info "Key Vault Rollen im Überblick"
    | Rolle | Darf |
    |-------|------|
    | Key Vault Secrets User | Secrets **lesen** |
    | Key Vault Secrets Officer | Secrets **lesen, erstellen, aktualisieren, löschen** |
    | Key Vault Administrator | Alles + RBAC-Konfiguration ändern |

---

## Secrets erstellen

### Im Portal

1. Navigiere zu `kv-aztraining-XXXX` → klicke links auf **Secrets**
2. Klicke **+ Generate/Import**

**Secret 1: Datenbankpasswort**

| Feld | Wert |
|------|------|
| Upload options | `Manual` |
| Name | `db-password` |
| Secret value | `P@ssw0rd-Datenbank-2026!` |

Klicke **Create**.

**Secret 2: API-Key**

| Name | Secret value |
|------|-------------|
| `external-api-key` | `sk-beispiel-apikey-abc123` |

!!! warning "Echte Secrets nie öffentlich"
    Für dieses Training nutzt du Platzhalter-Werte. In der Praxis kommen hier echte Passwörter rein – die du natürlich **nicht** in Screenshots oder Dokus zeigst.

---

## Secrets über die Azure CLI abfragen

Öffne die **Cloud Shell** im Azure Portal.

### Alle Secrets auflisten

```bash
az keyvault secret list \
  --vault-name kv-aztraining-XXXX \
  --query "[].name" \
  -o table
```

### Secret-Wert abrufen

```bash
az keyvault secret show \
  --vault-name kv-aztraining-XXXX \
  --name db-password \
  --query "value" \
  --output tsv
```

### Secret in Variable speichern (für Skripte)

```bash
DB_PASS=$(az keyvault secret show \
  --vault-name kv-aztraining-XXXX \
  --name db-password \
  --query "value" \
  --output tsv)

echo "Passwort geladen: ${#DB_PASS} Zeichen"
```

!!! success "Kein Passwort im Code!"
    Genau so würde ein echtes Skript das Passwort laden – ohne es im Code zu hinterlegen.

---

## Secret in Python abfragen

Öffne die Cloud Shell und erstelle eine neue Python-Datei:

```bash
code keyvault_demo.py
```

Füge folgenden Code ein:

```python
import subprocess

VAULT_NAME = "kv-aztraining-XXXX"  # Deinen Vault-Namen einsetzen

def get_secret(name: str) -> str:
    """Secret aus Azure Key Vault über die CLI laden."""
    result = subprocess.run(
        ["az", "keyvault", "secret", "show",
         "--vault-name", VAULT_NAME,
         "--name", name,
         "--query", "value",
         "--output", "tsv"],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"Secret '{name}' nicht gefunden: {result.stderr}")
    return result.stdout.strip()

# Secrets laden und nutzen
db_password   = get_secret("db-password")
api_key       = get_secret("external-api-key")

print(f"✅ db-password geladen ({len(db_password)} Zeichen)")
print(f"✅ external-api-key geladen: {api_key[:8]}...")
print()
print("Verbindung zur Datenbank wird mit geladenen Credentials aufgebaut...")
```

Speichern und ausführen:

```bash
python keyvault_demo.py
```

!!! info "In Produktionsumgebungen"
    In echten Anwendungen nutzt man das **Azure SDK** (`azure-keyvault-secrets` + `azure-identity`) zusammen mit **Managed Identity** – dann braucht die App gar keine eigenen Anmeldedaten. Das kommt in Lernpfad 4 (Identity & Access).

---

## Audit Log ansehen

Jeder Zugriff auf einen Secret wird protokolliert.

1. Navigiere zu deinem Key Vault → **Monitoring** → **Diagnostic settings**
2. Klicke **+ Add diagnostic setting**
3. Aktiviere: `AuditEvent` → Ziel: **Send to Log Analytics workspace** (oder einfach anschauen unter **Activity log**)

Unter **Activity log** siehst du alle Aktionen: Wer hat wann welches Secret abgerufen, erstellt oder geändert.

!!! tip "Compliance-Anforderung"
    In regulierten Umgebungen (DSGVO, ISO 27001, BSI IT-Grundschutz) ist ein lückenloses Audit-Log für Secret-Zugriffe oft Pflicht. Key Vault erfüllt diese Anforderung direkt.

---

## Challenge

!!! question "Challenge: Secret-Rotation"
    1. Erstelle eine neue Version von `db-password`:
       - Navigiere zu `db-password` im Key Vault → **New version**
       - Trage einen neuen Wert ein: `NeuesP@ssw0rd-2026!`
    2. Frage das Secret via CLI ab – welchen Wert bekommst du? (immer die neueste Version)
    3. Liste alle Versionen des Secrets auf

??? success "Hinweis"
    ```bash
    # Alle Versionen eines Secrets anzeigen
    az keyvault secret list-versions \
      --vault-name kv-aztraining-XXXX \
      --name db-password \
      --query "[].{version: id, created: attributes.created, enabled: attributes.enabled}" \
      -o table
    ```
    
    Key Vault behält alle Versionen – nur die aktuellste aktivierte Version wird bei `secret show` zurückgegeben. Ältere Versionen bleiben für Audits erhalten (können aber deaktiviert werden).

---

Weiter zu [Modul 13 – Private Endpoints: Dienste intern erreichbar machen](modul-13-privateendpoints.md) →
