# Modul 14 – Aufräumen Lernpfad 2

## Was macht dieser Schritt?

In Lernpfad 2 hast du folgende Ressourcen in der Resource Group **`rg-netzwerk`** erstellt:

| Ressource | Erstellt in | Monatliche Kosten (ca.) |
|-----------|-------------|------------------------|
| VNet `vnet-training` + Subnets | Modul 8 | 0 € |
| VMs `vm-web1`, `vm-web2` | Modul 8 | ~7 €/VM (wenn läuft) |
| NSG `nsg-frontend` | Modul 9 | 0 € |
| Azure Bastion `bastion-training` | Modul 10 | ~140 €/Monat (Basic) |
| VMs `vm-lb1`, `vm-lb2` | Modul 11 | ~7 €/VM (wenn läuft) |
| Load Balancer `lb-training` (Standard) | Modul 11 | ~4 €/Monat |
| Key Vault `kv-aztraining-XXXX` | Modul 12 | ~0 € |
| Private Endpoint `pe-keyvault` | Modul 13 | ~8 €/Monat |
| Private DNS Zone | Modul 13 | ~0,50 €/Monat |

!!! warning "Azure Bastion ist der teuerste Posten"
    Azure Bastion Basic kostet ~0,19 €/Stunde = ~140 €/Monat wenn er dauerhaft läuft. Falls du ihn nicht schon nach Modul 10 gelöscht hast: Er wird mit der Resource Group jetzt entfernt.

Da alle Ressourcen in `rg-netzwerk` liegen, reicht ein einziger Löschvorgang.

---

## Alles löschen

### Schritt 1: Resource Group öffnen

1. Tippe in der Suchleiste **`rg-netzwerk`** und klicke auf die Resource Group
2. Schaue die Ressourcen-Übersicht kurz durch – das ist alles, was du in Lernpfad 2 aufgebaut hast

### Schritt 2: Löschen starten

1. Klicke oben auf **Delete resource group**

### Schritt 3: Bestätigen

1. Tippe zur Bestätigung: **`rg-netzwerk`**
2. Klicke auf **Delete**

!!! warning "Endgültig und unwiderruflich"
    VMs, VNet, Bastion, Load Balancer, Key Vault, Private Endpoints, DNS-Zonen – alles wird dauerhaft gelöscht.

Azure löscht im Hintergrund. Bei 8–10 Ressourcen dauert das 3–8 Minuten. Die Glocke oben rechts zeigt eine Benachrichtigung wenn alles fertig ist.

### Schritt 4: Bestätigen

1. Tippe in der Suchleiste **`Resource groups`**
2. `rg-netzwerk` sollte nicht mehr in der Liste erscheinen

✅ Fertig – keine laufenden Kosten mehr für Lernpfad 2.

---

## 🎉 Lernpfad 2 abgeschlossen!

Du hast alle grundlegenden Netzwerk- und Sicherheitskonzepte von Azure in der Praxis umgesetzt:

| Was du gebaut hast | Technologie | Konzept |
|--------------------|------------|---------|
| Privates Netzwerk mit Subnetzen | Azure Virtual Network | Netzwerksegmentierung |
| Firewall-Regeln für Subnetze | Network Security Groups | Zugriffssteuerung auf Netzwerkebene |
| Sicherer Admin-Zugriff | Azure Bastion | Zero-Trust-Netzwerkzugang |
| Hochverfügbarer Webserver | Azure Load Balancer | Ausfallsicherheit & horizontale Skalierung |
| Sichere Passwortverwaltung | Azure Key Vault | Credential Management & Audit |
| Interne Azure-Dienste | Private Endpoints | Netzwerkisolation & Zero Public Exposure |

Du hast außerdem gelernt: cloud-init für automatisierte VM-Konfiguration, Health Probes, Private DNS Zones, RBAC für Key Vault, NSG-Debugging mit Effective Security Rules und den Unterschied zwischen Public und Private Endpoints.

---

## Was kommt als nächstes?

In Lernpfad 3 tauchen wir in Azure-Datenbankdienste ein – von relationalen Datenbanken über NoSQL bis hin zu Caching.

Weiter zu [Lernpfad 3 – Datenbanken: Modul 15 – Azure SQL Database](modul-15-sql.md) →
