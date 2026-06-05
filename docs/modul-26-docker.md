# Modul 26 – Docker Grundlagen: Images, Container, Registry

## Lernziele

Nach diesem Modul kannst du:

- Den Unterschied zwischen einem Image und einem Container erklären
- Ein einfaches `Dockerfile` schreiben und ein Image bauen
- Einen Container lokal starten, stoppen und Logs anzeigen
- Ein Image in eine Registry pushen
- Den Vorteil von Containern gegenüber klassischen VMs beschreiben

---

## Hintergrund: Warum Container?

**On-Prem-Vergleich:** Früher hat man Anwendungen direkt auf einem Windows-Server oder einer Linux-VM installiert. Das Problem: "Bei mir läuft's aber" – die Entwicklungsumgebung sah anders aus als der Produktionsserver. Bibliotheksversionen stimmten nicht überein, Abhängigkeiten fehlten.

**Container lösen das:** Ein Container enthält die Anwendung **zusammen mit allem was sie braucht** – Python-Version, pip-Pakete, Systemdateien. Auf jedem Rechner mit Docker läuft exakt dasselbe.

**VM vs. Container:**

| | Virtuelle Maschine | Container |
|--|-------------------|-----------|
| Enthält | Betriebssystem + App | Nur App + Abhängigkeiten |
| Startzeit | Minuten | Sekunden |
| Größe | GByte | MByte |
| Isolation | Vollständig (Hypervisor) | Prozess-Level (Kernel geteilt) |
| Verwendung | Alles | Einzelne Services/Apps |

!!! info "Docker ist nicht Azure-spezifisch"
    Docker läuft lokal auf Windows (Docker Desktop), Linux und macOS – genauso wie auf Azure. In diesem Modul arbeitest du erstmal lokal, ab Modul 27 kommt Azure ins Spiel.

---

## Docker Desktop installieren

Für dieses Modul brauchst du **Docker Desktop** auf deinem lokalen Rechner:

1. Lade [Docker Desktop](https://www.docker.com/products/docker-desktop/) herunter
2. Installiere es (Windows: WSL 2-Backend empfohlen)
3. Starte Docker Desktop und warte bis das Symbol in der Taskleiste grün wird

Prüfen ob Docker läuft:

```bash
docker --version
docker run hello-world
```

Die zweite Zeile lädt ein Test-Image und gibt "Hello from Docker!" aus – dann ist alles in Ordnung.

---

## Grundbegriffe

| Begriff | Bedeutung |
|---------|-----------|
| **Image** | Unveränderliches Paket (Snapshot): App + Abhängigkeiten |
| **Container** | Laufende Instanz eines Images (wie "VM gestartet von ISO") |
| **Dockerfile** | Bauanleitung für ein Image |
| **Registry** | Speicher für Images (Docker Hub, Azure Container Registry) |
| **Layer** | Images bestehen aus Schichten – jeder Schritt im Dockerfile = ein Layer |

---

## Erstes Dockerfile schreiben

Erstelle einen neuen Ordner `mein-container` und darin zwei Dateien:

**`app.py`:**

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hallo aus dem Container!")

    def log_message(self, format, *args):
        pass  # Logs unterdrücken

HTTPServer(("", 8080), Handler).serve_forever()
```

**`Dockerfile`:**

```dockerfile
# Basis-Image: Python 3.12 (schlanke Alpine-Variante)
FROM python:3.12-slim

# Arbeitsverzeichnis im Container setzen
WORKDIR /app

# App-Datei ins Image kopieren
COPY app.py .

# Port freigeben (nur Dokumentation, kein echtes Öffnen)
EXPOSE 8080

# Startbefehl wenn Container läuft
CMD ["python", "app.py"]
```

### Image bauen

Im `mein-container`-Ordner:

```bash
docker build -t mein-webserver:1.0 .
```

- `-t mein-webserver:1.0` – Name und Tag (Version) des Images
- `.` – Dockerfile im aktuellen Verzeichnis

```bash
# Gebaute Images anzeigen
docker images
```

### Container starten

```bash
docker run -d -p 8080:8080 --name webserver mein-webserver:1.0
```

- `-d` – Detached (im Hintergrund laufen)
- `-p 8080:8080` – Port 8080 des Hosts auf Port 8080 des Containers weiterleiten
- `--name webserver` – Name für den Container

Öffne [http://localhost:8080](http://localhost:8080) im Browser – du siehst "Hallo aus dem Container!"

---

## Container verwalten

```bash
# Laufende Container anzeigen
docker ps

# Alle Container (inkl. gestoppte)
docker ps -a

# Logs anzeigen
docker logs webserver

# Logs live mitverfolgen
docker logs -f webserver

# Container stoppen
docker stop webserver

# Container löschen
docker rm webserver

# Image löschen
docker rmi mein-webserver:1.0
```

---

## Mehrschichtiger Aufbau: Layer-Caching

Docker cached jeden Layer. Wenn du nur `app.py` änderst, wird nur der `COPY`-Layer neu gebaut – nicht Python neu installiert. Deshalb: Abhängigkeiten (die sich selten ändern) **vor** dem App-Code ins Dockerfile.

**Optimiertes Dockerfile mit requirements.txt:**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Erst requirements kopieren und installieren (wird gecached)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Dann App-Code kopieren (wird bei jeder Änderung neu gebaut)
COPY . .

EXPOSE 8080
CMD ["python", "app.py"]
```

---

## Image auf Docker Hub pushen (optional)

Docker Hub ist die öffentliche Standard-Registry:

```bash
# Einloggen
docker login

# Image taggen mit Docker Hub Benutzernamen
docker tag mein-webserver:1.0 DEIN-BENUTZERNAME/mein-webserver:1.0

# Pushen
docker push DEIN-BENUTZERNAME/mein-webserver:1.0
```

!!! info "Docker Hub vs. Azure Container Registry"
    **Docker Hub** ist öffentlich und kostenlos für öffentliche Images. **Azure Container Registry (ACR)** ist privat, in deiner Azure-Subscription, und direkt mit ACI, Container Apps und AKS integriert – kommt in Modul 27.

---

## Challenge

!!! question "Challenge: Python Flask-App containerisieren"
    Erstelle ein Image für eine Flask-Webanwendung:
    
    1. Erstelle `requirements.txt` mit: `flask`
    2. Erstelle `app.py` mit einer Flask-App die auf `/` antwortet: "Hallo von Flask im Container!"
    3. Schreibe das Dockerfile (Flask lauscht auf Port 5000)
    4. Baue das Image und starte den Container
    5. Überprüfe dass [http://localhost:5000](http://localhost:5000) antwortet

??? success "Hinweis"
    `Dockerfile`:
    ```dockerfile
    FROM python:3.12-slim
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY app.py .
    EXPOSE 5000
    CMD ["python", "app.py"]
    ```
    
    `app.py`:
    ```python
    from flask import Flask
    app = Flask(__name__)
    
    @app.route("/")
    def hallo():
        return "Hallo von Flask im Container!"
    
    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=5000)
    ```
    
    `docker build -t flask-app:1.0 .`
    `docker run -d -p 5000:5000 flask-app:1.0`

---

Weiter zu [Modul 27 – Azure Container Registry](modul-27-acr.md) →
