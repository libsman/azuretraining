# Modul 50 – Azure Virtual Desktop (AVD)

## Lernziele

Nach diesem Modul kannst du:

- Die AVD-Architektur und deren Komponenten erklären
- Einen Host Pool mit Session Host VMs erstellen
- Eine Application Group und einen Workspace konfigurieren
- Benutzer einer Desktop-Umgebung zuweisen
- Dich per Browser oder AVD-Client mit einer virtuellen Desktop-Umgebung verbinden

---

## Hintergrund: Virtueller Desktop

On-Premises kennst du Remote Desktop Services (RDS), Citrix oder VMware Horizon: Benutzer verbinden sich per RDP auf einen Session-Server und arbeiten dort. Das Setup ist aufwendig – RD Gateway, Connection Broker, License Server, Session Hosts, Zertifikate.

**Azure Virtual Desktop (AVD)** ist der vollständig verwaltete Microsoft-Dienst dafür: kein eigener Connection Broker, kein RD Gateway, keine Lizenzverwaltung in der Infrastruktur. Microsoft übernimmt die Steuerungsebene. Du verwaltest nur noch die Session-Host-VMs.

| On-Prem RDS | Azure Virtual Desktop |
|-------------|----------------------|
| RD Connection Broker | AVD Control Plane (von Microsoft verwaltet) |
| RD Gateway | Integriert, kein Setup nötig |
| RD Session Host | Session Host VM (z.B. Windows 11 Multi-Session) |
| RD Web Access | AVD Webclient (remote.desktops.microsoft.com) |
| RD Licensing Server | Windows 10/11 Multi-Session (Lizenz inklusive) |
| Einzelne Server-VMs | Host Pool (mehrere VMs, Loadbalancing automatisch) |

**Besonderheit – Windows 10/11 Multi-Session:**  
Microsoft stellt eine spezielle Windows-Version bereit, die mehrere gleichzeitige interaktive Sitzungen erlaubt – wie ein Server, aber mit der Desktop-Optik von Windows 11. Diese Version ist exklusiv für AVD und in Azure inklusive (kein zusätzliches Lizenz-Overhead).

```
Benutzer (Browser / AVD-Client)
          │ HTTPS
          ▼
AVD Control Plane (Microsoft-managed)
          │ Weiterleitung
          ▼
Host Pool
  ├── Session Host VM 1 (Windows 11 Multi-Session)
  ├── Session Host VM 2
  └── Session Host VM N
          │
          ├── Application Group: "Desktop" → ganzer Desktop
          └── Application Group: "RemoteApps" → einzelne Apps (Word, SAP, ...)
                    │
                    ▼
              Workspace (bündelt mehrere App Groups)
```

!!! warning "Kosten"
    Session-Host-VMs sind reguläre Azure VMs – Kosten fallen solange sie laufen an. Für das Training empfiehlt sich der **B2s-Typ** (~0,04 €/h) und ein sofortiges Herunterfahren nach dem Test. Nutze die **Autoscale-Funktion** in der Produktion, um VMs außerhalb der Geschäftszeiten zu stoppen.

---

## Schritt 1: Voraussetzungen prüfen

AVD benötigt zwingend **Microsoft Entra ID** (ehemals Azure AD). Deine Azure-Subscription ist bereits mit einem Entra-Tenant verknüpft.

```bash
# Aktuellen Tenant und Subscription prüfen
az account show --query "{Subscription:name, TenantId:tenantId}" -o table

# Ressourcengruppe (aus Modul 48/49 wiederverwenden oder neu erstellen)
az group create --name rg-lp9 --location westeurope
```

Registriere den AVD-Ressourcenprovider:

```bash
az provider register --namespace Microsoft.DesktopVirtualization
az provider show --namespace Microsoft.DesktopVirtualization --query registrationState -o tsv
# Warte bis "Registered" erscheint (ca. 1 Minute)
```

---

## Schritt 2: Host Pool erstellen

Ein **Host Pool** ist ein Container für Session-Host-VMs. Zwei Typen:

- **Pooled** (Standard): Mehrere Benutzer teilen sich VMs. Kostengünstiger.
- **Personal**: Jeder Benutzer bekommt eine dedizierte VM (wie ein eigener Desktop).

