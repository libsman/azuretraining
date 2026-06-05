# Modul 8 – Azure Virtual Network: Dein privates Netzwerk in der Cloud

## Lernziele

Nach diesem Modul kannst du:

- Ein Azure Virtual Network (VNet) mit mehreren Subnetzen erstellen
- VMs in verschiedene Subnetze deployen
- Interne Kommunikation zwischen Subnetzen testen und erklären
- Den Unterschied zwischen privaten und öffentlichen IP-Adressen in Azure beschreiben

---

## Hintergrund: Was ist ein Virtual Network?

**On-Prem-Vergleich:** Ein Azure VNet ist das Cloud-Äquivalent zu einem **lokalen VLAN-Segment**. In einem On-Prem-Rechenzentrum gibt es physische Switches, die Server in Netzwerkbereiche aufteilen – z.B. ein VLAN für Webserver, eines für Datenbankserver, eines für die Management-Ebene. In Azure übernimmt das VNet exakt diese Aufgabe, vollständig softwarebasiert.

| On-Prem | Azure |
|---------|-------|
| LAN / VLAN | Virtual Network (VNet) |
| IP-Subnetz | Subnet |
| Inter-VLAN-Routing (L3-Switch) | Automatisches System-Routing im VNet |
| VLAN-Peering | VNet Peering |
| Site-to-Site VPN | Azure VPN Gateway |

**Warum eigene VNets?**

In Modul 1 hast du eine VM erstellt – die bekam automatisch ein Standard-VNet zugewiesen. Das reicht für den Einstieg. In realen Projekten willst du:

- **Trennung nach Funktion**: Webserver im Frontend-Subnet, Datenbanken im Backend-Subnet
- **Interne Kommunikation**: VMs kommunizieren direkt ohne Umweg über das Internet
- **Sicherheit**: Backend-Ressourcen ohne öffentliche IP
- **Unterschiedliche Firewall-Regeln** pro Subnet (folgt in Modul 9)

Ein VNet ist immer auf eine **Region** beschränkt – du kannst aber mehrere VNets per **Peering** verbinden.

---

## Neue Resource Group erstellen

Lernpfad 2 bekommt eine eigene Resource Group.

1. Suche im Portal nach **Resource groups** → **+ Create**

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-netzwerk` |
| Region | `West Europe` |

2. **Review + create** → **Create**

---

## Virtual Network erstellen

### Schritt 1: Dienst öffnen

1. Tippe in der Suchleiste **`Virtual networks`** → klicke auf den Dienst
2. Klicke auf **+ Create**

### Schritt 2: Basics

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-netzwerk` |
| Name | `vnet-training` |
| Region | `West Europe` |

### Schritt 3: IP-Adressen konfigurieren

Wechsle zum Tab **IP addresses**.

!!! info "Private IP-Adressbereiche (RFC 1918)"
    Azure VNets nutzen private IP-Bereiche:
    - `10.0.0.0/8` – z.B. `10.0.0.0/16` für ein VNet
    - `172.16.0.0/12`
    - `192.168.0.0/16`

    **Tipp für die Praxis:** Wähle einen Bereich der nicht mit deinem On-Prem-Netzwerk überschneidet – sonst gibt es Routing-Konflikte wenn du später eine VPN-Verbindung aufbauen willst.

1. Standardadressraum `10.0.0.0/16` belassen (65.534 nutzbare Adressen)
2. Vorhandenes Default-Subnet löschen (Mülleimer-Symbol)
3. Klicke auf **+ Add a subnet**:

| Feld | Wert |
|------|------|
| Name | `frontend-subnet` |
| Starting address | `10.0.1.0` |
| Size | `/24` (256 Adressen) |

Klicke **Add**, dann erneut **+ Add a subnet**:

| Feld | Wert |
|------|------|
| Name | `backend-subnet` |
| Starting address | `10.0.2.0` |
| Size | `/24` |

Klicke **Add**.

### Schritt 4: Review + Create

**Review + create** → **Create**.

!!! success "VNet mit zwei Subnetzen erstellt"
    `frontend-subnet` für öffentlich erreichbare Ressourcen (Webserver), `backend-subnet` für interne Ressourcen (Datenbanken, Backend-Services).

---

