# Modul 48 – Azure Files & File Sync

## Lernziele

Nach diesem Modul kannst du:

- Einen Azure-Dateifreigabe-Dienst (Azure Files) erstellen und konfigurieren
- Eine SMB-Freigabe in Windows und Linux einbinden
- Den Unterschied zwischen Azure Files und Azure Blob Storage erklären
- Azure File Sync konzeptionell verstehen und den Aufbau beschreiben

---

## Hintergrund: Dateiserver in der Cloud

On-Premises hast du wahrscheinlich Windows-Dateiserver mit Netzwerkfreigaben (`\\server\abteilung`). Jede Abteilung hat ihr Verzeichnis, Nutzer mappen Laufwerke per Gruppenrichtlinie.

**Azure Files** ist das Cloud-Äquivalent: Eine verwaltete SMB-Freigabe (SMB 3.x), die du genauso einbinden kannst – mit `\\staccount.file.core.windows.net\sharename`. Kein Server, kein Patching, keine Hardware.

| On-Prem | Azure |
|---------|-------|
| Windows-Dateiserver | Azure Storage Account + File Share |
| `\\server\freigabe` | `\\konto.file.core.windows.net\freigabe` |
| SMB-Protokoll | SMB 3.0 / NFS 4.1 |
| NTFS-Berechtigungen | Azure AD Kerberos-Authentifizierung |
| Server-Backup | Azure Backup für File Shares |

**Azure File Sync** geht noch einen Schritt weiter: Dein bestehender Windows-Dateiserver bleibt – aber sein Inhalt wird automatisch mit Azure Files synchronisiert. Dateien, auf die selten zugegriffen wird, werden aus dem lokalen Cache entfernt (Cloud-Tiering) und bei Bedarf transparent nachgeladen.

```
On-Prem:                        Azure:
Windows Server                  Azure Files (Master-Kopie)
    ↕ File Sync Agent               ↑ Replikation
Lokaler Cache                   Andere Standorte / andere Server
(Hot Files lokal,               können dieselbe Freigabe einbinden
 Cold Files → Azure)
```

---

## Schritt 1: Ressourcengruppe und Storage Account

```bash
az group create --name rg-lp9 --location westeurope

SUFFIX=$RANDOM
STORAGE="stalp9${SUFFIX}"

az storage account create \
  --name $STORAGE \
  --resource-group rg-lp9 \
  --location westeurope \
  --sku Standard_LRS \
  --kind StorageV2 \
  --enable-large-file-share

echo "Storage Account: $STORAGE"
```

!!! info "Standard_LRS reicht für das Training"
    Für Produktivumgebungen empfiehlt sich `Standard_ZRS` (zoneredundant). `--enable-large-file-share` erlaubt Freigaben bis 100 TiB statt 5 TiB.

---

## Schritt 2: File Share erstellen

```bash
# File Share erstellen (5 GiB Quota für das Training)
az storage share-rm create \
  --resource-group rg-lp9 \
  --storage-account $STORAGE \
  --name company-data \
  --quota 5 \
  --enabled-protocols SMB

echo "File Share erstellt: company-data"
```

**Im Portal:** Storage Account → Data storage → File shares → `company-data` siehst du jetzt die leere Freigabe. Mit **Upload** kannst du Testdateien hochladen.

```bash
# Testdatei in der Cloud Shell erstellen und hochladen
echo "Hallo aus Azure Files!" > testdatei.txt

az storage file upload \
  --account-name $STORAGE \
  --share-name company-data \
  --source testdatei.txt \
  --path testdatei.txt \
  --auth-mode login

# Inhalt der Freigabe anzeigen
az storage file list \
  --account-name $STORAGE \
  --share-name company-data \
  --auth-mode login \
  --output table
```

---

## Schritt 3: Windows-Einbindung vorbereiten

Für die Einbindung auf einem Windows-PC (z.B. deinem Arbeitsrechner) generiert das Azure-Portal ein fertiges PowerShell-Skript:

1. Portal → Storage Account → File shares → `company-data` → **Connect**
2. Wähle **Windows** und Laufwerksbuchstabe `Z:`
3. Kopiere das generierte PowerShell-Skript

Das Skript sieht ungefähr so aus:

```powershell
# (Beispiel – nutze das vom Portal generierte Skript)
$connectTestResult = Test-NetConnection -ComputerName "stalp9XXXXX.file.core.windows.net" -Port 445
if ($connectTestResult.TcpTestSucceeded) {
    cmd.exe /C "cmdkey /add:`"stalp9XXXXX.file.core.windows.net`" /user:`"localhost\stalp9XXXXX`" /pass:`"ACCOUNTKEY`""
    New-PSDrive -Name Z -PSProvider FileSystem -Root "\\stalp9XXXXX.file.core.windows.net\company-data" -Persist
} else {
    Write-Error "Port 445 blockiert – prüfe Firewall/ISP"
}
```

