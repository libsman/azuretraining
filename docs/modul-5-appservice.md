# Modul 5 – Azure App Service: Web App ohne VM

## Lernziele

Nach diesem Modul kannst du:

- Den Unterschied zwischen IaaS (VM) und PaaS (App Service) erklären
- Eine Web App auf Azure App Service erstellen
- Eine einfache Python-Webanwendung über die Cloud Shell deployen

---

## Hintergrund: IaaS vs. PaaS

In Modul 1 hast du eine VM erstellt und darauf manuell nginx installiert. Das ist **IaaS (Infrastructure as a Service)**: du bekommst eine leere Maschine und musst alles selbst einrichten – Betriebssystem-Updates, Webserver, Sicherheitspatches, und so weiter.

**PaaS (Platform as a Service)** geht einen Schritt weiter: du übergibst Azure nur deinen Code, und Azure kümmert sich um alles andere.

| | IaaS – VM (Modul 1) | PaaS – App Service |
|---|---|---|
| Betriebssystem pflegen | Du | Microsoft |
| Webserver installieren | Du (nginx) | Automatisch |
| HTTPS-Zertifikat | Du konfigurierst es | Automatisch |
| Skalierung | Neue VM manuell erstellen | Ein Klick |
| Sicherheitspatches | Deine Verantwortung | Microsofts Verantwortung |
| Kosten | VM läuft 24/7 | Skaliert mit Nutzung |

In der Praxis nutzen Unternehmen App Service für Web-Anwendungen und APIs, wenn sie sich nicht um Serveradministration kümmern wollen.

---

## Web App erstellen

### Schritt 1: Zum Dienst navigieren

