# Modul 47 – Abschlussprojekt 3: Container-App mit CI/CD-Pipeline

## Projektübersicht

Im letzten Abschlussprojekt kombinierst du **Container** (Lernpfad 5) und **DevOps** (Lernpfad 6): Eine containerisierte Web-App wird über eine vollautomatische GitHub Actions CI/CD-Pipeline gebaut, getestet, in ACR gepusht und auf Azure Container Apps deployt.

**Architektur:**

```
Entwickler: git push → GitHub Repository
                │
                ▼
        GitHub Actions CI/CD-Pipeline
        ├── Tests ausführen (pytest)
        ├── Docker Image bauen
        ├── Image in ACR pushen
        └── Container App aktualisieren
                │
                ▼
        Azure Container Registry (ACR)
                │
                ▼
        Azure Container Apps (öffentlich erreichbar)
                │
                ▼
        Benutzer via HTTPS
```

**Was du baust:** Eine containerisierte Quote-App (zufällige Zitate) mit automatischer CI/CD-Pipeline. Jeder Push auf `main` deployt automatisch eine neue Version.

!!! info "Voraussetzungen"
    Lernpfad 5 (Modul 26 – Docker, Modul 27 – ACR, Modul 29 – Container Apps) und Lernpfad 6 (Modul 34 – GitHub Actions) sollten abgeschlossen sein.

---

## Schritt 1: Infrastruktur erstellen

```bash
az group create --name rg-projekt3 --location westeurope

SUFFIX=$RANDOM

# Azure Container Registry
az acr create \
  --resource-group rg-projekt3 \
  --name acrprojekt3${SUFFIX} \
  --sku Basic \
  --admin-enabled true

ACR_NAME=$(az acr list --resource-group rg-projekt3 --query "[0].name" -o tsv)

# Container Apps Environment
az containerapp env create \
  --name env-projekt3 \
  --resource-group rg-projekt3 \
  --location westeurope

# Initiale Container App (mit Placeholder-Image)
az containerapp create \
  --name quote-app \
  --resource-group rg-projekt3 \
  --environment env-projekt3 \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 5000 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 3 \
  --cpu 0.25 \
  --memory 0.5Gi

echo "ACR Name: $ACR_NAME"
az containerapp show --name quote-app --resource-group rg-projekt3 --query properties.configuration.ingress.fqdn -o tsv
```

---

## Schritt 2: App entwickeln

Erstelle ein GitHub-Repository `quote-app` und füge folgende Dateien hinzu:

**`app.py`:**
```python
import random
from flask import Flask, jsonify, render_template_string

app = Flask(__name__)

QUOTES = [
    {"text": "The cloud is just someone else's computer.", "author": "Someone on the internet"},
    {"text": "Move fast and break things.", "author": "Mark Zuckerberg"},
    {"text": "Any application that can be written in JavaScript will eventually be written in JavaScript.", "author": "Atwood's Law"},
    {"text": "There are only two hard things in Computer Science: cache invalidation and naming things.", "author": "Phil Karlton"},
    {"text": "The best way to predict the future is to implement it.", "author": "Alan Kay"},
    {"text": "Simplicity is the soul of efficiency.", "author": "Austin Freeman"},
    {"text": "Code is like humor. When you have to explain it, it's bad.", "author": "Cory House"},
]

HTML = """
<!DOCTYPE html>
<html>
<head>
  <title>Quote App</title>
  <style>
    body { font-family: Arial; display: flex; justify-content: center; align-items: center;
           min-height: 100vh; margin: 0; background: linear-gradient(135deg, #0078d4, #00b4d8); }
    .card { background: white; border-radius: 12px; padding: 40px; max-width: 500px;
            text-align: center; box-shadow: 0 4px 20px rgba(0,0,0,0.2); }
    blockquote { font-size: 1.3em; font-style: italic; color: #333; margin: 0 0 20px 0; }
    cite { color: #0078d4; font-weight: bold; }
    button { margin-top: 20px; padding: 10px 24px; background: #0078d4; color: white;
             border: none; border-radius: 6px; cursor: pointer; font-size: 1em; }
    button:hover { background: #005a9e; }
  </style>
</head>
<body>
  <div class="card">
    <h2>💬 Quote of the Moment</h2>
    <blockquote>"{{ quote.text }}"</blockquote>
    <cite>— {{ quote.author }}</cite>
    <br>
    <a href="/"><button>Neues Zitat</button></a>
  </div>
</body>
</html>
"""

@app.route("/")
def index():
    quote = random.choice(QUOTES)
    return render_template_string(HTML, quote=quote)

@app.route("/api/quote")
def api_quote():
    return jsonify(random.choice(QUOTES))

@app.route("/health")
def health():
    return jsonify({"status": "ok", "version": "1.0"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`:**