!!! warning "Port 445 muss offen sein"
    Manche ISPs blockieren Port 445 (SMB). Im Unternehmensumfeld ist dieser Port i.d.R. offen. Wenn nicht, kannst du Azure Files auch per **HTTPS (REST API)** oder mit dem **Azure Storage Explorer** (kostenlose App) zugreifen.

---

## Schritt 4: Linux-Einbindung (Cloud Shell Demo)

```bash
# In der Cloud Shell testen (Linux-Mount temporär)
ACCOUNT_KEY=$(az storage account keys list \
  --account-name $STORAGE \
  --resource-group rg-lp9 \
  --query "[0].value" -o tsv)

sudo mkdir -p /mnt/company-data

sudo mount -t cifs \
  //${STORAGE}.file.core.windows.net/company-data \
  /mnt/company-data \
  -o vers=3.0,username=${STORAGE},password=${ACCOUNT_KEY},serverino

ls /mnt/company-data

# Unmount wieder
sudo umount /mnt/company-data
```

!!! tip "NFS für Linux"
    Für Linux-Server empfiehlt sich statt SMB das **NFS 4.1-Protokoll** – kein Passwort nötig, native Linux-Performance. Erfordert einen Premium-Storage Account.

---

## Azure File Sync – Konzept und Einrichtung

Azure File Sync ist ein Agent, der auf einem Windows-Server (2012 R2 oder neuer) installiert wird. Er registriert den Server bei einem **Storage Sync Service** in Azure und synchronisiert ausgewählte Ordner mit einem Azure Files Endpoint.

**Kernbegriffe:**

| Begriff | Bedeutung |
|---------|-----------|
| Storage Sync Service | Azure-Ressource, die alles koordiniert |
| Sync Group | Verbindet Cloud-Endpoint (Azure Files) mit Server-Endpoints |
| Cloud-Endpoint | Dein Azure File Share |
| Server-Endpoint | Ordner auf deinem Windows-Server |
| Cloud-Tiering | Seltene Dateien werden aus lokalem Cache entfernt, Datei bleibt als "Platzhalter" |

**Einrichtung im Überblick (für echte Server):**

```bash
# 1. Storage Sync Service erstellen (Portal oder CLI)
az storagesync create \
  --resource-group rg-lp9 \
  --storage-sync-service-name sync-lp9 \
  --location westeurope

# 2. Sync Group erstellen
az storagesync sync-group create \
  --resource-group rg-lp9 \
  --storage-sync-service-name sync-lp9 \
  --sync-group-name sg-company-data

# 3. Cloud Endpoint verknüpfen
az storagesync sync-group cloud-endpoint create \
  --resource-group rg-lp9 \
  --storage-sync-service-name sync-lp9 \
  --sync-group-name sg-company-data \
  --name cloud-ep \
  --storage-account-resource-id $(az storage account show --name $STORAGE --resource-group rg-lp9 --query id -o tsv) \
  --azure-file-share-name company-data
```

Den **Azure File Sync Agent** (kostenloser MSI-Download) installierst du auf deinem Windows-Server und registrierst ihn beim Storage Sync Service.

!!! info "Kein lokaler Server im Training"
    Da dieses Training vollständig über die Cloud Shell läuft, kannst du den Agent-Teil nicht live ausprobieren. In einem echten Szenario mit einem Windows-Server (auch on-prem) funktioniert die Einrichtung über das Portal sehr komfortabel.

---

## Challenge

!!! question "Challenge: Snapshots"
    Azure Files unterstützt Share-Snapshots (read-only Momentaufnahmen).
    
    1. Erstelle einen Snapshot der Freigabe `company-data`
    2. Lösche die `testdatei.txt` aus der Freigabe
    3. Stelle die Datei aus dem Snapshot wieder her
    
    Tipp: `az storage share snapshot` und `az storage file copy start`

??? success "Hinweis"
    ```bash
    # Snapshot erstellen
    SNAPSHOT=$(az storage share snapshot \
      --name company-data \
      --account-name $STORAGE \
      --auth-mode login \
      --query snapshot -o tsv)
    
    # Datei löschen
    az storage file delete \
      --account-name $STORAGE \
      --share-name company-data \
      --path testdatei.txt \
      --auth-mode login
    
    # Aus Snapshot wiederherstellen
    az storage file copy start \
      --account-name $STORAGE \
      --destination-share company-data \
      --destination-path testdatei.txt \
      --source-account-name $STORAGE \
      --source-share company-data \
      --source-path testdatei.txt \
      --source-snapshot $SNAPSHOT
    ```

---

Weiter zu [Modul 49 – Azure Backup & Recovery](modul-49-backup.md) →
