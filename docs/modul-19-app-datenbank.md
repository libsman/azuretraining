# Modul 19 – App + Datenbank sicher verbinden

## Lernziele

Nach diesem Modul kannst du:

- Eine App Service Web-App mit Azure SQL verbinden
- Verbindungsstrings sicher in Key Vault speichern und über App Settings referenzieren
- Managed Identity nutzen damit die App ohne Passwort auf Key Vault zugreift
- Eine einfache Python Flask-App deployen die Datenbank-Ergebnisse anzeigt
- Das Zusammenspiel von App Service, Key Vault, SQL und Redis verstehen

---

## Hintergrund: Das Problem mit Passwörtern im Code

**Was passiert in der Praxis oft (falsch):**

```python
# ❌ NIEMALS SO – Passwort im Code
conn = pyodbc.connect("Server=sql-...;Password=MeinPasswort123!")
```

Wenn dieser Code in Git landet – und das passiert häufiger als man denkt – ist das Passwort öffentlich. Git-History vergisst nicht.

**Die richtige Architektur:**

```
App Service → Key Vault → Datenbankpasswort
     ↑                         ↓
  Managed Identity          Verbindungsstring
  (kein Passwort nötig)
```

**Ablauf:**
1. App Service hat eine **Managed Identity** (Modul 12 – Key Vault)
2. Key Vault hat eine **Access Policy** für diese Identity
3. App liest Verbindungsstring zur Laufzeit aus Key Vault
4. App verbindet sich mit SQL – Passwort war nie im Code

---

## Voraussetzungen prüfen

Wir bauen auf Modul 12 (Key Vault) und Modul 15 (Azure SQL) auf. Stelle sicher, dass folgendes existiert:

- ✅ `rg-aztraining` mit App Service `app-aztraining-XXXX` (aus LP1, Modul 5)
- ✅ `rg-datenbanken` mit `db-aztraining` auf Server `sql-aztraining-XXXX`

Falls du die LP1-Ressourcen bereits aufgeräumt hast, erstelle schnell einen neuen App Service:

??? info "App Service neu erstellen (optional)"
    1. **App Services** → **+ Create** → **Web App**
    
    | Feld | Wert |
    |------|------|
    | Resource group | `rg-datenbanken` |
    | Name | `app-db-XXXX` |
    | Runtime stack | `Python 3.12` |
    | OS | `Linux` |
    | Region | `West Europe` |
    | Plan | `Free F1` |
    
    Aktiviere unter **Identity** → **System assigned** → `On`

---

## Verbindungsstring in Key Vault speichern

!!! info "Key Vault aus Modul 12 nutzen"
    Falls du noch den `kv-aztraining-XXXX` Key Vault aus LP2 hast, nutze diesen. Sonst erstelle einen neuen in `rg-datenbanken` (Anleitung in Modul 12).

### Schritt 1: Verbindungsstring als Secret

1. Gehe zu deinem Key Vault → **Secrets** → **+ Generate/Import**

| Feld | Wert |
|------|------|
| Upload options | `Manual` |
| Name | `sql-connection-string` |
| Value | (Verbindungsstring aus `db-aztraining` → **Connection strings** → ODBC) |

Der Wert sieht so aus:
```
Driver={ODBC Driver 18 for SQL Server};Server=tcp:sql-aztraining-XXXX.database.windows.net,1433;Database=db-aztraining;Uid=sqladmin;Pwd=DEIN_PASSWORT;Encrypt=yes;TrustServerCertificate=no;
```

Klicke **Create**.

### Schritt 2: Managed Identity für App Service aktivieren

1. Gehe zu deinem App Service → **Identity** (linkes Menü)
2. Tab **System assigned** → Status auf **On** → **Save** → **Yes**
3. Kopiere die angezeigte **Object (principal) ID** – du brauchst sie gleich

### Schritt 3: Key Vault Access Policy

