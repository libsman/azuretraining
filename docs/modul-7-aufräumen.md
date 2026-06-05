# Modul 7 – Aufräumen: Alle Ressourcen löschen

## Was macht dieser Schritt?

Du hast in diesem Lernpfad folgende Ressourcen in Azure erstellt – alle in der Resource Group **`rg-aztraining`**:

| Ressource | Erstellt in | Monatliche Kosten (ca.) |
|-----------|-------------|------------------------|
| VM `vm-training` + Netzwerk | Modul 1 | ~7 € (wenn läuft) |
| Storage Account + Static Website | Modul 2 | < 1 € |
| Computer Vision `cv-aztraining` | Modul 3 | 0 € (Free F0) |
| Budget Alert, CPU Alert | Modul 4 | 0 € |
| Web App `webapp-aztraining-XXXX` | Modul 5 | 0 € (Free F1) |
| Function App `func-aztraining-XXXX` | Modul 6 | 0 € (Flex Consumption) |

Die VM kostet ~0,009 €/Stunde solange sie läuft. Da du Auto-Shutdown in Modul 1 aktiviert hast, schaltet sie sich automatisch ab – aber gelöscht ist sie noch nicht. Die anderen Ressourcen sind im Free Tier und kosten nichts, sollten aber trotzdem bereinigt werden.

!!! info "Warum aufräumen?"
    Ressourcen löschen wenn man fertig ist, ist eine wichtige Gewohnheit in der Cloud. In Unternehmen können vergessene Ressourcen monatlich hunderte Euro kosten. Das nennt man **Cloud Waste** – verschwendetes Geld für nicht genutzte Dienste.

---

## Alles auf einmal löschen

Da alle Ressourcen in einer gemeinsamen Resource Group `rg-aztraining` liegen, reicht ein einziger Klick um alles zu entfernen.

### Schritt 1: Resource Group öffnen

1. Tippe in der Suchleiste **`rg-aztraining`** und klicke auf die Resource Group
2. Du siehst die Übersicht mit allen enthaltenen Ressourcen – das ist alles, was du in diesem Lernpfad gebaut hast

### Schritt 2: Löschen starten

1. Klicke oben auf **Delete resource group**
2. Ein Bestätigungs-Dialogfeld öffnet sich

### Schritt 3: Bestätigen

1. Tippe den Namen der Resource Group zur Bestätigung ein: **`rg-aztraining`**
2. Klicke auf **Delete**

!!! warning "Dieser Schritt ist endgültig"
    Alle Ressourcen in der Gruppe werden **dauerhaft gelöscht**: VM, Storage Account, Computer Vision, Web App, Function App, Netzwerk, Alerts – alles auf einmal. Das geht nicht rückgängig.

Azure löscht nun alle Ressourcen im Hintergrund. Das dauert je nach Anzahl der Ressourcen 1–5 Minuten. Du bekommst eine Benachrichtigung (Glocken-Symbol oben rechts) wenn der Vorgang abgeschlossen ist.

### Schritt 4: Bestätigung prüfen

1. Tippe in der Suchleiste **`Resource groups`** und klicke darauf
2. `rg-aztraining` sollte nicht mehr in der Liste erscheinen

✅ Fertig – keine laufenden Ressourcen, keine weiteren Kosten.

---

## 🎉 Herzlichen Glückwunsch!

Du hast alle wichtigen Konzepte des Cloud Computings in der Praxis umgesetzt:

| Was du gebaut hast | Technologie | Konzept |
|--------------------|------------|---------|
| Linux-Server mit Webserver | Azure VM + nginx | IaaS – Infrastructure as a Service |
| Statische Website ohne Server | Azure Blob Storage | Managed Storage |
| KI-Bilderkennung mit Python | Azure Computer Vision | AI as a Service |
| Kosten & Alerts überwachen | Azure Monitor / Cost Management | Cloud Governance |
| Web App ohne Serververwaltung | Azure App Service | PaaS – Platform as a Service |
| Serverlose API-Funktion | Azure Functions | Serverless / FaaS |

Du hast außerdem gelernt: Resource Groups, Subscriptions, SSH in der Cloud Shell, Python-Scripting gegen eine REST-API, Budget-Alerts, Tags, VM-Metriken und ZIP-Deployment.

Das entspricht dem, wofür ein Unternehmen früher eigene Server, Netzwerktechnik, Hardware-Investitionen und viel Vorlaufzeit gebraucht hätte – du hast es alles an einem Tag, über einen Browser, in Microsofts Cloud aufgebaut.

---

## Was kommt als nächstes?

Falls du mehr über Azure lernen möchtest:

- [**Microsoft Learn – Azure Fundamentals**](https://learn.microsoft.com/de-de/training/paths/azure-fundamentals/) – kostenloser, interaktiver Kurs direkt von Microsoft
- [**AZ-900 Zertifizierung**](https://learn.microsoft.com/de-de/certifications/azure-fundamentals/) – die Azure Grundlagenzertifizierung, perfekt für den Einstieg
- [**Azure Architecture Center**](https://learn.microsoft.com/de-de/azure/architecture/) – wie bauen echte Unternehmen ihre Cloud-Architekturen?
- [**Azure Pricing Calculator**](https://azure.microsoft.com/de-de/pricing/calculator/) – berechne die Kosten für eigene Architekturen

---

Weiter zu [Lernpfad 2 – Netzwerk & Sicherheit: Modul 8 – Azure Virtual Network](modul-8-vnet.md) →