```
flask==3.0.3
gunicorn==22.0.0
pytest==8.2.2
```

**`Dockerfile`:**
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

**`test_app.py`:**
```python
import pytest
from app import app, QUOTES

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as client:
        yield client

def test_index_returns_200(client):
    response = client.get("/")
    assert response.status_code == 200

def test_api_quote_returns_json(client):
    response = client.get("/api/quote")
    assert response.status_code == 200
    data = response.get_json()
    assert "text" in data
    assert "author" in data

def test_health_endpoint(client):
    response = client.get("/health")
    assert response.status_code == 200
    assert response.get_json()["status"] == "ok"

def test_quotes_not_empty():
    assert len(QUOTES) > 0
    for q in QUOTES:
        assert "text" in q
        assert "author" in q
```

---

## Schritt 3: GitHub Actions Pipeline

Erstelle `.github/workflows/ci-cd.yml` im Repository:

```yaml
name: CI/CD – Quote App

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  ACR_NAME: ${{ secrets.ACR_NAME }}
  RESOURCE_GROUP: rg-projekt3
  CONTAINER_APP: quote-app
  IMAGE_NAME: quote-app

jobs:
  test:
    name: Tests ausführen
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Python einrichten
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Abhängigkeiten installieren
        run: pip install -r requirements.txt

      - name: Tests ausführen
        run: pytest test_app.py -v

  build-and-deploy:
    name: Build, Push & Deploy
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Bei Azure anmelden
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Bei ACR anmelden
        run: az acr login --name $ACR_NAME

      - name: Docker Image bauen und pushen
        run: |
          IMAGE_TAG="${ACR_NAME}.azurecr.io/${IMAGE_NAME}:${{ github.sha }}"
          IMAGE_LATEST="${ACR_NAME}.azurecr.io/${IMAGE_NAME}:latest"

          docker build -t $IMAGE_TAG -t $IMAGE_LATEST .
          docker push $IMAGE_TAG
          docker push $IMAGE_LATEST

          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV

      - name: Container App aktualisieren
        run: |
          az containerapp update \
            --name $CONTAINER_APP \
            --resource-group $RESOURCE_GROUP \
            --image $IMAGE_TAG

      - name: App-URL ausgeben
        run: |
          URL=$(az containerapp show \
            --name $CONTAINER_APP \
            --resource-group $RESOURCE_GROUP \
            --query properties.configuration.ingress.fqdn -o tsv)
          echo "✅ App live unter: https://$URL"
```

---

## Schritt 4: GitHub Secrets einrichten

```bash
# 1. Service Principal für GitHub Actions erstellen
SP_JSON=$(az ad sp create-for-rbac \
  --name "github-quote-app" \
  --role Contributor \
  --scopes /subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-projekt3 \
  --json-auth)

echo "AZURE_CREDENTIALS Secret:"
echo $SP_JSON

# 2. ACR Push-Berechtigung dem SP geben
SP_ID=$(echo $SP_JSON | python3 -c "import sys,json; print(json.load(sys.stdin)['clientId'])")
ACR_ID=$(az acr show --name $ACR_NAME --resource-group rg-projekt3 --query id -o tsv)

az role assignment create \
  --assignee $SP_ID \
  --role "AcrPush" \
  --scope $ACR_ID

echo ""
echo "ACR_NAME Secret: $ACR_NAME"
```

Füge in GitHub → Settings → Secrets → Actions folgende Secrets ein:

| Secret | Wert |
|--------|------|
| `AZURE_CREDENTIALS` | JSON-Output von `az ad sp create-for-rbac --json-auth` |
| `ACR_NAME` | Name deiner ACR (z. B. `acrprojekt312345`) |

