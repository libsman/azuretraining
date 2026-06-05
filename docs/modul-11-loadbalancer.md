# Modul 11 – Azure Load Balancer: Traffic auf mehrere Server verteilen

## Lernziele

Nach diesem Modul kannst du:

- Einen Azure Standard Load Balancer erstellen und konfigurieren
- Backend-Pool, Health Probe und Load Balancing Rule erklären und einrichten
- Überprüfen, dass Traffic gleichmäßig auf mehrere VMs verteilt wird
- Erläutern, wie Health Probes ausgefallene Server automatisch erkennen

---

## Hintergrund: Was ist ein Load Balancer?

**On-Prem-Vergleich:** Ein Azure Load Balancer ist das Cloud-Äquivalent zu einem **Hardware-Load-Balancer** (F5 BIG-IP, Kemp LoadMaster) oder einer Software-Lösung (HAProxy, nginx). Er empfängt Anfragen an einer einzelnen IP-Adresse und verteilt sie auf mehrere Backend-Server.

| On-Prem | Azure |
|---------|-------|
| Hardware Load Balancer (F5, Kemp) | Azure Load Balancer (Standard) |
| VIP – Virtual IP | Frontend IP Configuration |
| Server Pool / Farm | Backend Pool |
| Health Monitor / Check | Health Probe |
| Balance Rule | Load Balancing Rule |

**Warum Load Balancing?**

- **Ausfallsicherheit**: Fällt ein Server aus, übernimmt der andere – automatisch
- **Skalierung**: Mehr Traffic = mehr Server in den Pool
- **Zero-Downtime-Updates**: Server einzeln updaten während die anderen übernehmen

**Azure Load Balancer SKUs:**

| SKU | Kosten | Empfehlung |
|-----|--------|------------|
| Basic | Kostenlos | **Nicht mehr empfohlen** – wird eingestellt |
| Standard | ~0,005 €/Stunde + Daten | ✅ **Immer Standard wählen** |

---

## Zwei VMs mit nginx vorbereiten

Wir nutzen **cloud-init** um nginx automatisch beim VM-Start zu installieren. So spare wir manuelle Konfigurationsschritte.

### VM 1: vm-lb1 erstellen

1. Suche nach **Virtual machines** → **+ Create** → **Azure virtual machine**

**Basics:**

| Feld | Wert |
|------|------|
| Resource group | `rg-netzwerk` |
| Virtual machine name | `vm-lb1` |
| Region | `West Europe` |
| Image | `Ubuntu Server 24.04 LTS` |
| Size | `Standard_B1s` |
| Authentication type | `Password` |
| Username | `azureuser` |
| Password | Sicheres Passwort |

**Networking:**

| Feld | Wert |
|------|------|
| Virtual network | `vnet-training` |
| Subnet | `frontend-subnet` |
| Public IP | `None` (der Load Balancer bekommt die Public IP) |
| NIC network security group | `None` |

**Advanced** → scrolle zu **Custom data**:

Füge folgendes cloud-init-Skript ein:

```yaml
#cloud-config
packages:
  - nginx
runcmd:
  - echo "<h1>Antwort von: vm-lb1 ✅</h1>" > /var/www/html/index.html
  - systemctl enable nginx
  - systemctl start nginx
```

**Review + create** → **Create**

### VM 2: vm-lb2 erstellen

Exakt wie vm-lb1, mit diesen Änderungen:

| Feld | Wert |
|------|------|
| Virtual machine name | `vm-lb2` |

Custom data für vm-lb2:

```yaml
#cloud-config
packages:
  - nginx
runcmd:
  - echo "<h1>Antwort von: vm-lb2 ✅</h1>" > /var/www/html/index.html
  - systemctl enable nginx
  - systemctl start nginx
```

!!! info "cloud-init"
    cloud-init ist ein Standard-Mechanismus zur automatischen VM-Konfiguration beim ersten Start. Azure unterstützt es für alle Linux-VMs. Das `#cloud-config` am Anfang ist Pflicht und teilt cloud-init mit, was folgt.

Warte bis beide VMs **Running** sind (~2 Minuten). nginx wird parallel installiert.

---

## NSG für den Backend-Traffic konfigurieren

Die VMs haben keine Public IP, aber der Load Balancer schickt Traffic an Port 80. Das `frontend-subnet` braucht eine NSG-Regel die Port 80 erlaubt.

Falls du `nsg-frontend` aus Modul 9 bereits hast: stelle sicher dass Port 80 (allow-http) noch aktiv ist.

Falls keine NSG auf dem Subnet liegt, erstelle eine:

1. Neue NSG `nsg-lb` in `rg-netzwerk`
2. Inbound-Regel: Port `80`, Protocol `TCP`, Action `Allow`, Priority `200`, Name `allow-http`
3. NSG dem `frontend-subnet` zuweisen

---

