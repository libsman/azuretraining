# Modul 41 – Application Insights: Telemetrie für Web-Apps

## Lernziele

Nach diesem Modul kannst du:

- Application Insights als APM-Lösung (Application Performance Monitoring) erklären
- Eine Application Insights-Ressource erstellen und mit einer Web-App verbinden
- Die wichtigsten Dashboards (Live Metrics, Application Map, Failures) nutzen
- Das Python-SDK in eine Flask-App integrieren
- Custom Events und Custom Metrics senden
- Availability Tests (URL-Pings) einrichten

---

## Hintergrund: Warum Application Insights?

**On-Prem-Vergleich:** On-Premises weißt du als Admin ob ein Server läuft – aber ob eine Web-App korrekt antwortet, welche Seiten am langsamsten sind oder wo Fehler auftreten, siehst du nur durch mühsames Log-Analyse oder spezialisierte Tools wie New Relic oder Dynatrace.

**Application Insights** ist Microsofts APM-Lösung (Application Performance Monitoring) direkt in Azure. Du instrumentierst deine App mit wenigen Zeilen Code und bekommst:

- **Request-Tracking**: Jeder HTTP-Request wird aufgezeichnet (Dauer, Status, URL)
- **Dependency-Tracking**: Aufrufe zu Datenbanken, APIs, Storage automatisch gemessen
- **Exception-Tracking**: Ungefangene Exceptions werden automatisch geloggt
- **Live Metrics**: Echtzeit-Dashboard mit aktiven Requests und Fehlern
- **Application Map**: Grafische Karte aller Abhängigkeiten
- **Availability Tests**: Automatische URL-Pings von mehreren Weltregionen

---

## Schritt 1: Application Insights-Ressource erstellen

```bash
az monitor app-insights component create \
  --app ai-training \
  --resource-group rg-monitoring \
  --location westeurope \
  --kind web \
  --workspace $(az monitor log-analytics workspace show \
    --workspace-name law-training \
    --resource-group rg-monitoring \
    --query id -o tsv)
```

Im Portal: Suche nach **Application Insights** → **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-monitoring` |
| Name | `ai-training` |
| Region | `West Europe` |
| Resource Mode | `Workspace-based` |
| Log Analytics Workspace | `law-training` |

!!! info "Workspace-based Application Insights"
    Seit 2021 empfiehlt Microsoft "workspace-based" App Insights. Alle Daten landen im Log Analytics Workspace – du kannst App Insights-Daten und Infrastruktur-Logs in einer einzigen KQL-Abfrage kombinieren.

---

## Schritt 2: Connection String holen

```bash
az monitor app-insights component show \
  --app ai-training \
  --resource-group rg-monitoring \
  --query connectionString -o tsv
```

Der Connection String sieht so aus:
```
InstrumentationKey=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;...
```

Speichere ihn in einem Key Vault Secret (empfohlen) oder als Umgebungsvariable.

---

## Schritt 3: Python Flask-App instrumentieren

Erstelle eine neue Flask-App in der Cloud Shell:

```bash
mkdir flask-ai-demo && cd flask-ai-demo
pip install flask azure-monitor-opentelemetry
```

Erstelle `app.py`:

```python
import os
from flask import Flask, jsonify, request
from azure.monitor.opentelemetry import configure_azure_monitor

# Application Insights konfigurieren
configure_azure_monitor(
    connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"]
)

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify({"status": "ok", "message": "Hello from monitored Flask app!"})

@app.route("/api/items")
def get_items():
    items = ["Azure Monitor", "Application Insights", "Log Analytics"]
    return jsonify({"items": items, "count": len(items)})

@app.route("/api/error")
def trigger_error():
    raise ValueError("Dies ist ein Test-Fehler für Application Insights")

if __name__ == "__main__":
    app.run(debug=False, port=5000)
```

```bash
export APPLICATIONINSIGHTS_CONNECTION_STRING="DEIN-CONNECTION-STRING"
python app.py
```

Mach nun einige Requests mit `curl http://localhost:5000/` und `curl http://localhost:5000/api/items` – nach 1–2 Minuten erscheinen sie in Application Insights.

---

## Schritt 4: Custom Events und Metrics

