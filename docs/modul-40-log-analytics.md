# Modul 40 – Azure Monitor & Log Analytics: Logs zentral auswerten

## Lernziele

Nach diesem Modul kannst du:

- Azure Monitor als übergeordnete Monitoring-Plattform beschreiben
- Einen Log Analytics Workspace erstellen und Ressourcen damit verbinden
- KQL (Kusto Query Language) für grundlegende Log-Abfragen nutzen
- Diagnose-Einstellungen für VMs und App Services aktivieren
- Metric Alerts und Log-basierte Alerts erstellen
- Azure Workbooks für visuelle Dashboards verwenden

---

## Hintergrund: Von lokalen Logs zur zentralen Plattform

**On-Prem-Vergleich:** On-Premises landen Logs in Windows Event Logs, `/var/log/` auf Linux oder proprietären SIEM-Systemen wie Splunk oder SCOM. Jeder Server hat seine eigenen Logs – du musst dich per RDP/SSH einloggen um sie anzusehen. Korrelation über mehrere Server ist mühsam.

**Azure Monitor** ist die zentrale Monitoring-Plattform für alles in Azure:

```
Azure Monitor
├── Metrics       → Zahlen in Echtzeit (CPU %, Request/s, ...)
├── Logs          → Textbasierte Ereignisse → werden in Log Analytics gespeichert
├── Alerts        → Benachrichtigungen wenn Schwellwerte überschritten werden
├── Workbooks     → Visuelle Berichte aus Metrics + Logs
└── Application Insights → Telemetrie für Web-Apps (Modul 41)
```

**Log Analytics Workspace** ist die Datenbank für alle Logs. Ressourcen schicken ihre Logs dorthin, du frägst sie mit KQL ab.

!!! info "Kosten von Log Analytics"
    Die ersten **5 GB pro Monat** sind kostenlos (Pay-As-You-Go). Danach ca. 2,76 €/GB. Für dieses Training entstehen kaum Kosten wenn du die Ressourcen am Ende löschst.

---

## Schritt 1: Resource Group und Log Analytics Workspace erstellen

```bash
az group create --name rg-monitoring --location westeurope

az monitor log-analytics workspace create \
  --resource-group rg-monitoring \
  --workspace-name law-training \
  --location westeurope \
  --sku PerGB2018
```

Im Portal: Suche nach **Log Analytics workspaces** → **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-monitoring` |
| Name | `law-training` |
| Region | `West Europe` |
| Pricing tier | `Pay-as-you-go` |

---

## Schritt 2: Diagnose-Einstellungen aktivieren

Verbinde eine vorhandene Ressource (z. B. App Service aus LP1) mit dem Workspace:

### Im Portal

1. Öffne deine App Service-Instanz
2. Links: **Diagnostic settings** (unter Monitoring)
3. **+ Add diagnostic setting**

| Feld | Wert |
|------|------|
| Name | `appservice-to-law` |
| AppServiceHTTPLogs | ✅ |
| AppServiceConsoleLogs | ✅ |
| Destination | `Send to Log Analytics workspace` |
| Workspace | `law-training` |

4. **Save**

### Per CLI

```bash
APP_ID=$(az webapp show --name DEIN-APP-NAME --resource-group rg-aztraining --query id -o tsv)
LAW_ID=$(az monitor log-analytics workspace show \
  --workspace-name law-training \
  --resource-group rg-monitoring \
  --query id -o tsv)

az monitor diagnostic-settings create \
  --name "appservice-to-law" \
  --resource $APP_ID \
  --workspace $LAW_ID \
  --logs '[{"category":"AppServiceHTTPLogs","enabled":true},{"category":"AppServiceConsoleLogs","enabled":true}]'