1. Gehe zu deinem Key Vault → **Access policies** → **+ Create**
2. **Permissions**: aktiviere nur `Get` und `List` unter Secrets
3. **Principal**: suche nach dem Namen deines App Service → auswählen
4. **Create** → fertig

!!! tip "RBAC statt Access Policy"
    Neuere Key Vaults nutzen RBAC statt Access Policies (Modul 12 zeigt beide). Mit RBAC wäre es: Identity der App → Rolle **Key Vault Secrets User** auf dem Key Vault.

---

## App Service: Connection String als App Setting referenzieren

### Schritt 1: Secret URI kopieren

1. Key Vault → **Secrets** → `sql-connection-string` → aktuelle Version
2. Kopiere **Secret Identifier** (sieht aus wie `https://kv-aztraining-XXXX.vault.azure.net/secrets/sql-connection-string/abc123...`)

### Schritt 2: App Setting erstellen

1. App Service → **Configuration** → **+ New application setting**

| Feld | Wert |
|------|------|
| Name | `SQL_CONNECTION_STRING` |
| Value | `@Microsoft.KeyVault(SecretUri=https://kv-aztraining-XXXX.vault.azure.net/secrets/sql-connection-string/)` |

!!! info "Key Vault Reference Syntax"
    Die Syntax `@Microsoft.KeyVault(...)` ist eine spezielle App Service-Funktion. Azure löst den Wert zur Laufzeit auf – deine App liest `SQL_CONNECTION_STRING` ganz normal als Umgebungsvariable, ohne zu wissen, dass er aus Key Vault kommt.
    
    Neben `SecretUri` gibt es auch `VaultName=...,SecretName=...,SecretVersion=...`.

Klicke **Save**.

Im Configuration-Panel sollte nach kurzer Zeit ein grünes Häkchen **Key Vault Reference** erscheinen. Gelbes Ausrufezeichen = Berechtigungsproblem (Access Policy prüfen).

---

## Flask-App deployen

### Schritt 1: App-Code vorbereiten

Öffne die Cloud Shell und erstelle ein neues Verzeichnis:

```bash
mkdir ~/sql-app && cd ~/sql-app
```

Erstelle `app.py`:

```python
import os
import pyodbc
from flask import Flask, jsonify

app = Flask(__name__)


def get_connection():
    """Liest Verbindungsstring aus Umgebungsvariable (kommt aus Key Vault)."""
    conn_str = os.environ.get("SQL_CONNECTION_STRING")
    if not conn_str:
        raise RuntimeError("SQL_CONNECTION_STRING nicht gesetzt")
    return pyodbc.connect(conn_str)


@app.route("/")
def index():
    return """
    <h1>🗄️ Azure SQL Demo</h1>
    <p><a href="/mitarbeiter">Mitarbeiter anzeigen</a></p>
    """


@app.route("/mitarbeiter")
def mitarbeiter():
    try:
        conn   = get_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT id, name, abteilung, eingestellt FROM mitarbeiter ORDER BY name")
        rows = cursor.fetchall()
        conn.close()

        ergebnis = [
            {
                "id":          row[0],
                "name":        row[1],
                "abteilung":   row[2],
                "eingestellt": str(row[3])
            }
            for row in rows
        ]
        return jsonify(ergebnis)
    except Exception as e:
        return jsonify({"fehler": str(e)}), 500


if __name__ == "__main__":
    app.run(debug=True)
```

Erstelle `requirements.txt`:

```
flask==3.1.0
pyodbc==5.2.0
gunicorn==23.0.0
```

Erstelle `startup.sh`:

```bash
#!/bin/bash
pip install -r requirements.txt
gunicorn --bind=0.0.0.0:8000 app:app
```

### Schritt 2: App deployen

```bash
cd ~/sql-app
zip -r ../sql-app.zip .
az webapp deploy \
  --resource-group rg-aztraining \
  --name app-aztraining-XXXX \
  --src-path ../sql-app.zip \
  --type zip
```

