# Modul 33 – Azure DevOps: Repos, Boards und Pipelines

## Lernziele

Nach diesem Modul kannst du:

- Azure DevOps als Plattform für Quellcode, Aufgabenverwaltung und CI/CD erklären
- Ein Azure DevOps-Projekt mit Repository erstellen
- Work Items und Boards für einfaches Task-Tracking nutzen
- Eine Build-Pipeline (YAML) schreiben die Code baut und testet
- Eine Release-Pipeline erstellen die auf Azure App Service deployt

---

## Hintergrund: Was ist Azure DevOps?

**On-Prem-Vergleich:** In klassischen Umgebungen nutzt man TFS (Team Foundation Server) für Quellcode-Verwaltung, Arbeitsaufgaben und Builds. **Azure DevOps** ist der Cloud-Nachfolger von TFS – die gleiche Plattform, komplett gemanagt von Microsoft, kostenlos für kleine Teams.

Azure DevOps besteht aus fünf Diensten:

| Dienst | Funktion | On-Prem-Analog |
|--------|----------|---------------|
| **Boards** | Work Items, Sprint-Planung, Kanban | Jira, VSTS |
| **Repos** | Git-Repository (privat) | GitHub, TFS |
| **Pipelines** | CI/CD: Bauen, Testen, Deployen | Jenkins, TFS Build |
| **Test Plans** | Manuelle und automatisierte Tests | TestRail |
| **Artifacts** | Package-Feed (npm, NuGet, pip) | Artifactory |

In diesem Modul: **Repos** + **Pipelines**.

!!! info "Kostenlos für kleine Teams"
    Azure DevOps ist für bis zu 5 Benutzer kostenlos – Repos, Boards und Pipelines inklusive. Danach: ca. 6 $/Benutzer/Monat.

---

## Azure DevOps-Organisation erstellen

