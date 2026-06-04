# Modul 1 – Erste VM in Azure

## Lernziele

Nach diesem Modul kannst du:

- Eine Linux-VM in Azure erstellen
- Dich per SSH mit der VM verbinden (direkt im Browser mit Azure Cloud Shell)
- Einen nginx Webserver installieren und die Seite live im Internet aufrufen

---

## Hintergrund: VM in Azure vs. Hyper-V

In Hyper-V erstellst du eine VM, indem du eine ISO einbindest, das Betriebssystem installierst, Netzwerkkarten konfigurierst – das dauert eine Weile. In Azure läuft das deutlich schneller:

- Du wählst ein **Image** aus einer Bibliothek fertiger Betriebssysteme
- Azure startet die VM in **unter 2 Minuten**
- Netzwerk, IP-Adresse und Firewall werden automatisch mitkonfiguriert
- Du zahlst nur für die Zeit, in der die VM **läuft** – ca. 0,009 €/Stunde für die kleinste Größe

---

## VM erstellen

### Schritt 1: Zum VM-Dienst navigieren

1. Öffne das Azure Portal: [portal.azure.com](https://portal.azure.com)
2. Tippe in der Suchleiste **`Virtual machines`** und klicke auf den Dienst
3. Klicke auf **+ Create** → **Azure virtual machine**

### Schritt 2: Basics-Tab konfigurieren

Du siehst jetzt einen Wizard mit mehreren Tabs. Fange mit **Basics** an:

**Projektdetails:**

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-praktikum` |

**Instanzdetails:**

| Feld | Wert |
|------|------|
| Virtual machine name | `vm-training` |
| Region | dieselbe Region wie deine Resource Group (z.B. `Germany West Central`) |
| Availability options | `No infrastructure redundancy required` |
| Security type | `Standard` |
| Image | `Ubuntu Server 24.04 LTS - x64 Gen2` |
| Size | `Standard_B1s` (1 vCPU, 1 GiB RAM) |

!!! tip "Image auswählen"
    Falls Ubuntu 24.04 LTS beim Feld **Image** nicht direkt sichtbar ist, klicke auf **See all images**, suche nach `Ubuntu` und wähle **Ubuntu Server 24.04 LTS** von Canonical.

!!! tip "VM-Größe auswählen"
    Beim Feld **Size** klicke auf **See all sizes**. Suche in der Suchleiste nach `B1s` und wähle **Standard_B1s**. Diese Größe kostet ca. 7 €/Monat und ist für unsere Zwecke mehr als ausreichend.

**Administrator-Konto:**

| Feld | Wert |
|------|------|
| Authentication type | `Password` |
| Username | `azureuser` |
| Password | Ein sicheres Passwort (min. 12 Zeichen, Groß-/Kleinbuchstaben, Zahl, Sonderzeichen) |
| Confirm password | dasselbe Passwort nochmal |

!!! warning "Passwort merken!"
    Schreibe dir das Passwort auf – du brauchst es gleich für die SSH-Verbindung.

**Inbound port rules:**

| Feld | Wert |
|------|------|
| Public inbound ports | `Allow selected ports` |
| Select inbound ports | `SSH (22)` |

### Schritt 3: Disks-Tab

Klicke oben auf den Tab **Disks**:

| Feld | Wert |
|------|------|
| OS disk type | `Standard SSD (locally-redundant storage)` |

Alle anderen Einstellungen auf diesem Tab bleiben unverändert.

### Schritt 4: Networking-Tab

Klicke auf den Tab **Networking**:

Azure erstellt automatisch ein **Virtual Network (VNet)** und ein **Subnet** für dich – das ist dein virtuelles Netzwerk in der Cloud, vergleichbar mit einem VLAN in Hyper-V.

Prüfe folgendes:

- **Virtual network**: `(new) vm-training-vnet` ← automatisch erstellt, passt so
- **Subnet**: `(new) default` ← passt so
- **Public IP**: `(new) vm-training-ip` ← das ist deine öffentliche IP-Adresse

Alle anderen Einstellungen bleiben Standard.

### Schritt 5: Management-Tab

Klicke auf den Tab **Management** und aktiviere **Auto-shutdown**:

1. Setze **Enable auto-shutdown** auf **On**
2. **Shutdown time**: z.B. `19:00`
3. **Time zone**: `(UTC+01:00) Amsterdam, Berlin, Bern, Rome, Stockholm, Vienna`

!!! tip "Warum Auto-Shutdown?"
    In Azure läuft die Uhr solange eine VM eingeschaltet ist. Auto-Shutdown sorgt dafür, dass die VM abends automatisch ausgeht – egal ob man es vergisst. Das ist eine gute Gewohnheit.

### Schritt 6: Review + Create

1. Klicke auf den Tab **Review + create**
2. Azure prüft alle Einstellungen – du siehst oben **Validation passed** in grün
3. Oben auf der Seite siehst du die geschätzten **Kosten pro Monat** für diese VM
4. Klicke auf **Create**

Die VM wird jetzt bereitgestellt – das dauert ca. 1–2 Minuten.

### Schritt 7: Deployment abwarten

Du siehst die Seite "Deployment in progress" mit einem Fortschrittsbalken. Warte bis "Your deployment is complete" erscheint.

Klicke dann auf **Go to resource**.

---

## Mit der VM verbinden (per SSH im Browser)

Du verbindest dich jetzt per SSH mit deiner VM. Dafür nutzen wir **Azure Cloud Shell** – ein vollwertiges Linux-Terminal direkt im Browser, ohne dass du irgendetwas auf deinem Computer installieren musst.

### Schritt 1: Public IP-Adresse kopieren

1. Auf der Übersichtsseite deiner VM `vm-training` findest du rechts die **Public IP address** (z.B. `20.123.45.67`)
2. Klicke auf die IP-Adresse um sie zu kopieren

### Schritt 2: Azure Cloud Shell öffnen

1. Klicke auf das **Terminal-Symbol** `>_` in der oberen Menüleiste des Azure Portals (links neben dem Notifications-Symbol)
2. Beim ersten Öffnen erscheint eine Auswahl – wähle **Bash**
3. Es erscheint der Dialog "You have no storage mounted" – klicke auf **Create storage**
   - Azure erstellt automatisch einen kleinen Storage Account für die Cloud Shell (kostet wenige Cent/Monat)
4. Nach einigen Sekunden öffnet sich ein Terminal am unteren Bildschirmrand

!!! info "Was ist Azure Cloud Shell?"
    Cloud Shell ist ein vollwertiges Linux-Terminal (Bash), das direkt in deinem Browser läuft. Azure hat alle wichtigen Tools vorinstalliert: SSH, Python, Azure CLI (`az`), git und mehr. Kein lokales Setup notwendig.

### Schritt 3: Per SSH verbinden

Tippe im Cloud Shell Terminal den folgenden Befehl. Ersetze `DEINE_PUBLIC_IP` mit der kopierten IP-Adresse deiner VM:

```bash
ssh azureuser@DEINE_PUBLIC_IP
```

Beim ersten Verbinden erscheint diese Sicherheitsabfrage:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Tippe `yes` und drücke Enter.

Dann fragt SSH nach dem Passwort. Gib das Passwort ein, das du bei der VM-Erstellung gesetzt hast. Das Passwort wird beim Tippen nicht angezeigt – das ist normal und so gewollt.

Wenn alles geklappt hat, siehst du:

```
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-1021-azure x86_64)
...
azureuser@vm-training:~$
```

**Du bist jetzt auf deiner Azure-VM!** Was du ab jetzt eintippst, läuft direkt auf einem Server in Microsofts Rechenzentrum.

---

## nginx installieren

nginx ist ein weit verbreiteter Webserver. Wir installieren ihn jetzt auf der VM, damit sie Webseiten an Besucher ausliefern kann.

### Schritt 1: Paketliste aktualisieren

```bash
sudo apt update
```

Warte bis die Ausgabe fertig ist (ca. 15–30 Sekunden). Du siehst am Ende in etwa:

```
Reading package lists... Done
```

### Schritt 2: nginx installieren

```bash
sudo apt install -y nginx
```

Der `-y` Parameter beantwortet die Bestätigungsfrage automatisch mit "Ja". Das dauert ca. 30 Sekunden.

### Schritt 3: Status prüfen

```bash
sudo systemctl status nginx
```

Du solltest `active (running)` in grüner Schrift sehen. Drücke `q` um die Anzeige zu verlassen.

### Schritt 4: Eigene Startseite erstellen

Ersetze die nginx-Standardseite durch deine eigene. Ersetze `DEIN NAME` mit deinem echten Namen:

```bash
echo "<h1>Hallo Azure! Ich bin DEIN NAME</h1><p>Diese Seite läuft auf meiner VM in Microsofts Rechenzentrum.</p>" | sudo tee /var/www/html/index.html
```

---

## Port 80 in der Azure-Firewall öffnen

Aktuell blockiert die Azure-Firewall (**Network Security Group**) noch alle Verbindungen auf Port 80 (HTTP). Wir öffnen ihn jetzt im Azure Portal.

### Schritt 1: Zurück zum Portal

Öffne einen neuen Browser-Tab und gehe zum Azure Portal. Die Cloud Shell bleibt unten geöffnet.

### Schritt 2: Zur VM-Netzwerkkonfiguration navigieren

1. Suche im Portal nach `vm-training` und klicke auf deine VM
2. Klicke links im Menü auf **Networking** → **Network settings**
3. Du siehst die aktuellen Regeln der Network Security Group – momentan ist nur SSH (Port 22) erlaubt

### Schritt 3: HTTP-Regel hinzufügen

1. Klicke auf **+ Create port rule** → **Inbound port rule**
2. Fülle das Formular aus:

| Feld | Wert |
|------|------|
| Source | `Any` |
| Source port ranges | `*` |
| Destination | `Any` |
| Service | `HTTP` |
| Action | `Allow` |
| Priority | `310` |
| Name | `Allow-HTTP` |

3. Klicke auf **Add**

### Schritt 4: Webseite aufrufen

1. Kopiere die **Public IP address** deiner VM (steht auf der VM-Übersichtsseite)
2. Öffne einen neuen Browser-Tab
3. Gib in die Adressleiste ein: `http://DEINE_PUBLIC_IP` – **wichtig: http, nicht https!**
4. Drücke Enter

