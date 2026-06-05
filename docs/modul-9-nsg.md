# Modul 9 – Network Security Groups: Traffic wie eine Firewall filtern

## Lernziele

Nach diesem Modul kannst du:

- Eine Network Security Group (NSG) erstellen und an ein Subnet binden
- Eingehende und ausgehende Regeln definieren
- Die Azure-Standardregeln erklären und warum DenyAllInBound sinnvoll ist
- Firewall-Regeln testen und debuggen

---

## Hintergrund: Was ist eine NSG?

**On-Prem-Vergleich:** Eine NSG funktioniert wie eine **Paketfilter-Firewall** – ähnlich wie Windows Defender Firewall mit erweiterter Sicherheit, iptables/nftables unter Linux oder eine ACL auf einem Cisco-Router. Du definierst Regeln die festlegen, welcher Traffic erlaubt oder geblockt wird.

| On-Prem | Azure |
|---------|-------|
| Windows Firewall / iptables | NSG auf VM-Ebene (NIC) |
| VLAN-ACL auf Switch/Router | NSG auf Subnet-Ebene |
| Paketfilter-Firewall | NSG mit Prioritätsregeln |
| DMZ-Firewall | NSG auf mehreren Subnetzen |

**NSG wirkt auf zwei Ebenen:**

- **Subnet-Ebene**: Gilt für alle VMs im Subnet (empfohlen)
- **NIC-Ebene**: Gilt für eine einzelne VM

Du kannst beide kombinieren. Dann muss Traffic **beide** NSGs passieren.

**Wie funktionieren NSG-Regeln?**

Jede Regel hat:

| Eigenschaft | Bedeutung |
|-------------|-----------|
| **Priorität** | 100–4096. Niedrigere Zahl = höhere Priorität (wird zuerst ausgewertet) |
| **Source / Destination** | IP, IP-Range, Service Tag (`Internet`, `VirtualNetwork`) oder Application Security Group |
| **Port** | Einzeln, Range (`80-443`) oder `*` für alle |
| **Protocol** | TCP, UDP oder `*` |
| **Action** | `Allow` oder `Deny` |

Azure wertet Regeln von niedrigster zu höchster Priorität aus. Die **erste zutreffende Regel** gewinnt.

---

## NSG erstellen

### Schritt 1: Dienst öffnen

1. Tippe im Portal **`Network security groups`** → klicke auf den Dienst
2. **+ Create**

### Schritt 2: Konfigurieren

| Feld | Wert |
|------|------|
| Resource group | `rg-netzwerk` |
| Name | `nsg-frontend` |
| Region | `West Europe` |

3. **Review + create** → **Create**

---

## NSG dem frontend-subnet zuweisen

1. Navigiere zu `nsg-frontend`
2. Klicke links auf **Subnets** → **+ Associate**
3. Wähle:
   - Virtual network: `vnet-training`
   - Subnet: `frontend-subnet`
4. Klicke **OK**

!!! success "NSG ist jetzt aktiv"
    Jeder Traffic zu oder von VMs im `frontend-subnet` durchläuft die Regeln dieser NSG.

---

## Die drei Azure-Standardregeln verstehen

Gehe zu `nsg-frontend` → **Inbound security rules**. Du siehst:

| Priorität | Name | Aktion | Bedeutung |
|-----------|------|--------|-----------|
| 65000 | AllowVnetInBound | Allow | Gesamter VNet-interner Traffic erlaubt |
| 65001 | AllowAzureLoadBalancerInBound | Allow | Health-Probes von Load Balancern erlaubt |
| 65500 | **DenyAllInBound** | **Deny** | **Alles andere wird geblockt** |

!!! warning "DenyAllInBound – wichtig verstehen"
    Priorität 65500 blockt alles was nicht explizit erlaubt ist. Das ist gut für die Sicherheit.
    
    Da wir beim VM-Erstellen `Public inbound ports: SSH (22)` gewählt haben, hat Azure automatisch eine Allow-Regel für Port 22 (Priorität 1000) erstellt. Ohne diese Regel würde auch SSH blockiert.

---

## nginx installieren und Port 80 öffnen

### Schritt 1: nginx auf vm-web1 installieren

Öffne die Cloud Shell und verbinde dich:

```bash
ssh azureuser@<Public-IP-von-vm-web1>
```

nginx installieren und eine Test-Seite anlegen:

```bash
sudo apt update && sudo apt install -y nginx
echo "<h1>vm-web1 – Azure Netzwerk-Training</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable nginx && sudo systemctl start nginx
```

Lokaler Test:

```bash
curl localhost
# Ausgabe: <h1>vm-web1 – Azure Netzwerk-Training</h1>
```

### Schritt 2: Port 80 im Browser aufrufen

Versuche jetzt im Browser `http://<Public-IP-von-vm-web1>` aufzurufen.

**Ergebnis:** Zeitüberschreitung – Port 80 ist noch durch die NSG geblockt!

### Schritt 3: HTTP-Regel in der NSG hinzufügen

1. Gehe zu `nsg-frontend` → **Inbound security rules** → **+ Add**
2. Konfiguriere:

| Feld | Wert |
|------|------|
| Source | `Any` |
| Source port ranges | `*` |
| Destination | `Any` |
| Service | `HTTP` |
| Action | `Allow` |
| Priority | `200` |
| Name | `allow-http` |

3. Klicke **Add** (Regel wird sofort aktiv – kein Neustart nötig)

### Schritt 4: Website aufrufen

Öffne `http://<Public-IP-von-vm-web1>` im Browser.

Deine Seite erscheint!

!!! success "Firewall-Regel aktiv"
    Du steuerst präzise welcher Traffic deine VM erreicht. SSH (22) und HTTP (80) sind erlaubt, alles andere geblockt.

---

## Effective Security Rules – Debugging-Werkzeug

Möchtest du sehen welche Regeln tatsächlich für eine VM gelten (NSG auf Subnet-Ebene + NSG auf NIC-Ebene kombiniert)?

1. Gehe zu `vm-web1` → **Networking** → **Network interface** → klicke auf den Interface-Namen
2. Klicke oben auf **Effective security rules**

Du siehst alle kombinierten eingehenden und ausgehenden Regeln in der Reihenfolge ihrer Priorität. Sehr hilfreich wenn Traffic unerwartet blockiert oder durchgelassen wird.

---

## Challenge

!!! question "Challenge: HTTP blockieren, nur HTTPS erlauben"
    Ändere die NSG so, dass:
    
    1. Port 80 (HTTP) **explizit geblockt** wird (Deny-Regel)
    2. Port 443 (HTTPS) explizit erlaubt wird

    **Wichtig:** Eine Deny-Regel muss eine **niedrigere Priorität (= höhere Dringlichkeit)** haben als die bestehende Allow-Regel.

??? success "Hinweis"
    **Deny-Regel für Port 80:**
    - Service: `HTTP`, Action: `Deny`, Priority: `100`
    
    Da 100 < 200 (bestehende Allow-HTTP-Regel), wird Deny zuerst ausgewertet.
    
    **Allow-Regel für Port 443:**
    - Service: `HTTPS`, Action: `Allow`, Priority: `300`
    
    Test: Browser auf Port 80 → Zeitüberschreitung (Deny). Port 443 → Verbindungsaufbau beginnt, aber Zertifikatfehler weil kein TLS-Zertifikat konfiguriert (das ist OK – die NSG-Regel funktioniert).

---

Weiter zu [Modul 10 – Azure Bastion: Sicherer VM-Zugriff](modul-10-bastion.md) →