1. Öffne [dev.azure.com](https://dev.azure.com)
2. Melde dich mit deinem Azure-Konto an
3. Klicke **Create new organization** (falls noch keine existiert)
4. Organisationsname: z.B. `mein-azure-training`
5. Klicke **+ New project**

| Feld | Wert |
|------|------|
| Project name | `aztraining-ci-cd` |
| Visibility | Private |
| Version control | Git |
| Work item process | Basic |

---

## Azure Repos: Git-Repository nutzen

### Repository klonen

1. Links im Menü: **Repos** → **Files**
2. Klicke **Clone** oben rechts
3. Kopiere die HTTPS-URL

```bash
git clone https://ORGANISATION@dev.azure.com/ORGANISATION/aztraining-ci-cd/_git/aztraining-ci-cd
cd aztraining-ci-cd
```

### Erste Datei hinzufügen

```bash
# Einfache Python-App erstellen
cat > app.py << 'EOF'
def addiere(a, b):
    return a + b

if __name__ == "__main__":
    print(addiere(3, 4))
EOF

cat > test_app.py << 'EOF'
from app import addiere

def test_addiere():
    assert addiere(2, 3) == 5
    assert addiere(-1, 1) == 0
EOF

echo "pytest" > requirements.txt

git add .
git commit -m "Erste Python-App mit Test"
git push
```

---

## Azure Boards: Work Items anlegen

1. Links im Menü: **Boards** → **Work Items**
2. Klicke **+ New Work Item** → **Issue** (oder **Task**)
3. Erstelle ein paar Aufgaben:
   - "CI-Pipeline aufsetzen"
   - "Deployment auf App Service"
   - "Tests automatisieren"

### Board-Ansicht

**Boards** → **Boards**: zeigt alle Aufgaben als Kanban-Karten (To Do / Doing / Done). Verschiebe eine Karte auf "Doing".

---

## Build-Pipeline: YAML-Pipeline erstellen

### Schritt 1: Pipeline-Datei anlegen

Erstelle `azure-pipelines.yml` im Repository-Root:

```yaml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.12"
    displayName: "Python 3.12 installieren"

  - script: |
      python -m pip install --upgrade pip
      pip install -r requirements.txt
    displayName: "Abhängigkeiten installieren"

  - script: |
      pytest test_app.py --junitxml=results.xml
    displayName: "Tests ausführen"

  - task: PublishTestResults@2
    inputs:
      testResultsFormat: "JUnit"
      testResultsFiles: "results.xml"
    displayName: "Testergebnisse veröffentlichen"
    condition: always()
```

```bash
git add azure-pipelines.yml
git commit -m "CI-Pipeline hinzugefügt"
git push
```

### Schritt 2: Pipeline in Azure DevOps verknüpfen

1. Links: **Pipelines** → **Pipelines**
2. Klicke **New pipeline**
3. **Azure Repos Git** → wähle dein Repository
4. **Existing Azure Pipelines YAML file** → `/azure-pipelines.yml`
5. Klicke **Run**

Du siehst die Pipeline laufen: Python installieren → pip install → pytest.

!!! success "Pipeline läuft durch"
    Wenn alle Schritte grün sind, siehst du unter **Tests** die Testergebnisse. Bei jedem Push auf `main` startet die Pipeline automatisch.

---

## Release-Pipeline: Auf App Service deployen

### Service Connection einrichten

Damit Azure DevOps auf deine Azure-Subscription zugreifen kann:

1. **Project Settings** (unten links) → **Service connections**
2. **New service connection** → **Azure Resource Manager**
3. **Service principal (automatic)** → deine Subscription auswählen
4. Name: `azure-subscription`

### Deployment-Schritt zur Pipeline hinzufügen

Erweitere `azure-pipelines.yml`:

```yaml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

variables:
  resourceGroup: "rg-devops"
  appServiceName: "app-devops-XXXX"  # dein App Service Name

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: "3.12"

  - script: |
      pip install -r requirements.txt
      pytest test_app.py --junitxml=results.xml
    displayName: "Tests ausführen"

  - task: PublishTestResults@2
    inputs:
      testResultsFormat: JUnit
      testResultsFiles: results.xml
    condition: always()

  - task: AzureWebApp@1
    inputs:
      azureSubscription: "azure-subscription"
      appType: webAppLinux
      appName: $(appServiceName)
      package: $(System.DefaultWorkingDirectory)
    displayName: "Auf App Service deployen"
    condition: succeeded()
```

!!! tip "Pipeline-Stages für Dev/Prod"
    In echten Projekten trennt man in **Stages**: zuerst in `dev` deployen, manuell freigeben, dann `prod`. Das verhindert versehentliche Production-Deployments.

---

## Branch-Policies: Code-Qualität erzwingen

In Azure Repos kannst du erzwingen dass kein Code direkt auf `main` gepusht wird – nur via Pull Request:

1. **Repos** → **Branches**
2. Drei Punkte bei `main` → **Branch policies**
3. Aktiviere:
   - **Require a minimum number of reviewers**: 1
   - **Check for linked work items**: aktiv
   - **Build validation**: wähle deine Pipeline

Jetzt muss jede Änderung über einen PR – und die Pipeline muss grün sein.

---

## Challenge

!!! question "Challenge: Zweiter Test und fehlgeschlagene Pipeline"
    1. Füge in `test_app.py` einen absichtlich falschen Test hinzu:
       ```python
       def test_falsch():
           assert addiere(1, 1) == 3  # falsch!
       ```
    2. Pushe auf `main`
    3. Beobachte wie die Pipeline **rot** wird
    4. Korrigiere den Test und beobachte wie die Pipeline wieder **grün** wird

??? success "Hinweis"
    Der Fehler erscheint unter **Pipelines** → deine Pipeline → der fehlgeschlagene Run → **Tests**. Du siehst genau welcher Test fehlgeschlagen ist und die Fehlermeldung.

---

Weiter zu [Modul 34 – GitHub Actions: CI/CD direkt aus GitHub](modul-34-github-actions.md) →