## Zwei VMs in verschiedenen Subnetzen erstellen

Wir deployen eine VM ins Frontend- und eine ins Backend-Subnet.

### VM 1: vm-web1 (Frontend)

1. Suche nach **Virtual machines** → **+ Create** → **Azure virtual machine**

**Basics:**

| Feld | Wert |
|------|------|
| Resource group | `rg-netzwerk` |
| Virtual machine name | `vm-web1` |
| Region | `West Europe` |
| Availability options | `No infrastructure redundancy required` |
| Image | `Ubuntu Server 24.04 LTS` |
| Size | `Standard_B1s` |
| Authentication type | `Password` |
| Username | `azureuser` |
| Password | Sicheres Passwort (notiere es!) |

**Networking:**

| Feld | Wert |
|------|------|
| Virtual network | `vnet-training` |
| Subnet | `frontend-subnet (10.0.1.0/24)` |
| Public IP | `(new) vm-web1-ip` |
| NIC network security group | `Basic` |
| Public inbound ports | `SSH (22)` |

**Review + create** → **Create**

### VM 2: vm-web2 (Backend)

Wiederhole den Vorgang mit diesen Änderungen:

| Feld | Wert |
|------|------|
| Virtual machine name | `vm-web2` |
| Subnet | `backend-subnet (10.0.2.0/24)` |
| Public IP | `(new) vm-web2-ip` |
| Public inbound ports | `SSH (22)` |

Warte bis beide VMs den Status **Running** haben (~2 Minuten).

---

## Interne Kommunikation zwischen Subnetzen testen

In Azure können Subnetze desselben VNets standardmäßig miteinander kommunizieren – anders als bei echten VLANs wo du Inter-VLAN-Routing explizit konfigurieren musst.

### Schritt 1: Private IP von vm-web2 notieren

1. Navigiere im Portal zu `vm-web2` → **Networking** → **Network interface**
2. Klicke auf den Interface-Namen → **IP configurations**
3. Notiere die **Private IP address** (sollte `10.0.2.4` oder ähnlich sein)

### Schritt 2: SSH in vm-web1

Öffne die Cloud Shell (`>_` in der Menüleiste):

```bash
ssh azureuser@<Public-IP-von-vm-web1>
```

### Schritt 3: vm-web2 anpingen

```bash
ping 10.0.2.4 -c 4
```

Du siehst 4 erfolgreiche Antworten – beide VMs sehen sich direkt.

!!! success "Subnetz-Kommunikation funktioniert!"
    Die VMs kommunizieren über ihre privaten IPs, ohne das Internet zu berühren. Azure fügt automatisch eine **System Route** ein die Traffic zwischen allen Subnetzen eines VNets erlaubt.

!!! tip "Effektive Routen anzeigen"
    Gehe zu `vm-web1` → **Networking** → **Network interface** → **Effective routes** um alle aktiven Routen zu sehen. Du erkennst die automatischen System Routes für VNet-internes Routing.

---

## Challenge

!!! question "Challenge: VNet Peering einrichten"
    Erstelle ein zweites VNet `vnet-training-2` mit Adressraum `10.1.0.0/16` und einem Subnet `default-subnet` (`10.1.1.0/24`). Verbinde beide VNets über **VNet Peering** und überprüfe ob eine VM in `vnet-training` eine VM in `vnet-training-2` erreichen kann.

    **Wo?** `vnet-training` → **Peerings** → **+ Add**

??? success "Hinweis"
    1. `vnet-training-2` in `rg-netzwerk` erstellen (Adressraum `10.1.0.0/16`)
    2. `vnet-training` → **Peerings** → **+ Add**
    3. Peering Name (diese Seite): `to-vnet2`
    4. Remote Virtual Network: `vnet-training-2`
    5. **Aktiviere** „Allow traffic to remote virtual network" auf beiden Seiten
    6. Azure erstellt automatisch das symmetrische Gegenstück

    Nach dem Peering können VMs beider VNets über private IPs kommunizieren. VNet Peering ist **nicht transitiv** – wenn A ↔ B und B ↔ C, sehen A und C sich trotzdem nicht direkt.

---

Weiter zu [Modul 9 – Network Security Groups: Traffic filtern](modul-9-nsg.md) →
