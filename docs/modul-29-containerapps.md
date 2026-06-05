# Modul 29 – Azure Container Apps: Serverless Container mit Autoscaling

## Lernziele

Nach diesem Modul kannst du:

- Azure Container Apps (ACA) als serverlose Container-Plattform erklären
- Den Unterschied zwischen Container Apps und ACI sowie AKS beschreiben
- Eine Container App aus einem ACR-Image erstellen
- Autoscaling konfigurieren (HTTP-basiert, KEDA-Rules)
- Revisionen und Traffic-Splitting für Canary-Deployments verstehen
- Dapr (Distributed Application Runtime) als optionale Integration kennen

---

## Hintergrund: Die Lücke zwischen ACI und AKS

**On-Prem-Vergleich:** Du kennst IIS-Websites die mit Load Balancer auf mehrere Server verteilt werden. In der Cloud gibt es dafür ein modernes Äquivalent: Container Apps nehmen dir die gesamte Kubernetes-Infrastruktur ab und bieten trotzdem Skalierung, Traffic-Steuerung und Service-Discovery.

**Das Container-Spektrum:**

```
Einfach ◄─────────────────────────────► Komplex
   │                 │                    │
  ACI            Container Apps          AKS
(Einzel-        (Managed Plattform      (Volles
 Container,      mit Autoscale,          Kubernetes,
 kein Scale)     kein K8s-Wissen)        volle Kontrolle)
```

**Wann Container Apps?**

- Web-Apps und APIs die automatisch skalieren sollen
- Microservices die miteinander kommunizieren
- Event-getriebene Container (KEDA: Skalierung nach Queue-Länge, HTTP-Traffic, etc.)
- Du willst kein Kubernetes verwalten, aber mehr als ACI

---

## Container Apps Environment erstellen

Container Apps laufen in einer **Environment** (Laufzeitumgebung, geteilt über mehrere Container Apps):

### Im Portal

1. Suche nach **Container Apps** → **+ Create**
2. Im Schritt **Basics**: Resource group `rg-container`, App name und Region auswählen
3. Im Schritt **Container**: Image-Quelle und Konfiguration angeben
4. Eine neue **Environment** wird beim ersten Mal automatisch im Assistenten erstellt:
   - Klicke auf **Create new** neben dem Environment-Feld
   - Name: `env-container`, Region: `West Europe`
   - Logs: **Azure Log Analytics** → **Create new** (Portal erstellt automatisch einen Workspace)
   - **Create** bestätigen

!!! tip "Environment wiederverwenden"
    Wenn die Environment einmal erstellt ist, wählst du sie bei jeder weiteren Container App einfach aus dem Dropdown aus.

### Per CLI (Cloud Shell)

```bash
# Log Analytics Workspace für Logs
az monitor log-analytics workspace create \
  --resource-group rg-container \
  --workspace-name log-container

LOG_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-container \
  --workspace-name log-container \
  --query customerId -o tsv)

LOG_WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group rg-container \
  --workspace-name log-container \
  --query primarySharedKey -o tsv)

# Container Apps Environment
az containerapp env create \
  --name env-container \
  --resource-group rg-container \
  --location westeurope \
  --logs-workspace-id $LOG_WORKSPACE_ID \
  --logs-workspace-key $LOG_WORKSPACE_KEY
```

---

## Erste Container App erstellen

### Im Portal

1. Suche nach **Container Apps** → **+ Create**
2. Tab **Basics**:

| Feld | Wert |
|------|------|
| Resource group | `rg-container` |
| Container app name | `meine-container-app` |
| Region | `West Europe` |
| Container Apps Environment | `env-container` |

3. Tab **Container**:

| Feld | Wert |
|------|------|
| Image source | `Docker Hub or other registries` |
| Image and tag | `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest` |

Für eigene ACR-Images: Image source → **Azure Container Registry** → ACR und Image auswählen.

4. Tab **Ingress** (damit die App öffentlich erreichbar ist):

| Feld | Wert |
|------|------|
| Ingress | Aktiviert |
| Ingress traffic | `Accepting traffic from anywhere` |
| Target port | `80` |

5. **Review + create** → **Create**

Nach der Erstellung siehst du die App-URL in der Übersichtsseite unter **Application URL**.

### Per CLI (Cloud Shell)

#### Öffentliches Beispiel-Image (ohne ACR)

```bash
az containerapp create \
  --name meine-container-app \
  --resource-group rg-container \
  --environment env-container \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 3
```

Die URL erscheint in der Ausgabe unter `properties.configuration.ingress.fqdn`.

### Aus der eigenen ACR

```bash
ACR_NAME=$(az acr list --resource-group rg-container --query "[0].name" -o tsv)
ACR_PASS=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

az containerapp create \
  --name flask-container-app \
  --resource-group rg-container \
  --environment env-container \
  --image $ACR_NAME.azurecr.io/flask-app:1.0 \
  --registry-server $ACR_NAME.azurecr.io \
  --registry-username $ACR_NAME \
  --registry-password $ACR_PASS \
  --target-port 5000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5
```