---

## Schritt 5: Erster Deploy

```bash
# Git initialisieren und pushen
git init
git add .
git commit -m "Initial commit: Quote App"
git remote add origin https://github.com/DEIN-USER/quote-app.git
git push -u origin main
```

Öffne GitHub → Actions – du siehst die Pipeline laufen:
1. Tests ✅
2. Docker Build ✅
3. ACR Push ✅
4. Container App Update ✅

```bash
# App-URL abrufen
az containerapp show \
  --name quote-app \
  --resource-group rg-projekt3 \
  --query properties.configuration.ingress.fqdn -o tsv
```

---

## Schritt 6: Update deployen

Teste den vollständigen CI/CD-Zyklus: Füge ein neues Zitat in `app.py` hinzu und pushe:

```bash
# Neues Zitat in QUOTES-Liste hinzufügen, dann:
git add app.py
git commit -m "feat: neues Zitat hinzugefügt"
git push
```

Innerhalb von ~2 Minuten ist die neue Version live – ohne manuellen Eingriff.

---

## Aufräumen

```bash
# Service Principal löschen
az ad sp delete --id $SP_ID 2>/dev/null || true

# Resource Group löschen
az group delete --name rg-projekt3 --yes --no-wait
```

---

## Challenge

!!! question "Challenge: Blue-Green Deployment"
    Erweitere die Pipeline um ein Blue-Green Deployment:
    
    1. Erstelle eine zweite Container App `quote-app-staging`
    2. Die Pipeline deployt zuerst auf Staging
    3. Führe dort automatische Smoke-Tests durch (`curl /health`)
    4. Erst bei Erfolg wird die Produktions-App aktualisiert
    
    Tipp: Nutze `environment` in GitHub Actions für den Staging-Approval-Step.

??? success "Hinweis"
    ```yaml
    deploy-staging:
      environment: staging
      steps:
        - name: Auf Staging deployen
          run: az containerapp update --name quote-app-staging ...
        - name: Smoke Test
          run: |
            URL=$(az containerapp show --name quote-app-staging ... --query ...fqdn -o tsv)
            curl -f "https://$URL/health" || exit 1

    deploy-production:
      needs: deploy-staging
      environment: production  # Kann manuelle Freigabe erfordern
      steps:
        - name: Auf Produktion deployen
          run: az containerapp update --name quote-app ...
    ```

---

## 🎉 Herzlichen Glückwunsch!

Du hast alle drei Abschlussprojekte abgeschlossen und damit das gesamte Azure Einstiegstraining beendet!

**Was du über 47 Module gelernt hast:**

| Lernpfad | Themen |
|----------|--------|
| LP 1 – Grundlagen | VM, Storage, KI, App Service, Functions |
| LP 2 – Netzwerk | VNet, NSG, Bastion, Load Balancer, Key Vault, Private Endpoints |
| LP 3 – Datenbanken | SQL, Cosmos DB, PostgreSQL, Redis |
| LP 4 – Identity | Entra ID, Managed Identity, RBAC, Conditional Access |
| LP 5 – Container | Docker, ACR, ACI, Container Apps, AKS |
| LP 6 – DevOps & IaC | Azure DevOps, GitHub Actions, ARM, Bicep, Terraform, Policy |
| LP 7 – Monitoring | Log Analytics, App Insights, Defender for Cloud, Sentinel |
| LP 8 – Projekte | 3-Tier-App, Event-Driven, Container + CI/CD |

!!! tip "Nächste Schritte"
    - [AZ-900: Azure Fundamentals](https://learn.microsoft.com/de-de/certifications/azure-fundamentals/) – Zertifizierung für Azure-Grundlagen
    - [AZ-104: Azure Administrator](https://learn.microsoft.com/de-de/certifications/azure-administrator/) – Vertiefung für Administratoren
    - [AZ-204: Azure Developer](https://learn.microsoft.com/de-de/certifications/azure-developer/) – Vertiefung für Entwickler
    - [Microsoft Learn](https://learn.microsoft.com/de-de/training/) – Kostenlose offizielle Lernpfade
