# Azure Einstiegstraining

Willkommen! Dieses Training führt dich Schritt für Schritt durch die wichtigsten Konzepte von **Microsoft Azure** – praxisnah, selbstgeführt und kostenlos.

Du arbeitest im Microsoft-Umfeld als Windows-Admin, Azubi oder Praktikant – oder willst einfach Azure kennenlernen? Hier baust du echte Azure-Ressourcen auf, verstehst wie Cloud Computing funktioniert und sammelst Hands-on-Erfahrung die direkt im Berufsalltag hilft.

---

## Was du in Lernpfad 1 lernst

Nach Abschluss von Lernpfad 1 hast du folgendes in Azure selbst gebaut:

| | Was | Wo |
|---|---|---|
| 🖥️ | Eine echte Linux-VM mit Webserver | erreichbar über das Internet |
| 🌐 | Eine statische Website | ohne einen einzigen Server |
| 🤖 | Eine KI-Anwendung die Bilder versteht | mit ~20 Zeilen Python |
| 📊 | Monitoring & Kostenkontrolle | damit nichts aus dem Ruder läuft |
| 🚀 | Eine Python-Web-App | ohne VM – nur Code deployen |
| ⚡ | Eine serverlose API-Funktion | die nur bei Aufruf kostet |

## Was du in Lernpfad 2 lernst

| | Was | Wo |
|---|---|---|
| 🔗 | Ein privates Netzwerk mit Subnetzen | Azure Virtual Network |
| 🔥 | Firewall-Regeln für Subnetze | Network Security Groups |
| 🔐 | Sicherer Admin-Zugriff ohne offene Ports | Azure Bastion |
| ⚖️ | Hochverfügbarer Webserver | Azure Load Balancer |
| 🔑 | Secrets und Passwörter sicher speichern | Azure Key Vault |
| 🚧 | Azure-Dienste aus dem Internet nehmen | Private Endpoints |

## Was du in Lernpfad 3 lernst

| | Was | Wo |
|---|---|---|
| 🗄️ | Managed relationale Datenbank erstellen und abfragen | Azure SQL Database |
| 📄 | Flexible NoSQL-Dokumente speichern | Azure Cosmos DB |
| 🐘 | Open-Source-Datenbank als Service | Azure Database for PostgreSQL |
| ⚡ | Millisekundenanfragen mit In-Memory-Cache | Azure Cache for Redis |
| 🔒 | App mit Datenbank sicher verbinden | App Service + Key Vault + SQL |

---

## Übersicht der Module

| Modul | Thema | Dauer (ca.) |
|-------|-------|-------------|
| [Modul 0](modul-0-orientierung.md) | Orientierung: Portal, Konzepte, erste Resource Group | 45 Min |
| [Modul 1](modul-1-vm.md) | Erste VM: Ubuntu + nginx Webserver | 75 Min |
| [Modul 2](modul-2-storage.md) | Storage: Statische Website ohne Server | 60 Min |
| [Modul 3](modul-3-ai.md) | Azure KI: Bilderkennung mit Computer Vision | 90 Min |
| [Modul 4](modul-4-monitoring.md) | Monitoring: Kosten, Alerts, Tags | 30 Min |
| [Modul 5](modul-5-appservice.md) | App Service: Web App ohne VM deployen (PaaS) | 60 Min |
| [Modul 6](modul-6-functions.md) | Azure Functions: Serverless Computing | 45 Min |
| [Modul 7](modul-7-aufräumen.md) | Aufräumen Lernpfad 1 | 10 Min |
| [Modul 8](modul-8-vnet.md) | Virtual Network: Subnetze und Adressräume | 60 Min |
| [Modul 9](modul-9-nsg.md) | Network Security Groups: Traffic filtern | 45 Min |
| [Modul 10](modul-10-bastion.md) | Azure Bastion: Sicherer VM-Zugriff ohne Public IP | 30 Min |
| [Modul 11](modul-11-loadbalancer.md) | Load Balancer: Traffic auf mehrere Server verteilen | 60 Min |
| [Modul 12](modul-12-keyvault.md) | Key Vault: Secrets und API-Keys sicher speichern | 45 Min |
| [Modul 13](modul-13-privateendpoints.md) | Private Endpoints: Dienste intern erreichbar machen | 45 Min |
| [Modul 14](modul-14-aufräumen.md) | Aufräumen Lernpfad 2 | 10 Min |
| [Modul 15](modul-15-sql.md) | Azure SQL Database: Managed Datenbank erstellen und abfragen | 75 Min |
| [Modul 16](modul-16-cosmosdb.md) | Azure Cosmos DB: NoSQL-Datenbank für flexible Daten | 60 Min |
| [Modul 17](modul-17-postgresql.md) | Azure Database for PostgreSQL: Open-Source-DB als Service | 60 Min |
| [Modul 18](modul-18-redis.md) | Azure Cache for Redis: In-Memory-Cache für schnelle Apps | 45 Min |
| [Modul 19](modul-19-app-datenbank.md) | App + Datenbank sicher verbinden | 60 Min |
| [Modul 20](modul-20-aufräumen.md) | Aufräumen Lernpfad 3 | 10 Min |

!!! tip "Kein Stress mit der Zeit"
    Die Zeitangaben sind Orientierungshilfen, kein Zwang. Nimm dir so lange wie du brauchst. Wenn du nicht weiterkommst, hilft oft ein Blick in die offizielle [Microsoft Learn Dokumentation](https://learn.microsoft.com/de-de/azure/).

---

## Voraussetzungen

- [ ] Zugangsdaten für [portal.azure.com](https://portal.azure.com) (Azure Free Account oder von deinem Ausbilder)
- [ ] Ein moderner Browser (Edge oder Chrome empfohlen)
- [ ] Nichts weiter — alles andere erklären die Module

!!! info "Kosten"
    Mit einem Azure Free Account entstehen für Lernpfad 1 kaum Kosten – die meisten Dienste haben einen kostenlosen Tarif. Denke daran, die Ressourcen nach dem Training in Modul 7 zu löschen. In Modul 4 lernst du, wie man Kosten im Blick behält.

---

## Los geht's!

Starte mit **[Modul 0 – Orientierung](modul-0-orientierung.md)** →
