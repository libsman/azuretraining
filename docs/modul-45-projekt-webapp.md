# Modul 45 – Abschlussprojekt 1: Dreischichtige Web-App

## Projektübersicht

In diesem Abschlussprojekt baust du eine vollständige **dreischichtige Web-Applikation** die die Konzepte aus Lernpfad 1 (App Service, Functions), Lernpfad 2 (Load Balancer, Key Vault) und Lernpfad 3 (Azure SQL) kombiniert.

**Architektur:**

```
Internet
    │
    ▼
Azure Load Balancer (öffentliche IP)
    │
    ▼
App Service – Frontend (Python Flask, HTML/JS)
    │  sendet API-Requests
    ▼
App Service – API-Backend (Python REST API)
    │  liest/schreibt
    ▼
Azure SQL Database
    │
Key Vault (Verbindungsstring für das Backend)
```

**Was du baust:** Eine einfache Task-Manager-App. Benutzer können Aufgaben erstellen, anzeigen und als erledigt markieren.

!!! info "Voraussetzungen"
    Du solltest Lernpfad 1 (Modul 5 – App Service) und Lernpfad 3 (Modul 15 – Azure SQL) abgeschlossen haben. Lernpfad 2 (Modul 12 – Key Vault, Modul 11 – Load Balancer) ist hilfreich aber nicht zwingend.

---

## Schritt 1: Infrastruktur erstellen

```bash
# Resource Group
az group create --name rg-projekt1 --location westeurope

# Azure SQL Server + Datenbank
az sql server create \
  --name sql-projekt1-$RANDOM \
  --resource-group rg-projekt1 \
  --location westeurope \
  --admin-user sqladmin \
  --admin-password "AzureTraining2024!"

SQL_SERVER=$(az sql server list --resource-group rg-projekt1 --query "[0].name" -o tsv)

az sql db create \
  --resource-group rg-projekt1 \
  --server $SQL_SERVER \
  --name taskdb \
  --service-objective Basic

# Firewall-Regel: Azure-Dienste erlauben
az sql server firewall-rule create \
  --resource-group rg-projekt1 \
  --server $SQL_SERVER \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Key Vault
az keyvault create \
  --name kv-projekt1-$RANDOM \
  --resource-group rg-projekt1 \
  --location westeurope \
  --enable-rbac-authorization true

KV_NAME=$(az keyvault list --resource-group rg-projekt1 --query "[0].name" -o tsv)

# Verbindungsstring im Key Vault speichern
SQL_CONN="Server=tcp:${SQL_SERVER}.database.windows.net,1433;Database=taskdb;User ID=sqladmin;Password=AzureTraining2024!;Encrypt=True"
az keyvault secret set \
  --vault-name $KV_NAME \
  --name "sql-connection-string" \
  --value "$SQL_CONN"
```

---

## Schritt 2: API-Backend deployen

Erstelle lokal den Ordner `api-backend/` mit folgenden Dateien:

**`api-backend/app.py`:**
```python
import os
import pyodbc
from flask import Flask, jsonify, request
from azure.identity import ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

app = Flask(__name__)

def get_db_connection():
    kv_uri = os.environ["KEY_VAULT_URI"]
    credential = ManagedIdentityCredential()
    client = SecretClient(vault_url=kv_uri, credential=credential)
    conn_str = client.get_secret("sql-connection-string").value
    return pyodbc.connect(conn_str)

def init_db():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("""
        IF NOT EXISTS (SELECT * FROM sysobjects WHERE name='tasks' AND xtype='U')
        CREATE TABLE tasks (
            id INT IDENTITY(1,1) PRIMARY KEY,
            title NVARCHAR(255) NOT NULL,
            done BIT DEFAULT 0,
            created_at DATETIME DEFAULT GETDATE()
        )
    """)
    conn.commit()
    conn.close()

@app.route("/api/tasks", methods=["GET"])
def get_tasks():
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("SELECT id, title, done, created_at FROM tasks ORDER BY created_at DESC")
    tasks = [{"id": r[0], "title": r[1], "done": bool(r[2]), "created_at": str(r[3])} for r in cursor.fetchall()]
    conn.close()
    return jsonify(tasks)

@app.route("/api/tasks", methods=["POST"])
def create_task():
    data = request.get_json()
    title = data.get("title", "").strip()
    if not title:
        return jsonify({"error": "title required"}), 400
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("INSERT INTO tasks (title) VALUES (?)", title)
    conn.commit()
    conn.close()
    return jsonify({"status": "created"}), 201

@app.route("/api/tasks/<int:task_id>/done", methods=["PUT"])
def mark_done(task_id):
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute("UPDATE tasks SET done=1 WHERE id=?", task_id)
    conn.commit()
    conn.close()
    return jsonify({"status": "updated"})

@app.route("/health")
def health():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    init_db()
    app.run(host="0.0.0.0", port=8000)
```