1. Öffne das Azure Portal: [portal.azure.com](https://portal.azure.com)
2. Tippe in der Suchleiste **`App Service`** und klicke auf den Dienst
3. Klicke auf **+ Create** → **Web App**

### Schritt 2: Grundkonfiguration

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-aztraining` |
| Name | `webapp-aztraining-XXXX` (XXXX durch 4 zufällige Zahlen ersetzen) |
| Publish | `Code` |
| Runtime stack | `Python 3.12` |
| Operating System | `Linux` |
| Region | `West Europe` |

!!! tip "Eindeutiger Name erforderlich"
    App Service Namen müssen weltweit eindeutig sein, weil deine App unter `dein-name.azurewebsites.net` erreichbar ist. Füge z.B. deine Initialen und das Datum ein: `webapp-aztraining-ms0605`

### Schritt 3: Pricing Plan auswählen

Im Abschnitt **Pricing plans**:

1. Klicke auf **Explore pricing plans**
2. Wähle den Tab **Dev / Test**
3. Wähle **F1 (Free)** – 60 Minuten CPU/Tag, kostenlos

!!! info "Free Tier F1"
    Der Free Tier reicht für unser Training völlig aus. Die App schläft nach kurzer Inaktivität ein (~10 Sekunden Aufwachen beim ersten Aufruf) – das ist im kostenpflichtigen Tier nicht der Fall.

### Schritt 4: Review + Create

1. Klicke auf **Review + create**
2. Klicke auf **Create**
3. Warte ~1–2 Minuten
4. Klicke auf **Go to resource**

---

## App-Code in der Cloud Shell erstellen

### Schritt 1: Cloud Shell öffnen

Klicke im Azure Portal auf das Terminal-Symbol `>_` in der oberen Menüleiste. Wähle **Bash** falls gefragt.

### Schritt 2: Projektverzeichnis und Dateien anlegen

```bash
mkdir ~/webapp && cd ~/webapp
```

Öffne den Editor für die Hauptdatei:

```bash
code app.py
```

Füge folgenden Code ein:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return """
    <html>
    <head>
        <title>Meine Azure Web App</title>
        <style>
            body { font-family: Arial, sans-serif; max-width: 640px;
                   margin: 80px auto; padding: 20px; }
            h1   { color: #0078d4; }
            .box { background: #f0f8ff; border-left: 4px solid #0078d4;
                   padding: 16px; border-radius: 4px; margin: 20px 0; }
        </style>
    </head>
    <body>
        <h1>Hallo Azure! 👋</h1>
        <p>Diese Seite läuft auf <strong>Azure App Service</strong>.</p>
        <div class="box">
            <p>Kein Webserver installiert. Kein Betriebssystem konfiguriert.<br>
            Nur Code – Azure kümmert sich um den Rest.</p>
        </div>
        <p>Hosting-Modell: <strong>PaaS (Platform as a Service)</strong></p>
    </body>
    </html>
    """

@app.route("/status")
def status():
    return {"status": "ok", "dienst": "Azure App Service", "modell": "PaaS"}
```

Klicke im Editor oben auf **Save**, dann schließe den Editor.

### Schritt 3: Abhängigkeiten definieren

```bash
echo "flask" > requirements.txt
echo "gunicorn" >> requirements.txt
```

### Schritt 4: ZIP-Archiv erstellen

```bash
zip app.zip app.py requirements.txt
```

---

## App deployen

### Schritt 1: Code hochladen

Ersetze `webapp-aztraining-XXXX` mit deinem App-Namen aus Schritt 2 oben:

```bash
az webapp deploy \
  --resource-group rg-aztraining \
  --name webapp-aztraining-XXXX \
  --src-path app.zip \
  --type zip
```

!!! tip "Deployment dauert ~1–2 Minuten"
    Azure lädt den Code hoch, führt `pip install -r requirements.txt` aus und startet den Webserver. Du siehst den Fortschritt im Terminal.

### Schritt 2: Startup-Befehl setzen

```bash
az webapp config set \
  --resource-group rg-aztraining \
  --name webapp-aztraining-XXXX \
  --startup-file "gunicorn --bind=0.0.0.0 --timeout 600 app:app"
```

### Schritt 3: App aufrufen

1. Gehe im Portal auf deine Web App (`webapp-aztraining-XXXX`)
2. Oben siehst du die **Default domain** – z.B. `https://webapp-aztraining-xxxx.azurewebsites.net`
3. Klicke darauf – deine App öffnet sich im Browser

!!! success "Web App live!"
    Du hast eine Python-Webanwendung in der Cloud deployed – ohne einen einzigen Server zu konfigurieren. App Service hat Python installiert, den Webserver gestartet, HTTPS eingerichtet und die App öffentlich erreichbar gemacht.

---

## App anpassen und neu deployen

Öffne die Datei in der Cloud Shell:

```bash
cd ~/webapp
code app.py
```

Ändere den Text in der `home()`-Funktion. Speichere, dann neu deployen:

```bash
zip app.zip app.py requirements.txt
az webapp deploy \
  --resource-group rg-aztraining \
  --name webapp-aztraining-XXXX \
  --src-path app.zip \
  --type zip
```

Lade die Seite im Browser neu und sieh die Änderung.

---

## Logs ansehen

Falls etwas nicht funktioniert:

1. Gehe zu deiner Web App im Portal
2. Links: **Deployment** → **Deployment Center** → Tab **Logs**
3. Oder: **Monitoring** → **Log stream** für Live-Ausgabe der App

---

## Challenge

!!! question "Challenge: API-Endpunkte hinzufügen"
    Deine App hat schon einen `/status`-Endpunkt. Füge einen weiteren Endpunkt `/info` hinzu, der als JSON zurückgibt:
    - deinen Namen
    - das heutige Datum
    - eine Liste der Azure-Services, die du in diesem Lernpfad genutzt hast

??? success "Hinweis"
    ```python
    from datetime import date

    @app.route("/info")
    def info():
        return {
            "name": "Dein Name",
            "datum": str(date.today()),
            "azure_services": ["VM", "Storage", "Computer Vision",
                               "Cost Management", "App Service"]
        }
    ```

    Aufruf im Browser: `https://deine-app.azurewebsites.net/info`

---

Weiter zu [Modul 6 – Azure Functions: Serverless](modul-6-functions.md) →
