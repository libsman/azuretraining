# Modul 6 – Azure Functions: Serverless Computing

## Lernziele

Nach diesem Modul kannst du:

- Erklären, was Serverless Computing bedeutet und wann es sinnvoll ist
- Eine Azure Function App im Portal erstellen
- Eine HTTP-ausgelöste Funktion schreiben und direkt im Browser testen
- Den Unterschied zwischen HTTP-Trigger und Timer-Trigger nennen

---

## Hintergrund: Der nächste Abstraktionsschritt

Schaue auf die Abstraktionsebenen, die du heute kennengelernt hast:

| Modell | Was du verwaltest | Beispiel heute |
|--------|------------------|----------------|
| **IaaS** | VM, OS, Software, Webserver | Azure VM + nginx (Modul 1) |
| **PaaS** | Nur deinen Code, kein Server | Azure App Service (Modul 5) |
| **Serverless** | Nur eine einzelne Funktion | Azure Functions (dieses Modul) |

Bei **Azure Functions** geht es noch einen Schritt weiter als App Service:

- Du schreibst keine ganze Anwendung – nur eine **einzelne Funktion**
- Die Funktion läuft nur, wenn sie aufgerufen wird – du zahlst nur für tatsächliche Ausführungen
- Bei 0 Aufrufen: **0 Kosten, 0 laufende Ressourcen**

Das klingt unscheinbar, ist aber ein Paradigmenwechsel: Früher musste ein Server 24/7 laufen und warten, auch wenn niemand die API aufruft. Mit Functions zahlst du nur für das, was wirklich passiert.

**Typische Einsatzgebiete in der Praxis:**

- Ein Bild wird hochgeladen → Function verarbeitet es automatisch
- Jede Nacht um 2 Uhr → Function bereinigt die Datenbank (Timer Trigger)
- Ein HTTP-Request kommt rein → Function antwortet mit Daten
- Eine neue Nachricht in einer Queue → Function verarbeitet sie sofort

---

## Function App erstellen

### Schritt 1: Zum Dienst navigieren

1. Tippe in der Suchleiste **`Function App`** und klicke auf den Dienst
2. Klicke auf **+ Create**
3. Wähle **Consumption** als Hosting-Option

### Schritt 2: Grundkonfiguration

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-praktikum` |
| Function App name | `func-praktikum-XXXX` (XXXX durch Zahlen ersetzen – weltweit eindeutig) |
| Runtime stack | `Python` |
| Version | `3.12` |
| Region | `West Europe` |
| Operating System | `Linux` |

!!! info "Consumption Plan = echtes Serverless"
    Der Consumption Plan ist der echte Serverless-Ansatz: du zahlst **nur für jede Ausführung** deiner Funktion. Die ersten **1.000.000 Aufrufe pro Monat sind kostenlos**. Für dieses Training entstehen keine Kosten.

### Schritt 3: Storage Account

Azure Functions benötigt intern einen Storage Account für Logs und Deployment-Pakete. Azure erstellt automatisch einen neuen – lass die Standardeinstellung.

### Schritt 4: Review + Create

1. Klicke auf **Review + create**
2. Klicke auf **Create**
3. Warte ~2 Minuten auf das Deployment
4. Klicke auf **Go to resource**

---

## Erste HTTP-Funktion erstellen

### Schritt 1: Neue Function anlegen

1. In deiner Function App: links im Menü auf **Functions** klicken
2. Klicke oben auf **+ Create**
3. Entwicklungsumgebung: **Develop in portal** (direkt im Browser, kein lokales Setup)
4. Template: **HTTP trigger**
5. Name: `HalloAzure`
6. Authorization level: `Anonymous`
7. Klicke auf **Create**

!!! info "Authorization Level"
    `Anonymous` bedeutet: jeder der die URL kennt, kann die Function aufrufen – kein API-Key nötig. Für unser Training ist das praktisch. In der Produktion würde man `Function` oder `Admin` wählen.

### Schritt 2: Code anschauen und anpassen

Azure hat automatisch eine Python-Funktion angelegt. Klicke im linken Menü auf **Code + Test** um den Code zu sehen und zu bearbeiten.

Ersetze den gesamten Inhalt mit folgendem Code:

```python
import azure.functions as func
import logging
import json
from datetime import datetime

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

@app.route(route="HalloAzure")
def HalloAzure(req: func.HttpRequest) -> func.HttpResponse:
    logging.info("HalloAzure wurde aufgerufen.")

    # Name aus URL-Parameter lesen (z.B. ?name=Max)
    name = req.params.get("name", "Welt")

    # Aktuelle Serverzeit (UTC)
    jetzt = datetime.utcnow().strftime("%d.%m.%Y %H:%M UTC")

    # Antwort als JSON
    antwort = {
        "nachricht":        f"Hallo, {name}! 👋",
        "dienst":           "Azure Functions (Serverless)",
        "serverzeit":       jetzt,
        "kosten_pro_aufruf": "~0,0000002 €"
    }

    return func.HttpResponse(
        body=json.dumps(antwort, ensure_ascii=False),
        status_code=200,
        mimetype="application/json"
    )
