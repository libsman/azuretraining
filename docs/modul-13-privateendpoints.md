# Modul 13 – Private Endpoints: Azure-Dienste aus dem Internet nehmen

## Lernziele

Nach diesem Modul kannst du:

- Erklären warum Public Endpoints ein Sicherheitsrisiko darstellen
- Einen Private Endpoint für Azure Key Vault erstellen
- Eine Private DNS Zone erstellen und mit dem VNet verknüpfen
- Öffentlichen Zugriff auf einen Azure-Dienst deaktivieren
- Den Unterschied zwischen Private Endpoint und Service Endpoint erklären

---

## Hintergrund: Das Problem mit Public Endpoints

**On-Prem-Vergleich:** Stell dir vor, dein Datenbankserver ist aus dem Internet erreichbar – obwohl nur interne Server darauf zugreifen sollen. Das würde kein Admin so konfigurieren. In Azure ist das aber oft der Standard: Key Vault, Storage Accounts, SQL-Datenbanken – alle haben standardmäßig eine **öffentliche IP** und sind weltweit aufrufbar (mit Authentifizierung, aber trotzdem angreifbar).

**Private Endpoints** lösen dieses Problem strukturell:

```
Ohne Private Endpoint:
VM in VNet  →  Internet  →  kv-aztraining.vault.azure.net (Public IP 20.x.x.x)

Mit Private Endpoint:
VM in VNet  →  VNet-intern  →  10.0.2.5 (Private IP)  →  Key Vault
                                          ↑
                                  Kein Internet-Hop
```

Der Traffic bleibt **vollständig im Microsoft-Backbone-Netzwerk**. Der Azure-Dienst bekommt eine private IP direkt in deinem VNet.

**Private Endpoint vs. Service Endpoint**

| | Private Endpoint | Service Endpoint |
|--|-----------------|-----------------|
| Eigene private IP im VNet? | ✅ Ja | ❌ Nein |
| Traffic über öffentliches Internet? | ❌ Nein | ❌ Nein |
| Von On-Prem über VPN erreichbar? | ✅ Ja | ❌ Nein |
| DNS-Konfiguration nötig? | Ja (Automatisch) | Nein |
| Empfohlen für neue Projekte? | ✅ Ja | Nicht mehr bevorzugt |

---

## Subnet für Private Endpoint vorbereiten

Wir nutzen das `backend-subnet` aus Modul 8. Private Endpoints können in jedem bestehenden Subnet liegen.

!!! tip "Eigenes Subnet für Übersicht?"
    In großen Projekten legt man oft ein eigenes `private-endpoints-subnet` an um alle Private Endpoints gebündelt zu haben. Für dieses Training nutzen wir das `backend-subnet`.

---

## Private Endpoint für Key Vault erstellen

### Schritt 1: Über den Key Vault navigieren

1. Gehe zu `kv-aztraining-XXXX` im Portal
2. Klicke links auf **Networking** → Tab **Private endpoint connections**
3. Klicke **+ Create private endpoint**

