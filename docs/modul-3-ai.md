# Modul 3 – Azure KI: Bilderkennung

## Lernziele

Nach diesem Modul kannst du:

- Einen Azure Computer Vision Dienst erstellen
- Bilder im Azure Vision Studio ohne Code analysieren
- Ein Python-Script schreiben, das die Computer Vision API aufruft und Bilder beschreibt

---

## Hintergrund: KI als Dienst

Machine Learning und künstliche Intelligenz klingen kompliziert – und die Forschung dahinter ist es auch. Aber **Azure AI Services** ermöglichen es dir, fertig trainierte KI-Modelle mit wenigen Zeilen Code zu nutzen:

- Du schickst ein Bild an eine URL (API)
- Azure gibt dir zurück, was auf dem Bild zu sehen ist
- Du brauchst kein Wissen über Machine Learning oder neuronale Netze

Das Modell hinter Azure Computer Vision wurde von Microsoft mit Hunderten von Millionen Bildern trainiert. Du nutzt es mit einer einzigen HTTP-Anfrage.

---

## Computer Vision Ressource erstellen

### Schritt 1: Zum Dienst navigieren

1. Öffne das Azure Portal: [portal.azure.com](https://portal.azure.com)
2. Tippe in der Suchleiste **`Computer Vision`** und klicke auf den Dienst (Kategorie: AI + machine learning)
3. Klicke auf **+ Create**

### Schritt 2: Konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-aztraining` |
| Region | `West Europe` |
| Name | `cv-aztraining` |
| Pricing tier | `Free F0` |

!!! warning "Region: West Europe wählen"
    Computer Vision ist nicht in allen Regionen verfügbar. Wähle hier **West Europe** (Amsterdam), auch wenn du vorhin Germany West Central genutzt hast.

!!! info "Free Tier F0"
    Der Free Tier erlaubt **5.000 Analysen pro Monat** und 20 pro Minute. Für dieses Training mehr als genug – und es entstehen **keine Kosten**.

### Schritt 3: Review + Create

1. Klicke auf **Review + create**
2. Klicke auf **Create**
3. Nach dem Deployment: **Go to resource**

---

## API-Schlüssel und Endpoint kopieren

Du brauchst zwei Dinge um die API aufzurufen: einen **Schlüssel** (API-Key) und den **Endpoint** (URL deiner Ressource).

### Schlüssel und Endpoint finden

1. Links im Menü deiner Computer Vision Ressource: **Resource Management** → **Keys and Endpoint**
2. Du siehst:
    - **KEY 1** und **KEY 2**: zwei gleichwertige API-Schlüssel (du brauchst nur einen)
    - **Endpoint**: die URL deiner Ressource (z.B. `https://cv-aztraining.cognitiveservices.azure.com/`)

Kopiere **KEY 1** und den **Endpoint** – du brauchst beide gleich. Lass diesen Tab offen.

!!! warning "API-Key geheim halten"
    Dein API-Key ist wie ein Passwort. Wer ihn kennt, kann deinen Dienst aufrufen. Teile ihn nicht mit anderen und lade ihn nicht in öffentliche Git-Repositories hoch.

---

## Vision Studio: KI ohne Code ausprobieren

Bevor wir programmieren, testen wir die KI direkt im Browser.

### Schritt 1: Vision Studio öffnen

Öffne in einem neuen Tab:

**[portal.vision.cognitive.azure.com](https://portal.vision.cognitive.azure.com)**

### Schritt 2: Mit deiner Ressource verbinden

1. Klicke oben rechts auf **View all resources**
2. Deine Ressource `cv-aztraining` sollte in der Liste erscheinen – klicke darauf
3. Klicke auf **Select as default resource** und dann **Done**

### Schritt 3: Bild analysieren

1. Auf der Startseite: klicke auf die Kachel **Image analysis** (oder "Analyze images")
2. Wähle **Add captions to images**
3. Du siehst eine Demo-Oberfläche mit Beispielbildern
4. Lade ein eigenes Bild hoch (Button "Browse for a file") – oder wähle eines der Beispielbilder
5. Aktiviere die Checkbox **"I acknowledge..."** (Nutzungsbedingungen)
6. Klicke auf **Run**

Azure zeigt dir:
- Eine **Beschreibung** des Bildinhalts in natürlicher Sprache
- Erkannte **Objekte** mit Markierungen
- Erkannte **Tags** (Kategorien)

Probiere mehrere Bilder aus – Fotos von draußen, Leute, Tiere, Gegenstände.

!!! success "Das ist echte KI!"
    Was du siehst, sind die Ausgaben eines neuronalen Netzes. Microsoft hat dieses Modell mit Milliarden von Bildern trainiert. Du hast es gerade mit einem Klick genutzt – ohne eine einzige Zeile Code.

---

## Python-Script: Die API direkt aufrufen

Jetzt rufen wir dieselbe KI selbst mit einem Python-Script auf. Wir nutzen dafür **Azure Cloud Shell** – kein lokales Setup nötig.

### Schritt 1: Cloud Shell öffnen

1. Gehe zurück zum Azure Portal
2. Klicke auf das Terminal-Symbol `>_` in der oberen Menüleiste
3. Die Cloud Shell aus Modul 1 sollte noch da sein – falls nicht, wähle **Bash**

### Schritt 2: Neue Python-Datei erstellen

Öffne den eingebauten Cloud Shell Editor:

```bash
code analyse.py
```

Ein Editor öffnet sich im oberen Teil der Cloud Shell. Füge jetzt folgenden Code ein.

**Ersetze in Zeile 5 den Platzhalter mit deinem KEY 1:**

```python
import requests

# ─── Deine Zugangsdaten ───────────────────────────────────────────────────────
endpoint = "https://cv-aztraining.cognitiveservices.azure.com"
key      = "ERSETZE_MICH_MIT_DEINEM_KEY_1"
# ─────────────────────────────────────────────────────────────────────────────

# Sicherheit: trailing slash entfernen falls vorhanden
endpoint = endpoint.rstrip("/")

# API-URL für Image Analysis
url = f"{endpoint}/vision/v3.2/analyze"

headers = {
    "Ocp-Apim-Subscription-Key": key,
    "Content-Type": "application/json"
}

params = {
    "visualFeatures": "Description,Objects,Tags"
}

# Bild-URL – jede öffentliche Bild-URL funktioniert!
body = {
    "url": "https://upload.wikimedia.org/wikipedia/commons/thumb/1/14/Gatto_europeo4.jpg/800px-Gatto_europeo4.jpg"
}

# API aufrufen
print("Sende Anfrage an Azure Computer Vision...")
response = requests.post(url, headers=headers, params=params, json=body)
result   = response.json()

# Fehlerprüfung
if "error" in result:
    print(f"Fehler: {result['error']['message']}")
    exit(1)

# Ergebnis anzeigen
print("\n=== Azure Computer Vision Ergebnis ===\n")

print("Bildbeschreibung:")
for caption in result["description"]["captions"]:
    confidence = int(caption["confidence"] * 100)
    print(f"  → {caption['text']}  ({confidence}% sicher)")

print("\nErkannte Tags:")
tags = result["description"]["tags"][:8]
print("  " + ", ".join(tags))

print("\nErkannte Objekte:")
if result.get("objects"):
    for obj in result["objects"]:
        confidence = int(obj["confidence"] * 100)
        print(f"  • {obj['object']}  ({confidence}% sicher)")
else:
    print("  (keine spezifischen Objekte erkannt)")

print("\n✓ Fertig!")
```

### Schritt 3: Datei speichern

Klicke im Editor oben rechts auf **Save** (oder drücke `Ctrl+S`). Dann schließe den Editor mit dem X.

### Schritt 4: Script ausführen

Im Cloud Shell Terminal:

```bash
python3 analyse.py
```

Du solltest eine Ausgabe wie diese sehen:

```
Sende Anfrage an Azure Computer Vision...

=== Azure Computer Vision Ergebnis ===

Bildbeschreibung:
  → a cat sitting on a surface  (91% sicher)

Erkannte Tags:
  cat, mammal, animal, domestic, indoor, sitting, whiskers, looking

Erkannte Objekte:
  • cat  (90% sicher)

✓ Fertig!
```

!!! success "Du hast gerade KI programmiert!"
    Mit etwa 20 Zeilen Python hast du ein trainiertes KI-Modell aufgerufen, das Bilder in natürlicher Sprache beschreiben kann. Genau so bauen Entwickler bei Microsoft Features wie automatische Bildbeschriftung, Barrierefreiheitsfunktionen oder Suchindizes über Fotos.

---

## Eigenes Bild analysieren

Du kannst jede öffentliche Bild-URL verwenden. Tausche einfach die URL in Zeile `body = {...}` aus.

**Gute Quellen für Bild-URLs:**

- [Wikipedia Commons](https://commons.wikimedia.org) → Rechtsklick auf ein Bild → "Bild-Adresse kopieren"
- Jede andere Website: Rechtsklick auf ein Bild → "Bild-Adresse kopieren" → URL muss direkt auf `.jpg` oder `.png` enden

**Datei bearbeiten:**

```bash
code analyse.py
```

Ändere die URL, speichere, und führe das Script erneut aus:

```bash
python3 analyse.py
```

---

## Challenge

!!! question "Challenge: Mehrere Bilder vergleichen"
    Analysiere drei verschiedene Bilder (z.B. eine Landschaft, ein Tier, eine Stadtansicht) und vergleiche:

    - Wie hoch ist die Konfidenz bei verschiedenen Bilden?
    - Was erkennt die KI gut, was schlechter?
    - Gibt es ein Bild das die KI falsch beschreibt?

??? success "Hinweis"
    Ändere im Script einfach die `body`-Variable und führe das Script jedes Mal neu aus:

    ```python
    body = {
        "url": "HIER_NEUE_BILD_URL"
    }
    ```

    Interessante Testbilder:

    - Sehr einfaches Bild (einzelner Gegenstand) → hohe Konfidenz erwartet
    - Sehr komplexes Bild (Menschenmenge, viele Objekte) → niedrigere Konfidenz
    - Kunstgemälde oder abstrakte Kunst → was erkennt die KI hier?

---

Weiter zu [Modul 4 – Monitoring & Kosten](modul-4-monitoring.md) →
