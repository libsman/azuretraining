# Modul 6 – Azure Functions: Serverless Computing

## Lernziele

Nach diesem Modul kannst du:

- Erklären, was Serverless Computing bedeutet und wann es sinnvoll ist
- Eine Azure Function App (Flex Consumption, Python) im Portal erstellen
- Die Azure Functions Extension in VS Code installieren und nutzen
- Eine HTTP-ausgelöste Funktion schreiben, deployen und im Browser testen
- Den Unterschied zwischen HTTP-Trigger und Timer-Trigger nennen

---

## Hintergrund: Der nächste Abstraktionsschritt

Schaue auf die Abstraktionsebenen aus diesem Lernpfad:

| Modell | Was du verwaltest | Beispiel (Lernpfad 1) |
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
3. Wähle **Flex Consumption** als Hosting-Option

!!! info "Warum Flex Consumption?"
    Seit 2025 gibt es beim Erstellen einer Function App zwei relevante Serverless-Optionen:

    | Plan | OS | Python möglich? | Empfehlung |
    |------|-----|----------------|-----------|
    | **Consumption** | Windows | ❌ Nein | .NET, Node.js, PowerShell |
    | **Flex Consumption** | Linux | ✅ Ja | Python (dieses Training) |
    | Premium / App Service | beides | ✅ | Produktion mit fester Kapazität |

    Python auf Azure Functions läuft immer auf **Linux** und braucht deshalb den **Flex Consumption**-Plan.

### Schritt 2: Grundkonfiguration

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-aztraining` |
| Function App name | `func-aztraining-XXXX` (XXXX durch Zahlen ersetzen – weltweit eindeutig) |
| Runtime stack | `Python` |
| Version | `3.12` |
| Region | `West Europe` |

!!! info "Flex Consumption – Kosten"
    Du zahlst pro Ausführung und pro GB·s verbrauchter Laufzeit. Die ersten **100.000 Ausführungen und 250.000 GB·s** pro Monat sind kostenlos. Für dieses Training entstehen **keine Kosten**.

### Schritt 3: Storage Account

Azure Functions benötigt intern einen Storage Account für Logs und Deployment-Pakete. Azure erstellt automatisch einen neuen – lass die Standardeinstellung.

### Schritt 4: Review + Create

1. Klicke auf **Review + create**
2. Klicke auf **Create**
3. Warte ~1–2 Minuten auf das Deployment
4. Klicke auf **Go to resource**

!!! warning "Kein Code-Editor im Portal"
    Anders als früher gibt es in Azure Functions **keinen eingebauten Code-Editor** mehr im Azure Portal. Code schreiben, testen und deployen läuft jetzt über **VS Code** (mit der Azure Functions Extension) oder die Azure Functions Core Tools. Das ist auch für die Praxis die empfohlene Methode.

---

## VS Code vorbereiten

### Schritt 1: Azure Functions Extension installieren

Öffne VS Code und installiere die Extension:

1. Klicke in der linken Leiste auf das **Extensions-Icon** (oder `Ctrl+Shift+X`)
2. Suche nach **`Azure Functions`**
3. Wähle die Extension von **Microsoft** (Publisher: `ms-azuretools.vscode-azurefunctions`)
4. Klicke auf **Install**

Die Extension installiert automatisch als Abhängigkeit auch **Azure Resources** und **Azure Account**.

!!! tip "Azure Functions Core Tools"
    Die Extension fragt beim ersten Start, ob du die **Azure Functions Core Tools** installieren möchtest. Klicke auf **Install**. Damit kannst du Funktionen später auch lokal testen.
    
    Falls der Dialog ausbleibt, kannst du sie manuell installieren:
    ```powershell
    winget install Microsoft.AzureFunctionsCoreTools
    ```

### Schritt 2: In Azure einloggen

1. Klicke in der linken Leiste auf das **Azure-Icon** (Wolke)
2. Im Bereich **RESOURCES** → klicke auf **Sign in to Azure...**
3. Ein Browser-Fenster öffnet sich – logge dich mit deinem Azure-Konto ein
4. Schließe den Browser-Tab wenn "You are signed in now and can close this page" erscheint
5. In VS Code siehst du jetzt deine Subscription und darunter `func-aztraining-XXXX`

---

## Neues Functions-Projekt erstellen

### Schritt 1: Leeren Ordner anlegen

Erstelle einen neuen Ordner auf deinem PC für das Projekt, z.B. `C:\Projekte\meine-functions`. Öffne ihn in VS Code:

**Datei** → **Ordner öffnen...** → Wähle deinen neuen Ordner

### Schritt 2: Neues Projekt erstellen

Drücke `F1` (oder `Ctrl+Shift+P`) um die Command Palette zu öffnen. Tippe:

```
Azure Functions: Create New Project...
```

Folge dem Assistenten:

| Frage | Antwort |
|-------|---------|
| Select the folder... | Aktueller Ordner (Bestätigen) |
| Select a language | `Python` |
| Select a Python programming model | `Model V2` (empfohlen) |
| Select a Python interpreter | Python 3.12 (aus deiner Installation) |
| Select a template | `HTTP trigger` |
| Provide a function name | `HalloAzure` |
| Authorization level | `ANONYMOUS` |

VS Code erstellt jetzt folgende Dateien:

```
meine-functions/
├── function_app.py      ← dein Code (hier arbeitest du)
├── requirements.txt     ← Python-Pakete
├── host.json            ← Functions-Konfiguration
├── local.settings.json  ← lokale Einstellungen (nicht in Azure)
└── .gitignore
```

!!! info "Model V2 – der moderne Weg"
    Das Python **Model V2** nutzt Dekoratoren (`@app.route`, `@app.timer_trigger`) statt separater `function.json`-Dateien. Sauberer, weniger Dateien, leichter zu lesen.

### Schritt 3: Code anpassen

Öffne `function_app.py` und ersetze den gesamten Inhalt mit:

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
        "nachricht":         f"Hallo, {name}! 👋",
        "dienst":            "Azure Functions (Flex Consumption)",
        "serverzeit":        jetzt,
        "kosten_pro_aufruf": "~0,000004 €"
    }

    return func.HttpResponse(
        body=json.dumps(antwort, ensure_ascii=False),
        status_code=200,
        mimetype="application/json"
    )
```