```

Klicke oben auf **Save**.

### Schritt 3: Funktion testen

Klicke oben auf **Test/Run**:

1. Stelle sicher, dass **HTTP method** auf `GET` steht
2. Füge unter **Query** einen Parameter hinzu:
    - Name: `name`
    - Wert: `Azure-Praktikant`
3. Klicke auf **Run**

Im Bereich **Output** siehst du die JSON-Antwort:

```json
{
  "nachricht": "Hallo, Azure-Praktikant! 👋",
  "dienst": "Azure Functions (Serverless)",
  "serverzeit": "05.06.2026 14:30 UTC",
  "kosten_pro_aufruf": "~0,0000002 €"
}
```

!!! success "Serverlose Funktion läuft!"
    Du hast eine Funktion erstellt und getestet – komplett im Browser, ohne lokales Setup, ohne Server-Konfiguration.

---

## Funktion im Browser aufrufen

### Function URL holen

1. Klicke oben auf **Get function URL**
2. Kopiere die URL (sie endet auf `/api/HalloAzure`)

### Im Browser aufrufen

Öffne die URL in einem neuen Tab. Füge deinen Namen als Parameter an:

```
https://func-praktikum-xxxx.azurewebsites.net/api/HalloAzure?name=DeinName
```

Du siehst die JSON-Antwort direkt im Browser.

!!! tip "JSON besser lesbar machen"
    Wenn der Browser das JSON unformatiert anzeigt, installiere die Extension **JSON Viewer** für Chrome/Edge. Dann sieht es viel lesbarer aus.

---

## Zweite Funktion: Timer Trigger

Neben HTTP-Triggern gibt es **Timer Trigger** – das sind Cron Jobs in der Cloud: Code der automatisch zu einem bestimmten Zeitpunkt läuft.

### Timer Trigger erstellen

1. Gehe zurück zur Übersicht deiner Function App
2. **Functions** → **+ Create**
3. Entwicklungsumgebung: **Develop in portal**
4. Template: **Timer trigger**
5. Name: `TaeglicheBegruessung`
6. Schedule: `0 0 8 * * *` (täglich um 8:00 Uhr UTC)
7. Klicke auf **Create**

Schaue dir den generierten Code an:

```python
import azure.functions as func
import datetime
import logging

app = func.FunctionApp()

@app.timer_trigger(schedule="0 0 8 * * *", arg_name="myTimer",
                   run_on_startup=False, use_monitor=False)
def TaeglicheBegruessung(myTimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.datetime.now(datetime.timezone.utc).isoformat()
    logging.info(f"Tägliche Begrüßung läuft – UTC-Zeit: {utc_timestamp}")
```

Diese Funktion läuft jeden Tag automatisch um 8 Uhr morgens und schreibt einen Log-Eintrag. In der Praxis würde man hier z.B. tägliche Reports erstellen oder Daten bereinigen.

!!! info "Cron-Syntax in Azure Functions"
    Azure Functions nutzt eine 6-stellige Cron-Syntax: `{Sekunde} {Minute} {Stunde} {Tag} {Monat} {Wochentag}`

    Beispiele:

    | Cron-Ausdruck | Bedeutung |
    |---|---|
    | `0 0 8 * * *` | Täglich um 8:00 Uhr |
    | `0 */15 * * * *` | Alle 15 Minuten |
    | `0 0 0 * * 1` | Jeden Montag um Mitternacht |
    | `0 30 9 * * 1-5` | Mo–Fr um 9:30 Uhr |

---

## Überblick: Was hast du heute kennengelernt?

| | VM (Modul 1) | App Service (Modul 5) | Functions (dieses Modul) |
|---|---|---|---|
| Startzeit | ~2 Minuten | ~2 Minuten | < 1 Sekunde |
| Minimale Kosten | ~7 €/Monat | 0 € (Free F1) | 0 € (1M Aufrufe frei) |
| Skalierung | Manuell | Halbautomatisch | Vollautomatisch |
| Wartung | Du | Microsoft | Microsoft |
| Eignet sich für | Komplexe Systeme, Datenbanken | Web Apps, APIs | Event-Handler, kleine APIs, Cron Jobs |

---

## Challenge

!!! question "Challenge: Taschenrechner-Funktion"
    Erstelle eine zweite HTTP-Funktion mit dem Namen `Rechner`:

    - Sie nimmt zwei URL-Parameter: `a` und `b` (Zahlen)
    - Sie gibt Summe, Produkt und Differenz zurück als JSON
    - Beispielaufruf: `/api/Rechner?a=7&b=3`
    - Erwartete Antwort:

    ```json
    { "summe": 10, "produkt": 21, "differenz": 4 }
    ```

??? success "Hinweis"
    ```python
    @app.route(route="Rechner")
    def Rechner(req: func.HttpRequest) -> func.HttpResponse:
        a = float(req.params.get("a", 0))
        b = float(req.params.get("b", 0))

        return func.HttpResponse(
            body=json.dumps({
                "summe":     a + b,
                "produkt":   a * b,
                "differenz": abs(a - b)
            }),
            mimetype="application/json"
        )
    ```

    Teste im Browser: `https://func-praktikum-xxxx.azurewebsites.net/api/Rechner?a=7&b=3`

---

Weiter zu [Modul 7 – Aufräumen](modul-7-aufräumen.md) →
