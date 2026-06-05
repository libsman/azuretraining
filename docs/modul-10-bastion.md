# Modul 10 – Azure Bastion: Sicher in VMs einloggen ohne Public IP

## Lernziele

Nach diesem Modul kannst du:

- Das Sicherheitsrisiko offener SSH/RDP-Ports erklären
- Azure Bastion deployen und konfigurieren
- Eine VM ohne öffentliche IP und ohne offene Ports über den Browser administrieren
- Die Kostenstruktur von Azure Bastion einschätzen

---

## Hintergrund: Das Problem mit offenen Ports

**On-Prem-Vergleich:** In Unternehmensnetzen ist ein Server nicht direkt per RDP oder SSH aus dem Internet erreichbar. Stattdessen gibt es einen **Jump Server** oder **Bastion Host** in der DMZ – ein gehärteter Server als einziger Einstiegspunkt für Administratoren. Azure Bastion übernimmt genau diese Rolle als vollständig von Microsoft verwalteter Dienst.

**Was passiert wenn Port 22 oder 3389 offen ist:**

- Bots scannen das gesamte Internet laufend nach offenen Ports
- Innerhalb von Minuten beginnen automatisierte Brute-Force-Angriffe
- Bei schwachen Passwörtern oder veralteter Software: schnelle Kompromittierung
- CVE-Schwachstellen in SSH/RDP werden aktiv ausgenutzt

**Die Lösung: Azure Bastion**

```
Ohne Bastion:
Internet → Port 22/3389 (öffentlich) → VM (Public IP)

Mit Bastion:
Internet → HTTPS Port 443 → Bastion (Public IP) → VM (Private IP, kein Public Port)
```

- Verbindung läuft über **HTTPS im Azure Portal** – kein SSH-Client nötig
- Die VM braucht **keine Public IP** mehr
- Port 22/3389 muss nicht aus dem Internet erreichbar sein
- Microsoft übernimmt das Patching und die Härtung des Bastion-Hosts

!!! info "Kosten Azure Bastion"
    - **Basic SKU**: ~0,19 €/Stunde für den Host + 0,019 €/Stunde pro aktiver Verbindung
    - Für dieses Modul (~1–2 Stunden): ca. **0,50 €**
    - **Lösche Bastion nach dem Modul** wenn du Kosten sparen willst – oder lass es für Modul 13 stehen

---

## AzureBastionSubnet hinzufügen

Azure Bastion braucht ein eigenes Subnet. Der Name muss exakt **`AzureBastionSubnet`** lauten (Groß-/Kleinschreibung beachten!). Mindestgröße: `/26` (64 Adressen).

### Schritt 1: Subnet zum bestehenden VNet hinzufügen

1. Navigiere zu `vnet-training` im Portal
2. Klicke links auf **Subnets** → **+ Subnet**
3. Konfiguriere:

| Feld | Wert |
|------|------|
| Name | `AzureBastionSubnet` |
| Starting address | `10.0.3.0` |
| Size | `/26` (64 Adressen) |

4. Klicke **Add**

!!! warning "Exakter Name ist Pflicht"
    Azure erkennt das Bastion-Subnet nur wenn es genau `AzureBastionSubnet` heißt. Groß-/Kleinschreibung stimmt exakt überein – kein Tippfehler erlaubt.

---

## Azure Bastion deployen

### Schritt 1: Dienst öffnen

1. Tippe in der Suchleiste **`Bastions`** → klicke auf den Dienst
2. Klicke auf **+ Create**

### Schritt 2: Konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-netzwerk` |
| Name | `bastion-training` |
| Region | `West Europe` |
| Tier | `Basic` |
| Virtual network | `vnet-training` |
| Subnet | `AzureBastionSubnet` (wird automatisch erkannt) |
| Public IP address name | `bastion-training-ip` |

### Schritt 3: Review + Create

**Review + create** → **Create**