```

!!! tip "Logs erscheinen nach einigen Minuten"
    Nach dem Aktivieren der Diagnose-Einstellungen dauert es 5–15 Minuten bis die ersten Logs im Workspace erscheinen.

---

## Schritt 3: KQL – Kusto Query Language

KQL ist die Abfragesprache für Log Analytics. Syntax ähnelt SQL aber ist für Zeitreihendaten optimiert.

### Basis-Syntax

```kusto
TabellenName
| where Bedingung
| project Spalte1, Spalte2
| summarize Aggregation by Gruppe
| order by Spalte desc
| take 10
```

### Erste Abfragen im Portal

1. Öffne **Log Analytics workspace** → **Logs**
2. Schließe das "Queries"-Popup
3. Gib folgende Abfragen ein:

**Alle Heartbeat-Signale der letzten Stunde:**
```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| summarize count() by Computer, OSType
| order by count_ desc
```

**HTTP-Fehler der App Service (4xx und 5xx):**
```kusto
AppServiceHTTPLogs
| where TimeGenerated > ago(24h)
| where ScStatus >= 400
| project TimeGenerated, CsMethod, CsUriStem, ScStatus, TimeTaken
| order by TimeGenerated desc
```

**Top 10 langsamste Requests:**
```kusto
AppServiceHTTPLogs
| where TimeGenerated > ago(1h)
| top 10 by TimeTaken desc
| project TimeGenerated, CsUriStem, TimeTaken, ScStatus
```

**Azure Activity Log – wer hat was in meiner Subscription gemacht:**
```kusto
AzureActivity
| where TimeGenerated > ago(7d)
| where ActivityStatusValue == "Failure"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup
| order by TimeGenerated desc
```

---

## Schritt 4: Metric Alert erstellen

Alerts lösen aus wenn eine Metrik einen Schwellwert überschreitet.

### Im Portal

1. Öffne deine App Service-Instanz
2. Links: **Alerts** → **+ Create** → **Alert rule**
3. **Condition**: Klicke **+ Add condition**

| Feld | Wert |
|------|------|
| Signal | `Http 5xx` |
| Operator | `Greater than` |
| Threshold value | `5` |
| Aggregation | `Count` |
| Period | `5 minutes` |

4. **Actions**: Klicke **+ Add action groups** → **+ Create action group**

| Feld | Wert |
|------|------|
| Action group name | `ag-email-training` |
| Notification type | `Email` |
| Email | deine E-Mail-Adresse |

5. **Alert rule details**:

| Feld | Wert |
|------|------|
| Alert rule name | `5xx-errors-alert` |
| Severity | `2 – Warning` |

---

## Schritt 5: Log-basierter Alert

Alert der auslöst wenn eine KQL-Abfrage Ergebnisse liefert:

```bash
LAW_ID=$(az monitor log-analytics workspace show \
  --workspace-name law-training \
  --resource-group rg-monitoring \
  --query id -o tsv)

az monitor scheduled-query create \
  --name "failed-logins-alert" \
  --resource-group rg-monitoring \
  --scopes $LAW_ID \
  --condition-query "AzureActivity | where ActivityStatusValue == 'Failure' | where OperationNameValue contains 'signIn'" \
  --condition-threshold 5 \
  --condition-time-aggregation "Count" \
  --window-size "PT5M" \
  --evaluation-frequency "PT5M" \
  --severity 2 \
  --description "Fehlgeschlagene Login-Versuche"
```

---

## Schritt 6: Azure Workbook erstellen

Workbooks sind interaktive Dashboards die Metrics und Logs kombinieren:

1. Im Log Analytics Workspace: **Workbooks** → **+ New**
2. Klicke **+ Add** → **Add query**
3. Füge eine KQL-Abfrage ein und wähle **Visualization: Bar chart**
4. Klicke **+ Add** → **Add metric** für einen Echtzeit-Graphen
5. **Save** → Name vergeben

---

## Challenge

!!! question "Challenge: KQL-Abfrage für Ressourcen-Änderungen"
    Schreibe eine KQL-Abfrage die folgendes zeigt:
    
    - Alle Ressource-Änderungen in der letzten Woche aus dem Azure Activity Log
    - Nur erfolgreiche Operationen (`ActivityStatusValue == "Success"`)
    - Gruppiert nach `Caller` (wer hat es gemacht) mit der Anzahl der Aktionen
    - Sortiert nach Anzahl absteigend
    
    Erstelle dann einen Alert der auslöst wenn mehr als 20 Ressource-Änderungen in 5 Minuten passieren.

??? success "Hinweis"
    ```kusto
    AzureActivity
    | where TimeGenerated > ago(7d)
    | where ActivityStatusValue == "Success"
    | summarize Aktionen = count() by Caller
    | order by Aktionen desc
    ```

---

Weiter zu [Modul 41 – Application Insights: Telemetrie für Web-Apps](modul-41-app-insights.md) →