Eigene Ereignisse und Metriken senden (für Business-Logik-Tracking):

```python
import os
from flask import Flask, jsonify, request
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

configure_azure_monitor(
    connection_string=os.environ["APPLICATIONINSIGHTS_CONNECTION_STRING"]
)

tracer = trace.get_tracer(__name__)
app = Flask(__name__)

@app.route("/api/purchase")
def purchase():
    item = request.args.get("item", "unknown")
    price = float(request.args.get("price", 0))

    # Custom Span (wird als Dependency in App Map sichtbar)
    with tracer.start_as_current_span("process-purchase") as span:
        span.set_attribute("purchase.item", item)
        span.set_attribute("purchase.price", price)
        # Hier würde Datenbankzugriff stattfinden...

    return jsonify({"status": "purchased", "item": item})
```

---

## Schritt 5: Application Insights-Dashboards nutzen

### Live Metrics

1. Öffne deine Application Insights-Ressource im Portal
2. Links: **Live Metrics**
3. Du siehst in Echtzeit: aktive Verbindungen, eingehende Requests/s, fehlgeschlagene Requests

### Failures

1. Links: **Failures**
2. Zeigt alle Exceptions, fehlgeschlagene Requests und Dependency-Fehler
3. Klicke auf eine Exception → End-to-End Transaction Details (kompletter Call Stack)

### Performance

1. Links: **Performance**
2. Zeigt Response Times per Endpoint, Abhängigkeitszeiten
3. Hilft Bottlenecks zu finden

### Application Map

1. Links: **Application Map**
2. Zeigt eine grafische Karte: deine App → alle Abhängigkeiten (DB, API-Calls, Storage)
3. Rote Knoten = hohe Fehlerrate

---

## Schritt 6: Availability Test einrichten

Availability Tests pingen deine URL regelmäßig von 5 Weltregionen:

1. Application Insights → **Availability** → **+ Create Standard test**

| Feld | Wert |
|------|------|
| Test name | `homepage-ping` |
| URL | URL deiner App (z. B. App Service URL aus LP1) |
| Test frequency | `5 minutes` |
| Test locations | Mehrere Regionen auswählen |
| Success criteria | `HTTP status code = 200` |

2. **Create**

Wenn deine App nicht mehr antwortet, bekommst du automatisch eine E-Mail (wenn eine Action Group konfiguriert ist).

---

## KQL-Abfragen für Application Insights

Im Log Analytics Workspace kannst du App Insights-Daten mit KQL abfragen:

```kusto
// Alle Requests der letzten Stunde
requests
| where timestamp > ago(1h)
| project timestamp, name, duration, resultCode, success
| order by timestamp desc

// Top 5 langsamste Operationen
requests
| where timestamp > ago(24h)
| summarize avg(duration), count() by name
| top 5 by avg_duration desc

// Exceptions der letzten 24h
exceptions
| where timestamp > ago(24h)
| project timestamp, type, outerMessage, operation_Name
| order by timestamp desc
```

---

## Challenge

!!! question "Challenge: Slow Query Alert"
    1. Baue die Flask-App lokal oder auf einem App Service auf
    2. Erstelle einen Alert der auslöst wenn die durchschnittliche Request-Dauer über 1000 ms liegt
    3. Teste ihn indem du `time.sleep(2)` in eine Route einfügst
    
    (Tipp: Alerts → + Create → Alert rule → Signal: "Server response time")

??? success "Hinweis"
    In Application Insights: **Alerts** → **+ Create** → **Alert rule** → Signal: `Server response time` → Threshold: `1000` ms (Average over 5 minutes).
    
    Oder per CLI:
    ```bash
    AI_ID=$(az monitor app-insights component show \
      --app ai-training --resource-group rg-monitoring --query id -o tsv)
    
    az monitor metrics alert create \
      --name "slow-responses" \
      --resource-group rg-monitoring \
      --scopes $AI_ID \
      --condition "avg requests/duration > 1000" \
      --window-size 5m \
      --evaluation-frequency 1m
    ```

---

Weiter zu [Modul 42 – Microsoft Defender for Cloud: Security Posture und Alerts](modul-42-defender.md) →