!!! warning "Deployment dauert 5–10 Minuten"
    Azure Bastion braucht länger als normale Ressourcen. Lies in der Zwischenzeit den nächsten Abschnitt.

---

## Public IP von vm-web2 entfernen

`vm-web2` liegt im Backend-Subnet – sie soll **nicht direkt aus dem Internet** erreichbar sein. Jetzt wo Bastion aktiv ist, braucht sie keine Public IP mehr.

1. Navigiere zu `vm-web2` im Portal
2. Klicke links auf **Networking** → klicke auf den Namen des **Network interface** (z.B. `vm-web2123`)
3. Klicke links auf **IP configurations** → klicke auf `ipconfig1`
4. Setze **Public IP address** auf **None**
5. Klicke **Save**

!!! success "vm-web2 hat keine Public IP mehr"
    `vm-web2` ist jetzt ausschließlich über ihre private IP `10.0.2.x` erreichbar – und über Azure Bastion aus dem Portal.

---

## Via Bastion in eine VM einloggen

1. Navigiere zu `vm-web2` im Portal
2. Klicke oben auf **Connect** → wähle **Bastion**
3. Trage ein:
   - Username: `azureuser`
   - Password: dein Passwort
4. Klicke **Connect**

Ein neuer Browser-Tab öffnet sich – du bist direkt in der VM, komplett über HTTPS. Kein Terminal-App nötig, kein offener Port!

!!! success "Secure Admin-Zugriff ohne offenen Port"
    Du bist eingeloggt – aber Port 22 der VM ist nicht aus dem Internet erreichbar. Die VM hat keine Public IP. Trotzdem voller SSH-Zugriff über den Browser.

**Was passiert im Hintergrund:**

```
Dein Browser → HTTPS (443) → Bastion Public IP → VNet-intern → vm-web2:22
                                                (nur intern erreichbar)
```

Der SSH-Tunnel läuft vollständig innerhalb des Azure-Backbone-Netzwerks.

---

## Challenge

!!! question "Challenge: NSG-Regeln für das AzureBastionSubnet"
    Für maximale Sicherheit sollte das `AzureBastionSubnet` selbst durch eine NSG geschützt werden. Microsoft gibt dafür konkrete Regeln vor.
    
    1. Erstelle eine NSG `nsg-bastion` und weise sie dem `AzureBastionSubnet` zu
    2. Füge die **Pflicht-Inbound-Regeln** für Bastion hinzu
    3. Füge die **Pflicht-Outbound-Regeln** hinzu
    
    Dokumentation: [Azure Bastion NSG-Anforderungen](https://learn.microsoft.com/de-de/azure/bastion/bastion-nsg)

??? success "Hinweis – Benötigte Regeln"
    **Inbound (eingehend):**
    
    | Priorität | Quelle | Port | Aktion | Zweck |
    |-----------|--------|------|--------|-------|
    | 100 | Internet | 443 | Allow | Nutzerzugriff über HTTPS |
    | 110 | GatewayManager | 443 | Allow | Azure-interne Verwaltung |
    | 120 | AzureLoadBalancer | 443 | Allow | Health Probes |
    
    **Outbound (ausgehend):**
    
    | Priorität | Ziel | Port | Aktion | Zweck |
    |-----------|------|------|--------|-------|
    | 100 | VirtualNetwork | 22, 3389 | Allow | SSH/RDP zu VMs im VNet |
    | 110 | AzureCloud | 443 | Allow | Bastion-Diagnose und -Telemetrie |
    
    **Service Tags** wie `Internet`, `GatewayManager`, `AzureLoadBalancer`, `VirtualNetwork`, `AzureCloud` sind Azure-interne Bezeichnungen für IP-Adressbereiche – du musst keine konkreten IPs pflegen.

---

Weiter zu [Modul 11 – Azure Load Balancer: Traffic verteilen](modul-11-loadbalancer.md) →