### Schritt 2: Basics

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-netzwerk` |
| Name | `pe-keyvault` |
| Region | `West Europe` |

### Schritt 3: Resource

| Feld | Wert |
|------|------|
| Connection method | `Connect to an Azure resource in my directory` |
| Resource type | `Microsoft.KeyVault/vaults` |
| Resource | `kv-aztraining-XXXX` |
| Target sub-resource | `vault` |

### Schritt 4: Virtual Network

| Feld | Wert |
|------|------|
| Virtual network | `vnet-training` |
| Subnet | `backend-subnet` |
| Private IP configuration | `Dynamically allocate IP address` |

### Schritt 5: DNS Integration

| Feld | Wert |
|------|------|
| Integrate with private DNS zone | `Yes` |
| Private DNS Zone | `(new) privatelink.vaultcore.azure.net` |

!!! info "Was ist eine Private DNS Zone?"
    Azure erstellt automatisch eine DNS-Zone `privatelink.vaultcore.azure.net` und fügt einen A-Record ein:
    
    ```
    kv-aztraining-XXXX.vault.azure.net → 10.0.2.5  (private IP im VNet)
    ```
    
    VMs **im VNet** lösen den Key Vault auf die private IP auf.
    Anfragen **von außen** sehen weiterhin die öffentliche IP – bis wir den Public Access deaktivieren.

### Schritt 6: Review + Create

**Review + create** → **Create** (dauert ~1 Minute)

---

## Öffentlichen Zugriff auf Key Vault deaktivieren

Jetzt wo der Private Endpoint existiert, sperren wir den öffentlichen Zugang.

1. Gehe zu `kv-aztraining-XXXX` → **Networking**
2. Tab **Firewalls and virtual networks**
3. Setze **Public network access** auf **Disable**
4. Klicke **Apply**

!!! warning "Cloud Shell hat keinen Zugriff mehr"
    Nach dieser Änderung ist der Key Vault **nur noch aus dem VNet** erreichbar. Die Cloud Shell läuft nicht im VNet – du kannst also nicht mehr direkt von dort abfragen.
    
    **Ausnahme hinzufügen** (optional): Klicke auf **Add your client IP** um deine aktuelle IP-Adresse der Ausnahmeliste hinzuzufügen. Dann funktioniert der Cloud Shell-Zugriff noch.

---

## Private Endpoint testen

Verbinde dich via **Bastion** (Modul 10) mit `vm-web1` (im `frontend-subnet`, das mit der Private DNS Zone verknüpft ist).

### DNS-Auflösung testen (soll private IP zeigen)

```bash
nslookup kv-aztraining-XXXX.vault.azure.net
```

Erwartete Ausgabe:

```
Non-authoritative answer:
Name:    kv-aztraining-XXXX.privatelink.vaultcore.azure.net
Address: 10.0.2.5    ← Private IP aus deinem VNet!
```

!!! success "Private DNS-Auflösung funktioniert"
    Die VM löst den Key Vault auf eine private IP auf – kein Internet-Hop.

### Erreichbarkeit testen (soll HTTP 200/401 liefern)

```bash
curl -I https://kv-aztraining-XXXX.vault.azure.net
```

Du siehst HTTP 401 (Unauthorized) – der Key Vault ist erreichbar, aber du bist nicht authentifiziert. Das ist das gewünschte Verhalten.

### Von außen testen (soll scheitern)

In der Cloud Shell (außerhalb des VNets):

```bash
az keyvault secret list --vault-name kv-aztraining-XXXX
```

Fehler: `Public network access is disabled for this key vault.`

!!! success "Vollständige Netzwerkisolation"
    Der Key Vault ist aus dem Internet nicht mehr erreichbar. Nur Ressourcen im `vnet-training` können ihn über die private IP ansprechen.

---

## Private DNS Zone anzeigen

1. Suche im Portal nach **Private DNS zones**
2. Klicke auf `privatelink.vaultcore.azure.net`
3. Klicke links auf **Recordsets**

Du siehst den A-Record der von `kv-aztraining-XXXX` auf die private IP zeigt.

4. Klicke auf **Virtual network links**

Du siehst die Verknüpfung mit `vnet-training` – deswegen funktioniert die DNS-Auflösung von innen.

---

## Challenge

!!! question "Challenge: Private Endpoint für Storage Account"
    Erstelle einen Private Endpoint für den Storage Account aus Lernpfad 1 (oder erstelle einen neuen in `rg-netzwerk`):
    
    1. Target sub-resource: `blob`
    2. Private DNS Zone: `privatelink.blob.core.windows.net`
    3. Deaktiviere Public Access auf dem Storage Account
    4. Teste per `nslookup` aus vm-web1: zeigt die Auflösung auf eine private IP?

??? success "Hinweis"
    ```bash
    # Aus vm-web1 (im VNet):
    nslookup <dein-storageaccount>.blob.core.windows.net
    # Soll auflösen auf 10.0.x.x (private IP)
    
    # URL-Aufbau für Blob Private Endpoint DNS:
    # <account>.blob.core.windows.net → <account>.privatelink.blob.core.windows.net → 10.x.x.x
    ```
    
    **Wichtig beim Deaktivieren von Public Access:** Wenn du Cloud Shell-Zugriff auf den Storage Account brauchst, füge zuerst deine aktuelle IP unter **Networking → Firewalls and virtual networks → Add your client IP** hinzu.

---

Weiter zu [Modul 14 – Aufräumen Lernpfad 2](modul-14-aufräumen.md) →