**`api-backend/requirements.txt`:**
```
flask==3.0.3
pyodbc==5.1.0
azure-identity==1.17.1
azure-keyvault-secrets==4.8.0
gunicorn==22.0.0
```

**`api-backend/startup.sh`:**
```bash
python -c "from app import init_db; init_db()"
gunicorn --bind=0.0.0.0:8000 app:app
```

### App Service für API erstellen und deployen

```bash
# App Service Plan (Linux, Free Tier reicht nicht für ODBC – mindestens B1)
az appservice plan create \
  --name plan-projekt1 \
  --resource-group rg-projekt1 \
  --is-linux \
  --sku B1

# API-Backend App Service
az webapp create \
  --resource-group rg-projekt1 \
  --plan plan-projekt1 \
  --name api-projekt1-$RANDOM \
  --runtime "PYTHON:3.12"

API_APP=$(az webapp list --resource-group rg-projekt1 --query "[?contains(name,'api-')].name" -o tsv | head -1)

# Managed Identity aktivieren
az webapp identity assign \
  --resource-group rg-projekt1 \
  --name $API_APP

API_PRINCIPAL=$(az webapp identity show --resource-group rg-projekt1 --name $API_APP --query principalId -o tsv)

# RBAC: App Service darf Key Vault Secrets lesen
KV_ID=$(az keyvault show --name $KV_NAME --resource-group rg-projekt1 --query id -o tsv)
az role assignment create \
  --assignee $API_PRINCIPAL \
  --role "Key Vault Secrets User" \
  --scope $KV_ID

# Key Vault URI als App-Setting setzen
az webapp config appsettings set \
  --resource-group rg-projekt1 \
  --name $API_APP \
  --settings KEY_VAULT_URI="https://${KV_NAME}.vault.azure.net/"

# Code deployen
cd api-backend
zip -r api.zip .
az webapp deploy \
  --resource-group rg-projekt1 \
  --name $API_APP \
  --src-path api.zip \
  --type zip
```

---

## Schritt 3: Frontend deployen

Erstelle `frontend/app.py`:

```python
import os
import requests
from flask import Flask, render_template_string, request, redirect

app = Flask(__name__)
API_URL = os.environ.get("API_URL", "http://localhost:8000")

HTML = """
<!DOCTYPE html>
<html>
<head>
  <title>Task Manager</title>
  <style>
    body { font-family: Arial; max-width: 600px; margin: 40px auto; padding: 0 20px; }
    .task { padding: 10px; border-bottom: 1px solid #eee; }
    .done { text-decoration: line-through; color: #999; }
    input[type=text] { width: 70%; padding: 8px; }
    button { padding: 8px 16px; background: #0078d4; color: white; border: none; cursor: pointer; }
  </style>
</head>
<body>
  <h1>📋 Task Manager</h1>
  <form method="POST" action="/tasks">
    <input type="text" name="title" placeholder="Neue Aufgabe..." required>
    <button type="submit">Hinzufügen</button>
  </form>
  <div style="margin-top:20px">
    {% for task in tasks %}
    <div class="task">
      {% if task.done %}
        <span class="done">✅ {{ task.title }}</span>
      {% else %}
        <span>{{ task.title }}</span>
        <form method="POST" action="/tasks/{{ task.id }}/done" style="display:inline">
          <button type="submit" style="font-size:12px;padding:4px 8px">✓ Erledigt</button>
        </form>
      {% endif %}
    </div>
    {% endfor %}
  </div>
</body>
</html>
"""

@app.route("/")
def index():
    resp = requests.get(f"{API_URL}/api/tasks", timeout=5)
    tasks = resp.json() if resp.ok else []
    return render_template_string(HTML, tasks=tasks)

@app.route("/tasks", methods=["POST"])
def create():
    title = request.form.get("title", "").strip()
    if title:
        requests.post(f"{API_URL}/api/tasks", json={"title": title}, timeout=5)
    return redirect("/")

@app.route("/tasks/<int:task_id>/done", methods=["POST"])
def mark_done(task_id):
    requests.put(f"{API_URL}/api/tasks/{task_id}/done", timeout=5)
    return redirect("/")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

`frontend/requirements.txt`:
```
flask==3.0.3
requests==2.32.3
gunicorn==22.0.0
```

```bash
# Frontend App Service
az webapp create \
  --resource-group rg-projekt1 \
  --plan plan-projekt1 \
  --name frontend-projekt1-$RANDOM \
  --runtime "PYTHON:3.12"