### Schritt 3: App testen

```bash
# App-URL anzeigen
az webapp show \
  --resource-group rg-aztraining \
  --name app-aztraining-XXXX \
  --query "defaultHostName" \
  --output tsv
```

Öffne `https://app-aztraining-XXXX.azurewebsites.net/mitarbeiter` – du siehst die Mitarbeiter aus der SQL-Datenbank als JSON.

---

## Mit Redis-Cache erweitern

Jetzt kombinieren wir App + SQL + Redis: Datenbankabfragen 1 Minute cachen.

Erweitere `app.py`:

```python
import os
import json
import pyodbc
import redis
from flask import Flask, jsonify

app  = Flask(__name__)
r    = None   # Redis-Client, lazy initialized


def get_redis():
    global r
    if r is None:
        host = os.environ.get("REDIS_HOST", "")
        pwd  = os.environ.get("REDIS_KEY",  "")
        if host and pwd:
            r = redis.Redis(host=host, port=6380, password=pwd, ssl=True, decode_responses=True)
    return r


def get_connection():
    conn_str = os.environ.get("SQL_CONNECTION_STRING")
    return pyodbc.connect(conn_str)


@app.route("/mitarbeiter")
def mitarbeiter():
    cache_key = "mitarbeiter:alle"
    cache     = get_redis()

    # Cache prüfen
    if cache:
        cached = cache.get(cache_key)
        if cached:
            daten = json.loads(cached)
            daten.append({"_cache": "hit"})
            return jsonify(daten)

    # Datenbank abfragen
    conn   = get_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT id, name, abteilung, eingestellt FROM mitarbeiter ORDER BY name")
    ergebnis = [
        {"id": r[0], "name": r[1], "abteilung": r[2], "eingestellt": str(r[3])}
        for r in cursor.fetchall()
    ]
    conn.close()

    # In Cache schreiben (60 Sekunden)
    if cache:
        cache.setex(cache_key, 60, json.dumps(ergebnis))
        ergebnis.append({"_cache": "miss"})

    return jsonify(ergebnis)
```

Füge in App Service Configuration zwei weitere App Settings hinzu:

| Name | Wert |
|------|------|
| `REDIS_HOST` | `redis-aztraining-XXXX.redis.cache.windows.net` |
| `REDIS_KEY` | (Primary Access Key aus Redis → Access keys) |

---

## Challenge

!!! question "Challenge: Health Endpoint"
    Füge der Flask-App einen `/health`-Endpoint hinzu, der:
    
    1. Die SQL-Verbindung testet (einfaches `SELECT 1`)
    2. Die Redis-Verbindung testet (PING)
    3. JSON zurückgibt: `{"sql": "ok", "redis": "ok"}` bzw. den Fehlertext wenn etwas nicht funktioniert
    
    Dieser Pattern heißt **Health Check** und wird von App Service und Load Balancern genutzt um zu erkennen ob eine App-Instanz gesund ist.

??? success "Hinweis"
    ```python
    @app.route("/health")
    def health():
        status = {}
        
        # SQL testen
        try:
            conn = get_connection()
            conn.execute("SELECT 1")
            conn.close()
            status["sql"] = "ok"
        except Exception as e:
            status["sql"] = f"fehler: {e}"
        
        # Redis testen
        try:
            cache = get_redis()
            if cache:
                cache.ping()
                status["redis"] = "ok"
            else:
                status["redis"] = "nicht konfiguriert"
        except Exception as e:
            status["redis"] = f"fehler: {e}"
        
        code = 200 if all(v == "ok" for v in status.values()) else 503
        return jsonify(status), code
    ```
    
    App Service Health Check: **Configuration** → **Health check** → Path `/health` eintragen.

---

Weiter zu [Modul 20 – Aufräumen Lernpfad 3](modul-20-aufräumen.md) →