Du solltest deine Seite sehen: "Hallo Azure! Ich bin [Dein Name]"

!!! success "Deine erste Azure-VM ist live!"
    Du hast eine Linux-VM in der Microsoft Cloud gestartet, dich per SSH verbunden, einen Webserver installiert, die Firewall konfiguriert – und deine Seite ist jetzt im Internet erreichbar. Das ist Cloud Computing in der Praxis.

---

## Challenge

!!! question "Challenge: Eigene HTML-Seite gestalten"
    Die aktuelle Seite ist sehr simpel. Gestalte sie mit richtigem HTML. Sie soll mindestens enthalten:

    - Eine Überschrift (`<h1>`)
    - Einen Absatz mit Text (`<p>`)
    - Eine Liste mit mindestens 3 Punkten (`<ul>` und `<li>`)
    - Ein bisschen CSS-Styling (Farbe, Schriftart – was du möchtest)

??? success "Lösung anzeigen"
    Verbinde dich per SSH mit deiner VM (der SSH-Befehl von vorhin). Öffne dann den Texteditor nano:

    ```bash
    sudo nano /var/www/html/index.html
    ```

    Lösche den alten Inhalt (`Ctrl+K` mehrmals drücken) und schreibe folgendes. Ersetze `[Dein Name]` mit deinem Namen:

    ```html
    <!DOCTYPE html>
    <html lang="de">
    <head>
        <meta charset="UTF-8">
        <title>Meine Azure VM</title>
        <style>
            body {
                font-family: 'Segoe UI', Arial, sans-serif;
                background: #0078d4;
                color: white;
                padding: 40px;
                max-width: 700px;
                margin: 0 auto;
            }
            h1 { font-size: 2.5em; }
            ul { font-size: 1.2em; line-height: 1.8; }
        </style>
    </head>
    <body>
        <h1>Hallo von [Dein Name]!</h1>
        <p>Diese Seite läuft auf meiner Azure VM in Microsofts Rechenzentrum Frankfurt.</p>
        <ul>
            <li>Azure VMs starten in unter 2 Minuten</li>
            <li>Ich zahle nur wenn die VM läuft (ca. 0,009 €/Stunde)</li>
            <li>Die Firewall heißt in Azure "Network Security Group"</li>
        </ul>
    </body>
    </html>
    ```

    Speichern: `Ctrl+O` → `Enter` → `Ctrl+X`

    Lade die Seite im Browser neu – die Änderungen sind sofort sichtbar!

---

Weiter zu [Modul 2 – Storage & Website](modul-2-storage.md) →