```bash
az desktopvirtualization hostpool create \
  --resource-group rg-lp9 \
  --name hp-training \
  --location westeurope \
  --host-pool-type Pooled \
  --load-balancer-type BreadthFirst \
  --preferred-app-group-type Desktop \
  --max-session-limit 10
```

| Parameter | Bedeutung |
|-----------|-----------|
| `Pooled` | Mehrere User pro VM |
| `BreadthFirst` | Neue Verbindungen auf VM mit wenigsten Sessions → gleichmäßige Last |
| `max-session-limit 10` | Maximal 10 gleichzeitige Sitzungen pro VM |

---

## Schritt 3: Session Host VM hinzufügen

**Im Portal** ist dieser Schritt am einfachsten (der Assistent konfiguriert alles automatisch):

1. Portal → **Azure Virtual Desktop** suchen → **Host pools** → `hp-training`
2. **Session hosts** → **Add**
3. Konfiguration:

| Feld | Wert |
|------|------|
| Virtual machine size | `Standard_B2s` |
| Number of VMs | `1` |
| Image | `Windows 11 Enterprise multi-session + Microsoft 365 Apps` |
| OS disk type | `Standard SSD` |
| Virtual network | Neu erstellen oder vorhandenes VNet nutzen |
| Domain to join | **Microsoft Entra ID** (kein AD DS nötig) |
| Entra ID registration | Aktiviert |

4. Unter **Virtual Machine Administrator account**: Admin-Username und Passwort vergeben
5. **Review + create** → Erstellen

!!! info "Entra-joined vs. AD-joined"
    AVD unterstützt seit 2021 **Entra ID Join** (kein Active Directory nötig). Für Windows-Admins, die einen AD DS haben, ist der **Hybrid-Join** die typische Wahl – damit bleiben Gruppenrichtlinien, Profilserver etc. erhalten.

**Per CLI (alternativ):**

```bash
# Registrierungstoken generieren (Session Hosts brauchen dieses Token zur Anmeldung am Host Pool)
TOKEN=$(az desktopvirtualization hostpool retrieve-registration-token \
  --resource-group rg-lp9 \
  --host-pool-name hp-training \
  --query token -o tsv 2>/dev/null || \
  az desktopvirtualization hostpool update \
  --resource-group rg-lp9 \
  --name hp-training \
  --registration-info expiration-time=$(date -d "+2 hours" -u +"%Y-%m-%dT%H:%M:%S.000Z") registration-token-operation=Update \
  --query registrationInfo.token -o tsv)

echo "Registration Token (für VM-Deployment):"
echo $TOKEN
```

Das Token wird beim VM-Deployment als Parameter übergeben, sodass der **AVD Agent** auf der VM sich automatisch beim Host Pool registriert.

---

## Schritt 4: Application Group und Workspace

Nach dem Host Pool erstellst du eine **Application Group** (welche Apps/Desktops angeboten werden) und einen **Workspace** (der die App Groups für Nutzer sichtbar macht).

```bash
# Desktop Application Group (ganzer Windows-Desktop)
az desktopvirtualization applicationgroup create \
  --resource-group rg-lp9 \
  --name ag-desktop \
  --location westeurope \
  --host-pool-arm-path $(az desktopvirtualization hostpool show \
    --resource-group rg-lp9 \
    --name hp-training \
    --query id -o tsv) \
  --application-group-type Desktop

# Workspace erstellen und Application Group zuweisen
az desktopvirtualization workspace create \
  --resource-group rg-lp9 \
  --name ws-training \
  --location westeurope \
  --application-group-references $(az desktopvirtualization applicationgroup show \
    --resource-group rg-lp9 \
    --name ag-desktop \
    --query id -o tsv)
```

---

## Schritt 5: Benutzer zuweisen

