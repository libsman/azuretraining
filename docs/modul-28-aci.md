# Modul 28 – Azure Container Instances: Container ohne Kubernetes

## Lernziele

Nach diesem Modul kannst du:

- Azure Container Instances (ACI) als einfachsten Weg zum Container-Deployment in Azure beschreiben
- Einen Container aus der ACR direkt in ACI starten
- Logs eines laufenden Containers in ACI lesen
- ACI mit Container Apps und AKS vergleichen und sinnvolle Einsatzbereiche nennen
- Kosten von ACI (Abrechnung per Sekunde) erklären

---

## Hintergrund: Container ohne Server

**On-Prem-Vergleich:** Wenn du on-prem einen Docker-Container betreiben willst, brauchst du einen Host (Server oder VM) mit Docker installiert. Du kümmerst dich um den Host, Updates, Networking.

**Azure Container Instances** nimmt dir das ab: Du sagst "starte diesen Container" – Azure wählt selbst einen Host, startet den Container und du zahlst nur für die tatsächliche Laufzeit (CPU-Sekunden + RAM-GB-Sekunden). Kein Cluster, kein Kubernetes, keine VM-Verwaltung.

**Wann ist ACI geeignet?**

| Szenario | ACI geeignet? |
|---------|--------------|
| Einmalige Batch-Jobs | ✅ Ja – läuft, fertig, zahlt auf |
| CI/CD-Build-Agent | ✅ Ja |
| Einfache Web-App für Tests | ✅ Ja |
| Produktions-Web-App mit hoher Last | ❌ Nein → Container Apps |
| Microservices mit Skalierung | ❌ Nein → Container Apps / AKS |
| Stateful Workloads | ❌ Nein → AKS |

---

## Container direkt starten (schnellster Weg)

Zuerst das Image aus Modul 27 nutzen. Falls `$ACR_NAME` nicht mehr gesetzt ist:

```bash
ACR_NAME=$(az acr list --resource-group rg-container --query "[0].name" -o tsv)
echo "ACR: $ACR_NAME"
```

### ACI aus ACR-Image starten

```bash
# ACR-Admin aktivieren (für ACI-Authentifizierung)
az acr update --name $ACR_NAME --admin-enabled true

# Admin-Passwort holen
ACR_PASS=$(az acr credential show --name $ACR_NAME --query "passwords[0].value" -o tsv)

# Container starten
az container create \
  --resource-group rg-container \
  --name mein-container \
  --image $ACR_NAME.azurecr.io/mein-webserver:1.0 \
  --registry-login-server $ACR_NAME.azurecr.io \
  --registry-username $ACR_NAME \
  --registry-password $ACR_PASS \
  --ports 8080 \
  --dns-name-label meincontainer$RANDOM \
  --os-type Linux \
  --cpu 1 \
  --memory 1.5
```

!!! info "DNS-Name = öffentliche URL"
    Der Parameter `--dns-name-label` erstellt automatisch eine öffentliche URL: `{label}.westeurope.azurecontainer.io`. Der Name muss in der Region eindeutig sein – deshalb `$RANDOM` anhängen.

### Status und URL abrufen

```bash
az container show \
  --resource-group rg-container \
  --name mein-container \
  --query "{Status:instanceView.state, FQDN:ipAddress.fqdn}" \
  --output table
```

Sobald `Status: Running` → öffne `http://{FQDN}:8080` im Browser.

---

## Container im Portal verwalten

1. Suche nach **Container instances** im Azure Portal
2. Klicke auf `mein-container`

Hier siehst du:
- **Overview**: Status, IP-Adresse, FQDN, CPU/RAM-Verbrauch
- **Containers**: Liste der Container in dieser Instanz + **Connect** (Shell-Zugang)
- **Logs**: Ausgaben des laufenden Containers
- **Events**: Start, Pull, Fehler

### Logs anzeigen

```bash
az container logs \
  --resource-group rg-container \
  --name mein-container
```

### Logs live verfolgen

```bash
az container attach \
  --resource-group rg-container \
  --name mein-container
```

Mit `Ctrl+C` beenden (der Container läuft weiter).

---

## Container neu starten und stoppen

```bash
# Container stoppen (Kosten stoppen)
az container stop \
  --resource-group rg-container \
  --name mein-container

# Container wieder starten
az container start \
  --resource-group rg-container \
  --name mein-container

# Container löschen
az container delete \
  --resource-group rg-container \
  --name mein-container \
  --yes
```

---

## Umgebungsvariablen übergeben

Container bekommen ihre Konfiguration über Umgebungsvariablen:

```bash
az container create \
  --resource-group rg-container \
  --name konfig-container \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --ports 80 \
  --dns-name-label konfig$RANDOM \
  --environment-variables \
    APP_ENV=production \
    APP_VERSION=1.2 \
  --secure-environment-variables \
    DB_PASSWORD=geheim123
```

!!! warning "Sichere Umgebungsvariablen"
    `--secure-environment-variables` verschlüsselt sensible Werte – sie erscheinen nicht in Logs oder Portal. Für echte Secrets: Managed Identity + Key Vault (Modul 22) nutzen statt Passwörter als Env-Variablen.

---

## Kosten verstehen

ACI rechnet pro Sekunde ab:

| Ressource | Preis ca. |
|-----------|----------|
| 1 vCPU pro Sekunde | 0,0000125 € |
| 1 GB RAM pro Sekunde | 0,0000014 € |

Ein Container mit 1 vCPU + 1,5 GB RAM der **1 Stunde** läuft kostet ca. **0,058 €** – also unter 6 Cent.

!!! success "Ideal für Batch-Jobs"
    Ein nächtlicher Batch-Job der 5 Minuten läuft: ca. 0,005 € – also 0,5 Cent. Perfekt für Aufgaben die nicht dauerhaft laufen müssen.

---

## Container Group: Mehrere Container zusammen

ACI unterstützt **Container Groups** – mehrere Container auf einer Instanz (ähnlich wie ein Kubernetes-Pod). Beispiel: App-Container + Logging-Sidecar:

```bash
# Container Group aus YAML-Datei deployen
az container create \
  --resource-group rg-container \
  --file container-group.yaml
```

Für dieses Training genügt ein einzelner Container – Container Groups sind Modul 31 (AKS Pods) vorbehalten.

---

## Challenge

!!! question "Challenge: Flask-App als ACI deployen"
    Starte das `flask-app:1.0`-Image aus der ACR als Azure Container Instance:
    
    - Port 5000 freigeben
    - DNS-Name vergeben
    - Mit einer Umgebungsvariable `GREETING=Hallo` starten
    - Überprüfe per CLI ob der Container läuft und rufe die URL auf

??? success "Hinweis"
    ```bash
    az container create \
      --resource-group rg-container \
      --name flask-app \
      --image $ACR_NAME.azurecr.io/flask-app:1.0 \
      --registry-login-server $ACR_NAME.azurecr.io \
      --registry-username $ACR_NAME \
      --registry-password $ACR_PASS \
      --ports 5000 \
      --dns-name-label flaskapp$RANDOM \
      --environment-variables GREETING=Hallo
    
    az container show \
      --resource-group rg-container \
      --name flask-app \
      --query "ipAddress.fqdn" -o tsv
    ```

---

Weiter zu [Modul 29 – Azure Container Apps](modul-29-containerapps.md) →