Speichere die Datei (`Ctrl+S`).

---

## In Azure deployen

### Schritt 1: Deploy to Function App

Drücke `F1` und tippe:

```
Azure Functions: Deploy to Function App...
```

Alternativ: Im **Azure-Bereich** (Wolken-Icon) → Rechtsklick auf `func-aztraining-XXXX` → **Deploy to Function App...**

| Frage | Antwort |
|-------|---------|
| Select subscription | deine Subscription |
| Select Function App | `func-aztraining-XXXX` |

Eine Warnung erscheint: *"Deploying will overwrite..."* → Klicke **Deploy**.

VS Code zippt alle Dateien und lädt sie auf Azure hoch. Im **Output-Bereich** (unten) siehst du den Fortschritt:

```
Deploying to Function App...
Deployment successful.
```

!!! tip "Deployment-Dauer"
    Das erste Deployment dauert 1–2 Minuten (Pakete installieren auf dem Server). Folge-Deployments sind schneller (~30 Sekunden).

### Schritt 2: Funktion im Browser aufrufen

1. Im Azure-Bereich von VS Code: Klappe `func-aztraining-XXXX` → **Functions** auf
2. Rechtsklick auf `HalloAzure` → **Copy Function URL**
3. Öffne die URL im Browser und füge deinen Namen an:

```
https://func-aztraining-xxxx.azurewebsites.net/api/HalloAzure?name=DeinName
```

Du siehst die JSON-Antwort:

```json
{
  "nachricht": "Hallo, DeinName! 👋",
  "dienst": "Azure Functions (Flex Consumption)",
  "serverzeit": "05.06.2026 14:30 UTC",
  "kosten_pro_aufruf": "~0,000004 €"
}
```

!!! success "Serverlose Funktion läuft!"
    Dein Code läuft in Azure – kein Server eingerichtet, kein OS konfiguriert. Du zahlst nur wenn jemand die URL aufruft.

---

## Lokal testen (optional)

Falls du die Azure Functions Core Tools installiert hast, kannst du Funktionen vor dem Deployment lokal testen:

1. Öffne `local.settings.json` und setze:
   ```json
   {
     "IsEncrypted": false,
     "Values": {
       "AzureWebJobsStorage": "UseDevelopmentStorage=true",
       "FUNCTIONS_WORKER_RUNTIME": "python"
     }
   }
   ```