```bash
# Benutzer-Objekt-ID aus Entra ID holen (eigene E-Mail-Adresse einsetzen)
USER_ID=$(az ad user show --id "deine@email.com" --query id -o tsv)

# Rolle "Desktop Virtualization User" auf Application Group zuweisen
AG_ID=$(az desktopvirtualization applicationgroup show \
  --resource-group rg-lp9 \
  --name ag-desktop \
  --query id -o tsv)

az role assignment create \
  --assignee $USER_ID \
  --role "Desktop Virtualization User" \
  --scope $AG_ID

echo "Benutzer wurde der Desktop-App-Group zugewiesen"
```

---

## Schritt 6: Verbinden

**Option 1 – Webbrowser (kein Client nötig):**

1. Öffne [https://client.wvd.microsoft.com/arm/webclient/](https://client.wvd.microsoft.com/arm/webclient/)
2. Melde dich mit deinem Azure-Account an
3. Du siehst den Workspace `ws-training` mit dem Desktop-Symbol
4. Klick → Windows 11 Multi-Session öffnet sich im Browser

**Option 2 – AVD Remote Desktop Client:**

1. Lade den **Windows Desktop Client** von [https://aka.ms/AVDclient](https://aka.ms/AVDclient) herunter
2. Workspace-URL hinzufügen: `https://rdweb.wvd.microsoft.com/api/arm/feeddiscovery`
3. Mit Azure-Account anmelden → Desktop erscheint in der App

!!! tip "Mobile Clients"
    AVD hat offizielle Clients für iOS, Android, macOS und Linux. Alle über [https://aka.ms/AVDclient](https://aka.ms/AVDclient) erreichbar.

---

## Schritt 7: Autoscale (Produktion)

In der Produktion willst du nicht, dass VMs nachts laufen. AVD Autoscale fährt Session Hosts automatisch hoch und runter:

```bash
# Scaling Plan erstellen (verknüpft später mit dem Host Pool)
az desktopvirtualization scalingplan create \
  --resource-group rg-lp9 \
  --name sp-training \
  --location westeurope \
  --host-pool-type Pooled \
  --time-zone "W. Europe Standard Time"
# Zeitpläne über das Portal konfigurieren:
# Ramp-up: 07:00 Uhr – VMs hochfahren
# Peak: 08:00–17:00 – maximale Kapazität
# Ramp-down: 17:00 – VMs abbauen
# Off-peak: 22:00–06:00 – nur Mindest-VMs
```

---

## Challenge

!!! question "Challenge: RemoteApp"
    Statt eines ganzen Desktops kann AVD auch einzelne Anwendungen streamen (RemoteApp).
    
    1. Erstelle eine zweite Application Group vom Typ `RemoteApp` (statt `Desktop`)
    2. Füge die App **Notepad** (`C:\Windows\System32\notepad.exe`) hinzu
    3. Weise deinen Benutzer der neuen App Group zu
    4. Verbinde dich – Notepad öffnet sich wie eine normale Anwendung, ohne dass der Desktop sichtbar ist

??? success "Hinweis"
    ```bash
    # RemoteApp Group erstellen
    az desktopvirtualization applicationgroup create \
      --resource-group rg-lp9 \
      --name ag-remoteapps \
      --location westeurope \
      --host-pool-arm-path $(az desktopvirtualization hostpool show \
        --resource-group rg-lp9 --name hp-training --query id -o tsv) \
      --application-group-type RemoteApp
    
    # Notepad als RemoteApp hinzufügen
    az desktopvirtualization application create \
      --resource-group rg-lp9 \
      --application-group-name ag-remoteapps \
      --name Notepad \
      --friendly-name "Notepad" \
      --file-path "C:\\Windows\\System32\\notepad.exe" \
      --command-line-setting DoNotAllow \
      --icon-path "C:\\Windows\\System32\\notepad.exe" \
      --icon-index 0 \
      --show-in-portal true
    
    # Workspace um RemoteApp Group erweitern
    az desktopvirtualization workspace update \
      --resource-group rg-lp9 \
      --name ws-training \
      --application-group-references \
        $(az desktopvirtualization applicationgroup show --resource-group rg-lp9 --name ag-desktop --query id -o tsv) \
        $(az desktopvirtualization applicationgroup show --resource-group rg-lp9 --name ag-remoteapps --query id -o tsv)
    ```

---

Weiter zu [Modul 51 – Aufräumen Lernpfad 9](modul-51-aufräumen.md) →
