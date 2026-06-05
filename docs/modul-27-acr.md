# Modul 27 – Azure Container Registry: Eigene Images speichern

## Lernziele

Nach diesem Modul kannst du:

- Eine Azure Container Registry (ACR) erstellen
- Ein lokales Docker-Image in die ACR pushen
- Images aus der ACR pullen und als Container starten
- Den Vorteil einer privaten Registry gegenüber Docker Hub erklären
- Images in der ACR verwalten und Tasks (automatische Builds) verstehen

---

## Hintergrund: Wozu eine eigene Registry?

**On-Prem-Vergleich:** In klassischen Umgebungen gibt es interne NuGet-Feeds, Artifactory oder ein eigenes npm-Repository für interne Pakete – damit nicht jeder einfach externe Pakete verwenden muss und Packages privat bleiben. Die **Azure Container Registry** ist genau das, nur für Docker-Images.

**Warum nicht einfach Docker Hub?**

| | Docker Hub (kostenlos) | Azure Container Registry |
|--|----------------------|--------------------------|
| Sichtbarkeit | Öffentlich | Privat (deine Subscription) |
| Authentifizierung | Optional | Entra ID / Admin-Passwort |
| Integration mit Azure | Manuell | Nativ (ACI, Container Apps, AKS) |
| Geo-Redundanz | Nein | Ja (Premium-Tier) |
| Kosten | 0 € (public) | ab ca. 5 €/Monat (Basic) |

---

## Resource Group vorbereiten

In Lernpfad 5 verwenden wir `rg-container`:

```bash
az group create --name rg-container --location westeurope
```

---

## Azure Container Registry erstellen

### Im Azure Portal

1. Suche nach **Container registries**
2. Klicke **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-container` |
| Registry name | `acrtraining{ZUFALLSZAHL}` (global eindeutig, nur Kleinbuchstaben+Zahlen) |
| Location | `West Europe` |
| SKU | `Basic` |

3. Klicke **Review + create** → **Create**

### Per CLI

```bash
# Eindeutigen Namen generieren
ACR_NAME="acrtraining$RANDOM"
echo "Registry-Name: $ACR_NAME"

az acr create \
  --resource-group rg-container \
  --name $ACR_NAME \
  --sku Basic
```

!!! info "SKU-Übersicht"
    | SKU | Speicher | Features | Kosten ca. |
    |-----|----------|----------|-----------|
    | Basic | 10 GB | Grundfunktionen | ~5 €/Monat |
    | Standard | 100 GB | Webhooks | ~20 €/Monat |
    | Premium | 500 GB | Geo-Replikation, Private Endpoints | ~55 €/Monat |
    
    Für das Training reicht **Basic** vollständig aus.

---

## Image bauen und in ACR pushen

### Methode 1: Lokal bauen, dann pushen

```bash
# In der ACR einloggen (Docker Desktop muss laufen)
az acr login --name $ACR_NAME

# Vollständiger Image-Name für ACR
ACR_IMAGE="$ACR_NAME.azurecr.io/mein-webserver:1.0"

# Lokales Image taggen
docker tag mein-webserver:1.0 $ACR_IMAGE

# In ACR pushen
docker push $ACR_IMAGE
```

### Methode 2: Direkt in ACR bauen (ohne lokales Docker)

Der **ACR Quick Task** baut das Image direkt in der Cloud – kein Docker Desktop nötig:

```bash
# Im Verzeichnis mit Dockerfile:
az acr build \
  --registry $ACR_NAME \
  --image mein-webserver:1.0 \
  .
```

Das ist besonders nützlich in CI/CD-Pipelines oder wenn Docker Desktop nicht installiert ist.

---

## Images in der ACR anzeigen

### Im Portal

1. Öffne die Container Registry im Azure Portal
2. Links: **Repositories** – dort siehst du alle gespeicherten Images
3. Klicke auf ein Repository um alle Tags (Versionen) zu sehen

### Per CLI

```bash
# Alle Repositories auflisten
az acr repository list --name $ACR_NAME --output table

# Tags eines Repositories anzeigen
az acr repository show-tags \
  --name $ACR_NAME \
  --repository mein-webserver \
  --output table
```

---

## Image aus ACR pullen und als Container starten

```bash
# Image lokal herunterladen
docker pull $ACR_NAME.azurecr.io/mein-webserver:1.0

# Container starten
docker run -d -p 8080:8080 $ACR_NAME.azurecr.io/mein-webserver:1.0
```

---

## ACR-Zugangsdaten (für spätere Module)

Andere Azure-Dienste (ACI, Container Apps, AKS) brauchen Zugangsdaten um aus der ACR zu pullen. Es gibt zwei Wege:

**Weg 1 – Admin-Benutzer (einfach, nicht empfohlen für Produktion):**

```bash
# Admin aktivieren
az acr update --name $ACR_NAME --admin-enabled true

# Zugangsdaten anzeigen
az acr credential show --name $ACR_NAME
```

**Weg 2 – Managed Identity (empfohlen, Modul 22-Prinzip):**

Ein ACI oder Container Apps bekommt eine Managed Identity, die die Rolle `AcrPull` auf der ACR hat.

```bash
# AcrPull-Rolle der Managed Identity zuweisen
ACR_ID=$(az acr show --name $ACR_NAME --query id -o tsv)
az role assignment create \
  --assignee <MANAGED-IDENTITY-CLIENT-ID> \
  --role AcrPull \
  --scope $ACR_ID
```

---

## ACR Tasks: Automatischer Image-Build bei Git-Push

ACR Tasks bauen Images automatisch neu, wenn sich Code ändert – ein einfaches CI/CD:

```bash
az acr task create \
  --registry $ACR_NAME \
  --name build-on-commit \
  --image mein-webserver:{{.Run.ID}} \
  --context https://github.com/DEIN-USER/DEIN-REPO.git \
  --file Dockerfile \
  --git-access-token <GITHUB-TOKEN>
```

!!! tip "ACR Tasks vs. GitHub Actions"
    ACR Tasks sind einfach einzurichten für reine Image-Builds. Für komplexe CI/CD-Pipelines mit Tests und Deployments ist GitHub Actions (Modul 34) mächtiger.

---

## Challenge

!!! question "Challenge: Flask-App in ACR pushen"
    Nimm das Flask-Image aus Modul 26 (oder baue es neu) und:
    
    1. Pushe das Image in deine ACR als `flask-app:1.0`
    2. Lösche das lokale Image: `docker rmi flask-app:1.0`
    3. Pulle es aus der ACR und starte einen Container davon
    4. Überprüfe dass [http://localhost:5000](http://localhost:5000) immer noch antwortet

??? success "Hinweis"
    ```bash
    # Image für ACR taggen und pushen
    docker tag flask-app:1.0 $ACR_NAME.azurecr.io/flask-app:1.0
    docker push $ACR_NAME.azurecr.io/flask-app:1.0
    
    # Lokales Image löschen
    docker rmi flask-app:1.0
    
    # Aus ACR pullen und starten
    docker pull $ACR_NAME.azurecr.io/flask-app:1.0
    docker run -d -p 5000:5000 $ACR_NAME.azurecr.io/flask-app:1.0
    ```

---

Weiter zu [Modul 28 – Azure Container Instances](modul-28-aci.md) →