2. Drücke `F5` in VS Code – der lokale Functions-Host startet
3. Im Terminal erscheint: `http://localhost:7071/api/HalloAzure`
4. Rufe diese URL im Browser auf – Änderungen am Code werden sofort aktiv

!!! info "Azurite für lokale Entwicklung"
    `UseDevelopmentStorage=true` nutzt **Azurite**, einen lokalen Speicher-Emulator. VS Code fragt ggf. ob du Azurite installieren möchtest – bestätige mit **Install**.

---

## Zweite Funktion: Timer Trigger

Neben HTTP-Triggern gibt es **Timer Trigger** – das sind Cron Jobs in der Cloud: Code der automatisch zu einem bestimmten Zeitpunkt läuft.

### Timer Trigger hinzufügen

Drücke `F1` → tippe:

```
Azure Functions: Create Function...
```

| Frage | Antwort |
|-------|---------|
| Template | `Timer trigger` |
| Function name | `TaeglicheBegruessung` |
| Cron expression | `0 0 8 * * *` (täglich um 8:00 Uhr UTC) |

VS Code fügt die neue Funktion in `function_app.py` ein. Schau dir den generierten Code an:

```python
@app.timer_trigger(schedule="0 0 8 * * *",
                   arg_name="myTimer",
                   run_on_startup=False,
                   use_monitor=False)
def TaeglicheBegruessung(myTimer: func.TimerRequest) -> None:
    utc_timestamp = datetime.datetime.now(datetime.timezone.utc).isoformat()
    logging.info(f"Tägliche Begrüßung läuft – UTC-Zeit: {utc_timestamp}")
```

Diese Funktion läuft jeden Tag automatisch um 8 Uhr morgens und schreibt einen Log-Eintrag. In der Praxis würde man hier z.B. tägliche Reports erstellen oder Daten bereinigen.

Deploye die aktualisierte App erneut: `F1` → **Azure Functions: Deploy to Function App...**

!!! info "Cron-Syntax in Azure Functions"
    Azure Functions nutzt eine 6-stellige Cron-Syntax: `{Sekunde} {Minute} {Stunde} {Tag} {Monat} {Wochentag}`

    | Cron-Ausdruck | Bedeutung |
    |---|---|
    | `0 0 8 * * *` | Täglich um 8:00 Uhr UTC |
    | `0 */15 * * * *` | Alle 15 Minuten |
    | `0 0 0 * * 1` | Jeden Montag um Mitternacht |
    | `0 30 9 * * 1-5` | Mo–Fr um 9:30 Uhr |

---

## Vergleich: Die drei Hosting-Modelle

| | VM (Modul 1) | App Service (Modul 5) | Functions (dieses Modul) |
|---|---|---|---|
| Startzeit | ~2 Minuten | ~2 Minuten | < 1 Sekunde |
| Minimale Kosten | ~7 €/Monat | 0 € (Free F1) | 0 € (Free Tier) |
| Skalierung | Manuell | Halbautomatisch | Vollautomatisch |
| Wartung | Du | Microsoft | Microsoft |
| Eignet sich für | Komplexe Systeme, Datenbanken | Web Apps, APIs | Event-Handler, kleine APIs, Cron Jobs |

---

## Challenge

!!! question "Challenge: Taschenrechner-Funktion"
    Füge deinem Functions-Projekt eine zweite HTTP-Funktion mit dem Namen `Rechner` hinzu:

    - Sie nimmt zwei URL-Parameter: `a` und `b` (Zahlen)
    - Sie gibt Summe, Produkt und Differenz zurück als JSON
    - Beispielaufruf: `/api/Rechner?a=7&b=3`
    - Erwartete Antwort:

    ```json
    { "summe": 10, "produkt": 21, "differenz": 4 }
    ```

    Deploye anschließend und teste die URL im Browser.

??? success "Hinweis"
    Füge in `function_app.py` unterhalb der `HalloAzure`-Funktion ein:
    
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

    Nach dem Speichern: `F1` → **Azure Functions: Deploy to Function App...**
    
    Teste im Browser: `https://func-aztraining-xxxx.azurewebsites.net/api/Rechner?a=7&b=3`

---

Weiter zu [Modul 7 – Aufräumen](modul-7-aufräumen.md) →