## Load Balancer erstellen

### Schritt 1: Dienst öffnen

1. Tippe im Portal **`Load balancers`** → **+ Create**

### Schritt 2: Basics

| Feld | Wert |
|------|------|
| Resource group | `rg-netzwerk` |
| Name | `lb-training` |
| Region | `West Europe` |
| SKU | `Standard` |
| Type | `Public` |
| Tier | `Regional` |

### Schritt 3: Frontend IP konfigurieren

1. Klicke auf **+ Add a frontend IP configuration**
2. Name: `lb-frontend`
3. Public IP: **Create new** → Name: `lb-training-ip`
4. Klicke **Add**

### Schritt 4: Backend Pool konfigurieren

1. Klicke auf **+ Add a backend pool**
2. Name: `lb-backend`
3. Virtual network: `vnet-training`
4. Backend Pool Configuration: `IP Address`
5. Klicke auf **+ Add** und trage die privaten IPs von vm-lb1 und vm-lb2 ein

!!! tip "Private IPs herausfinden"
    Navigiere zu `vm-lb1` → **Networking** → **Network interface** → **IP configurations** und notiere die Private IP-Adresse. Wiederhole für `vm-lb2`.

### Schritt 5: Health Probe konfigurieren

1. Klicke auf **+ Add a health probe**

| Feld | Wert |
|------|------|
| Name | `http-probe` |
| Protocol | `TCP` |
| Port | `80` |
| Interval | `5` Sekunden |
| Unhealthy threshold | `2` |

!!! info "Wie funktioniert eine Health Probe?"
    Azure sendet alle 5 Sekunden einen TCP-Connect auf Port 80 zu jeder VM. Schlägt das **2 Mal** hintereinander fehl (= 10 Sekunden), wird die VM als `Unhealthy` markiert und bekommt keinen Traffic mehr. Sobald die Probe wieder erfolgreich ist, kehrt die VM in den Pool zurück.

### Schritt 6: Load Balancing Rule

1. Klicke auf **+ Add a load balancing rule**

| Feld | Wert |
|------|------|
| Name | `http-rule` |
| Frontend IP | `lb-frontend` |
| Backend pool | `lb-backend` |
| Protocol | `TCP` |
| Port | `80` |
| Backend port | `80` |
| Health probe | `http-probe` |
| Session persistence | `None` |

### Schritt 7: Review + Create

**Review + create** → **Create**

---

## Load Balancing testen

### Schritt 1: Public IP des Load Balancers notieren

1. Gehe zu `lb-training` → **Frontend IP configurations**
2. Notiere die **Public IP address**

### Schritt 2: Mehrfach aufrufen

Öffne die Cloud Shell und rufe den Load Balancer mehrfach auf:

```bash
for i in {1..10}; do curl -s http://<LB-Public-IP> && echo; done
```

Du siehst die Antworten abwechselnd von vm-lb1 und vm-lb2:

```
<h1>Antwort von: vm-lb1 ✅</h1>
<h1>Antwort von: vm-lb2 ✅</h1>
<h1>Antwort von: vm-lb1 ✅</h1>
...
```

!!! success "Load Balancing funktioniert!"
    Beide VMs bedienen Anfragen. Der Client sieht immer dieselbe IP (die des Load Balancers).

### Schritt 3: Backend-Health im Portal prüfen

Gehe zu `lb-training` → **Backend pools** → **lb-backend** → **Check backend health**.

Beide VMs zeigen den Status **Healthy** ✅.

---

## Challenge

!!! question "Challenge: Failover simulieren"
    Stoppe nginx auf vm-lb1 und beobachte das automatische Failover:
    
    1. Verbinde dich via Bastion (Modul 10) mit vm-lb1
    2. Stoppe nginx: `sudo systemctl stop nginx`
    3. Warte 10–15 Sekunden (2 × Health Probe Intervall)
    4. Rufe den Load Balancer erneut auf: `curl http://<LB-IP>` – welche VM antwortet?
    5. Prüfe im Portal: Backend health von vm-lb1 = **Unhealthy**
    6. Starte nginx wieder: `sudo systemctl start nginx`
    7. Warte 10–15 Sekunden – vm-lb1 kehrt automatisch in den Pool zurück

??? success "Hinweis"
    Nach 2 fehlgeschlagenen Health Probes (5 + 5 = 10 Sekunden) markiert der Load Balancer vm-lb1 als Unhealthy und leitet **100% des Traffics** zu vm-lb2. Sobald nginx wieder läuft, sind 2 erfolgreiche Probes nötig bis vm-lb1 zurückkommt.
    
    Das nennt sich **Health-based failover** – der Load Balancer erkennt ausgefallene Server automatisch ohne manuellen Eingriff.

---

Weiter zu [Modul 12 – Azure Key Vault: Secrets sicher speichern](modul-12-keyvault.md) →
