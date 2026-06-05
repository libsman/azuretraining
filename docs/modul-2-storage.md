# Modul 2 – Azure Storage & Statische Website

## Lernziele

Nach diesem Modul kannst du:

- Einen Azure Storage Account erstellen
- Die Static Website Funktion aktivieren
- Eine HTML-Seite direkt aus Azure Blob Storage im Internet veröffentlichen – ohne Server

---

## Hintergrund: Dateien in der Cloud

Für die Website in Modul 1 hast du einen nginx-Server auf einer VM gebraucht. Das funktioniert, kostet aber laufend Geld solange die VM läuft.

Für eine einfache Website aus HTML, CSS und Bildern gibt es einen viel schlankeren Weg: **Azure Blob Storage** mit der Static Website Funktion.

| | VM + nginx | Blob Storage Static Website |
|---|---|---|
| Server nötig? | Ja | Nein |
| Kosten | ~7 €/Monat | < 0,01 €/Monat |
| Skaliert automatisch | Bedingt | Ja, bis zu Millionen Aufrufe |
| Wann geeignet? | Apps, APIs, Backends | Statische HTML/CSS/JS-Seiten |

---

## Storage Account erstellen

### Schritt 1: Zum Storage-Dienst navigieren

1. Öffne das Azure Portal: [portal.azure.com](https://portal.azure.com)
2. Tippe in der Suchleiste **`Storage accounts`** und klicke auf den Dienst
3. Klicke auf **+ Create**

### Schritt 2: Basics konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-aztraining` |
| Storage account name | `staztraining` + deine Initialen oder eine Zahl (z.B. `staztrainingmax1`) |
| Region | dieselbe Region wie deine VM |
| Performance | `Standard` |
| Redundancy | `Locally-redundant storage (LRS)` |

!!! warning "Name muss weltweit eindeutig sein"
    Storage Account Namen müssen **global einzigartig** sein – wie eine URL. Erlaubt sind nur **Kleinbuchstaben und Ziffern**, 3–24 Zeichen. Falls der Name bereits vergeben ist, erscheint ein roter Hinweis – dann einfach eine andere Zahl anhängen.

### Schritt 3: Review + Create

1. Klicke auf **Review + create**
2. Kurz überprüfen, dann **Create** klicken

Das Deployment dauert ca. 20–30 Sekunden. Danach auf **Go to resource** klicken.

---

## Static Website aktivieren

### Schritt 1: Static Website Einstellung öffnen

1. In der linken Navigation deines Storage Accounts: scrolle nach unten zum Abschnitt **Data management**
2. Klicke auf **Static website**

    !!! tip "Tipp: Suchfunktion in der Navigation"
        Falls du die Option nicht sofort findest, klicke oben in der linken Seitenleiste auf das Suchfeld und tippe `static`.

### Schritt 2: Static Website aktivieren

1. Setze den Schalter auf **Enabled**
2. Trage beim Feld **Index document name** ein: `index.html`
3. Das Feld **Error document path** kannst du leer lassen
4. Klicke auf **Save**

Nach dem Speichern erscheinen zwei wichtige Informationen:

- **Primary endpoint**: Die öffentliche URL deiner Website – sieht ungefähr so aus:
  `https://staztrainingmax1.z6.web.core.windows.net/`
- **$web**: Ein neuer Container (Ordner) wurde angelegt – dort kommen deine Dateien rein

!!! info "Kopiere die Primary Endpoint URL!"
    Diese URL brauchst du am Ende um deine fertige Website aufzurufen. Lasse diesen Tab offen oder kopiere die URL in einen Texteditor.

---

## HTML-Datei erstellen

### Schritt 1: index.html erstellen

Erstelle auf deinem Computer eine neue Textdatei und benenne sie `index.html`. Öffne sie mit einem Texteditor (z.B. Notepad oder VS Code) und füge folgenden Inhalt ein.

**Ersetze `DEIN NAME` mit deinem echten Namen!**

```html
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Meine Azure Website</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            font-family: 'Segoe UI', Arial, sans-serif;
            background: linear-gradient(135deg, #0078d4 0%, #106ebe 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
            color: white;
        }
        .card {
            background: rgba(255, 255, 255, 0.15);
            border-radius: 16px;
            padding: 48px 64px;
            text-align: center;
            max-width: 600px;
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.2);
        }
        h1 {
            font-size: 2.5em;
            margin-bottom: 16px;
        }
        p {
            font-size: 1.2em;
            margin-bottom: 8px;
            opacity: 0.9;
        }
        .badge {
            display: inline-block;
            background: rgba(255, 255, 255, 0.2);
            border-radius: 20px;
            padding: 10px 24px;
            margin-top: 28px;
            font-size: 1em;
        }
    </style>
</head>
<body>
    <div class="card">
        <h1>🚀 Hallo Azure!</h1>
        <p>Diese Seite läuft ohne Server direkt aus</p>
        <p><strong>Azure Blob Storage</strong></p>
        <div class="badge">☁️ Erstellt von: DEIN NAME</div>
    </div>
</body>
</html>
```

Speichere die Datei als `index.html`.

---

## Datei hochladen

### Schritt 1: Zum $web Container navigieren

1. Gehe zurück zum Storage Account im Azure Portal
2. Links im Menü unter **Data storage**: klicke auf **Containers**
3. Du siehst den Container **$web** – klicke darauf

### Schritt 2: Datei hochladen

1. Klicke oben auf **Upload**
2. Ein Seitenbereich öffnet sich rechts – klicke auf **Browse for files**
3. Wähle deine `index.html` aus
4. Klicke unten auf **Upload**

Du siehst jetzt deine `index.html` als Datei im Container.

### Schritt 3: Website aufrufen

1. Navigiere links im Menü zurück zu **Static website** (unter Data management)
2. Kopiere die **Primary endpoint** URL
3. Öffne einen neuen Browser-Tab und füge die URL ein
4. Drücke Enter

Du solltest deine blaue Website sehen!

!!! success "Serverlose Website live!"
    Kein Server, kein nginx, keine VM – nur eine HTML-Datei in einem Storage Account, und die ganze Welt kann sie aufrufen. So funktionieren viele moderne Websites und Single-Page-Applications in der Cloud.

---

## Challenge

!!! question "Challenge: Zweite Seite hinzufügen"
    Erweitere deine Website:

    1. Erstelle eine zweite Seite `about.html` mit ein paar Infos über dich (Name, was du in diesem Training lernst)
    2. Füge auf der `index.html` einen Link ein, der zu `about.html` führt
    3. Lade beide Dateien in den `$web` Container hoch und teste die Links

??? success "Lösung anzeigen"
    **1. `about.html` erstellen:**

    ```html
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <title>Über mich</title>
        <style>
            body {
                font-family: 'Segoe UI', Arial, sans-serif;
                background: #f3f2f1;
                padding: 48px;
                max-width: 600px;
                margin: 0 auto;
            }
            h1 { color: #0078d4; }
            a { color: #0078d4; }
        </style>
    </head>
    <body>
        <h1>Über mich</h1>
        <p>Mein Name ist [DEIN NAME].</p>
        <p>Ich lerne gerade Azure kennen und baue erste Cloud-Ressourcen.</p>
        <p>In diesem Training habe ich meine erste VM und meine erste serverlose Website gebaut.</p>
        <br>
        <a href="index.html">← Zurück zur Startseite</a>
    </body>
    </html>
    ```

    **2. Link in `index.html` ergänzen:**

    Füge direkt vor dem schließenden `</div>` in deiner `index.html` folgende Zeile ein:

    ```html
    <br><br>
    <a href="about.html" style="color: white; font-size: 1em;">Über mich →</a>
    ```

    **3. Hochladen:**

    Lade beide Dateien in den `$web` Container hoch. Bei `index.html` erscheint die Frage ob du überschreiben möchtest – wähle **Overwrite**.

---

Weiter zu [Modul 3 – Azure KI](modul-3-ai.md) →
