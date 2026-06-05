# Modul 34 – GitHub Actions: CI/CD direkt aus GitHub nach Azure

## Lernziele

Nach diesem Modul kannst du:

- GitHub Actions als CI/CD-Plattform direkt in GitHub erklären
- Einen Workflow (`.github/workflows/`) schreiben der bei Push automatisch ausgeführt wird
- Eine Python-App mit GitHub Actions testen und auf Azure App Service deployen
- Secrets in GitHub hinterlegen und sicher in Workflows verwenden
- Den Unterschied zwischen GitHub Actions und Azure DevOps Pipelines einschätzen

---

## Hintergrund: CI/CD direkt im Repository

**On-Prem-Vergleich:** Früher musste man einen eigenen Jenkins-Server aufsetzen, pflegen und skalieren. **GitHub Actions** ist CI/CD direkt eingebaut in GitHub – kein separater Server, kein Setup, 2000 kostenlose Minuten pro Monat für public und private Repos.

**GitHub Actions vs. Azure DevOps Pipelines:**

| | GitHub Actions | Azure DevOps Pipelines |
|--|---------------|----------------------|
| Wo | Im GitHub-Repository | Separates Tool (dev.azure.com) |
| Syntax | YAML (`.github/workflows/`) | YAML (`azure-pipelines.yml`) |
| Integration | Nativ mit GitHub (PRs, Issues) | Nativ mit Azure Boards/Repos |
| Marketplace | Tausende Actions | Weniger Extensions |
| Kosten | 2000 Min/Monat gratis | 1 Parallel Job gratis |
| Empfehlung | Code liegt auf GitHub | Code liegt auf Azure Repos |

---

## Vorbereitung: GitHub-Repository

Falls du noch kein GitHub-Repository hast:

1. Gehe zu [github.com](https://github.com) → **New repository**
2. Name: `azure-training-app`
3. Visibility: Public oder Private
4. Klicke **Create repository**

Klone es lokal und erstelle die App:

```bash
git clone https://github.com/DEIN-USER/azure-training-app
cd azure-training-app

# App-Dateien aus Modul 33 verwenden oder neu anlegen
cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "Hallo von GitHub Actions deployt!"

@app.route("/health")
def health():
    return {"status": "ok"}
EOF

cat > test_app.py << 'EOF'
import pytest
from app import app

@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c

def test_index(client):
    response = client.get("/")
    assert response.status_code == 200

def test_health(client):
    response = client.get("/health")
    assert response.status_code == 200
EOF

echo -e "flask\npytest" > requirements.txt

git add .
git commit -m "Flask-App mit Tests"
git push
```

---

## Ersten Workflow schreiben

Erstelle den Ordner und die Datei:

```bash
mkdir -p .github/workflows
```

Erstelle `.github/workflows/ci.yml`:

```yaml
name: CI – Testen

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Code auschecken
        uses: actions/checkout@v4

      - name: Python 3.12 installieren
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Abhängigkeiten installieren
        run: |
          pip install -r requirements.txt

      - name: Tests ausführen
        run: pytest test_app.py -v
```

```bash
git add .github/
git commit -m "GitHub Actions CI-Workflow"
git push
```

Öffne das Repository auf GitHub → Tab **Actions** → du siehst den Workflow laufen.

---

## Deployment auf Azure App Service

### Schritt 1: App Service vorbereiten

```bash
az group create --name rg-devops --location westeurope

az appservice plan create \
  --name plan-devops \
  --resource-group rg-devops \
  --sku F1 \
  --is-linux

az webapp create \
  --name app-devops-$RANDOM \
  --resource-group rg-devops \
  --plan plan-devops \
  --runtime "PYTHON:3.12"

# App-Namen merken
az webapp list --resource-group rg-devops --query "[0].name" -o tsv
```

### Schritt 2: Publish Profile als GitHub Secret hinterlegen

```bash
# Publish Profile herunterladen
az webapp deployment list-publishing-profiles \
  --name DEIN-APP-NAME \
  --resource-group rg-devops \
  --xml
```

Kopiere die komplette XML-Ausgabe.

In GitHub:
1. Repository → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret**
3. Name: `AZURE_WEBAPP_PUBLISH_PROFILE`
4. Value: die XML einfügen

Zweites Secret:
- Name: `AZURE_WEBAPP_NAME`
- Value: dein App-Name (z.B. `app-devops-12345`)

### Schritt 3: Deploy-Workflow erweitern

Erstelle `.github/workflows/deploy.yml`:

```yaml
name: CI/CD – Testen und Deployen

on:
  push:
    branches: [ main ]

jobs:
  test-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Python 3.12 installieren
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Abhängigkeiten installieren
        run: pip install -r requirements.txt

      - name: Tests ausführen
        run: pytest test_app.py -v

      - name: Auf Azure App Service deployen
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: .
```

```bash
git add .github/workflows/deploy.yml
git commit -m "CD-Workflow für Azure App Service"
git push
```

Der Workflow testet zuerst – wenn alle Tests grün sind, deployt er automatisch.

---

## Workflows per Matrix testen

Wenn du mehrere Python-Versionen testen willst:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest test_app.py -v
```

Beide Python-Versionen werden parallel getestet – doppelter Schutz.

---

## Environment Variables und Secrets

```yaml
steps:
  - name: App mit Config starten
    env:
      APP_ENV: production
      # Secrets niemals direkt in YAML schreiben!
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    run: python app.py
```

!!! warning "Niemals Secrets in YAML hart kodieren"
    GitHub Actions maskiert automatisch alle Secrets in Logs – du siehst `***` statt dem echten Wert. Aber: Secrets dürfen nur über `secrets.NAME` eingebunden werden, nie als Klartext.

---

## Workflow-Status-Badge

Zeige den Pipeline-Status im README:

```markdown
![CI/CD](https://github.com/DEIN-USER/azure-training-app/actions/workflows/deploy.yml/badge.svg)
```

Das ergibt einen grünen oder roten Badge der den aktuellen Pipeline-Status zeigt.

---

## Challenge

!!! question "Challenge: Manueller Trigger und Umgebung"
    Erweitere den Workflow so dass er:
    
    1. Auch **manuell** gestartet werden kann (`workflow_dispatch`)
    2. Eine Umgebungsvariable `STAGE=staging` setzt
    3. Diese Variable in einem `echo`-Schritt ausgibt

??? success "Hinweis"
    ```yaml
    on:
      push:
        branches: [ main ]
      workflow_dispatch:   # manueller Trigger
    
    jobs:
      deploy:
        runs-on: ubuntu-latest
        env:
          STAGE: staging
        steps:
          - run: echo "Deploye in Umgebung: $STAGE"
    ```
    
    Manuell starten: GitHub → **Actions** → Workflow → **Run workflow**

---

Weiter zu [Modul 35 – ARM Templates: Infrastruktur als JSON](modul-35-arm.md) →