FRONTEND_APP=$(az webapp list --resource-group rg-projekt1 --query "[?contains(name,'frontend-')].name" -o tsv | head -1)

# API-URL als Setting setzen
az webapp config appsettings set \
  --resource-group rg-projekt1 \
  --name $FRONTEND_APP \
  --settings API_URL="https://${API_APP}.azurewebsites.net"

# Code deployen
cd ../frontend
zip -r frontend.zip .
az webapp deploy \
  --resource-group rg-projekt1 \
  --name $FRONTEND_APP \
  --src-path frontend.zip \
  --type zip

echo "Frontend: https://${FRONTEND_APP}.azurewebsites.net"
```

---

## Schritt 4: Testen

```bash
# API direkt testen
API_URL="https://${API_APP}.azurewebsites.net"

# Task erstellen
curl -X POST "$API_URL/api/tasks" \
  -H "Content-Type: application/json" \
  -d '{"title": "Azure Training abschließen"}'

# Tasks abrufen
curl "$API_URL/api/tasks"

# Frontend aufrufen
echo "Öffne im Browser: https://${FRONTEND_APP}.azurewebsites.net"
```

---

## Schritt 5: Monitoring hinzufügen (optional)

```bash
# Application Insights verbinden (aus Modul 41)
LAW_ID=$(az monitor log-analytics workspace show \
  --workspace-name law-training --resource-group rg-monitoring \
  --query id -o tsv 2>/dev/null)

if [ -n "$LAW_ID" ]; then
  az monitor app-insights component create \
    --app ai-projekt1 \
    --resource-group rg-projekt1 \
    --location westeurope \
    --kind web \
    --workspace $LAW_ID

  AI_CONN=$(az monitor app-insights component show \
    --app ai-projekt1 --resource-group rg-projekt1 \
    --query connectionString -o tsv)

  az webapp config appsettings set \
    --resource-group rg-projekt1 --name $API_APP \
    --settings APPLICATIONINSIGHTS_CONNECTION_STRING="$AI_CONN"
fi
```

---

## Aufräumen

```bash
az group delete --name rg-projekt1 --yes --no-wait
```

---

## Challenge

!!! question "Challenge: Authentifizierung hinzufügen"
    Erweitere die App um eine einfache Authentifizierung:
    
    1. Aktiviere **Easy Auth** (App Service Authentication) auf dem Frontend App Service
    2. Konfiguriere Microsoft als Identity Provider (Entra ID)
    3. Nur angemeldete Benutzer können Aufgaben sehen und erstellen

??? success "Hinweis"
    Im Portal: Frontend App Service → **Authentication** → **Add identity provider** → **Microsoft** → Tenant auswählen → **Add**.
    
    Oder per CLI:
    ```bash
    az webapp auth microsoft update \
      --resource-group rg-projekt1 \
      --name $FRONTEND_APP \
      --client-id DEINE-APP-REGISTRATION-ID \
      --issuer "https://login.microsoftonline.com/TENANT-ID/v2.0"
    ```

---

Weiter zu [Modul 46 – Abschlussprojekt 2: Event-Driven Architektur](modul-46-projekt-eventdriven.md) →