URL abrufen:

```bash
az containerapp show \
  --name flask-container-app \
  --resource-group rg-container \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

---

## Autoscaling konfigurieren

Container Apps skaliert automatisch basierend auf Regeln. Standardmäßig: **HTTP-basiertes Scaling**.

### Im Portal

1. Öffne die Container App im Portal
2. Links: **Scale and replicas**
3. Klicke **+ Add scale rule**

| Feld | Wert |
|------|------|
| Rule name | `http-scaling` |
| Type | `HTTP scaling` |
| Concurrent requests | `10` |

Das bedeutet: Für je 10 gleichzeitige HTTP-Anfragen wird eine neue Instanz gestartet, bis `max-replicas` erreicht ist.

!!! info "Scale to Zero"
    Mit `--min-replicas 0` skaliert die App auf null wenn keine Anfragen kommen – dann keine Kosten. Beim ersten Request startet sie neu (Cold Start: 1–5 Sekunden). Ideal für dev/test-Umgebungen.

### KEDA-basierte Regeln (Event-Driven Scaling)

Container Apps nutzt intern **KEDA** (Kubernetes Event-Driven Autoscaling) für erweiterte Regeln:

```bash
# Skalierung nach Azure Service Bus Queue-Länge
az containerapp update \
  --name flask-container-app \
  --resource-group rg-container \
  --scale-rule-name "queue-scaling" \
  --scale-rule-type "azure-servicebus" \
  --scale-rule-metadata \
    "queueName=meine-queue" \
    "messageCount=5" \
  --scale-rule-auth \
    "connection=connection-string-secret"
```

---

## Revisionen: Deployments ohne Downtime

Jedes Update einer Container App erstellt eine neue **Revision** – die alte läuft weiter. So kannst du Traffic aufteilen (Canary Deployment):

### Neue Revision deployen

```bash
# Neues Image deployen (neue Revision)
az containerapp update \
  --name flask-container-app \
  --resource-group rg-container \
  --image $ACR_NAME.azurecr.io/flask-app:2.0
```

### Traffic-Split: 80% alt, 20% neu

```bash
# Multiple Revision Mode aktivieren
az containerapp revision set-mode \
  --name flask-container-app \
  --resource-group rg-container \
  --mode multiple

# Traffic verteilen
az containerapp ingress traffic set \
  --name flask-container-app \
  --resource-group rg-container \
  --revision-weight \
    latest=20 \
    flask-container-app--ALTE-REVISION=80
```

!!! tip "Canary Deployment in der Praxis"
    Du rollst Version 2.0 zuerst mit 10% Traffic aus, beobachtest Fehlerraten und Response-Zeiten – wenn alles gut ist, erhöhst du auf 100%. Falls Fehler: Traffic zurück auf 0% für die neue Version.

---

## Umgebungsvariablen und Secrets

```bash
# Secrets setzen (verschlüsselt gespeichert)
az containerapp secret set \
  --name flask-container-app \
  --resource-group rg-container \
  --secrets "db-password=SuperGeheim123"

# Umgebungsvariable die auf Secret verweist
az containerapp update \
  --name flask-container-app \
  --resource-group rg-container \
  --set-env-vars "DB_PASSWORD=secretref:db-password"
```

---

## Logs anzeigen

```bash
# Live-Logs
az containerapp logs show \
  --name flask-container-app \
  --resource-group rg-container \
  --follow
```

Im Portal: **Log stream** in der linken Navigation.

---

## Challenge

!!! question "Challenge: Container App mit Scale-to-Zero"
    Erstelle eine Container App aus dem `mein-webserver:1.0`-Image mit:
    
    - Port 8080
    - `min-replicas: 0`, `max-replicas: 3`
    - HTTP-Scaling-Regel: 5 gleichzeitige Anfragen = 1 Instanz
    
    Beobachte im Portal wie viele Replicas laufen wenn du die URL mehrfach aufrufst.

??? success "Hinweis"
    ```bash
    az containerapp create \
      --name webserver-app \
      --resource-group rg-container \
      --environment env-container \
      --image $ACR_NAME.azurecr.io/mein-webserver:1.0 \
      --registry-server $ACR_NAME.azurecr.io \
      --registry-username $ACR_NAME \
      --registry-password $ACR_PASS \
      --target-port 8080 \
      --ingress external \
      --min-replicas 0 \
      --max-replicas 3 \
      --scale-rule-name "http" \
      --scale-rule-type "http" \
      --scale-rule-http-concurrency 5
    ```

---

Weiter zu [Modul 30 – Azure Kubernetes Service: Überblick und Konzepte](modul-30-aks-ueberblick.md) →
