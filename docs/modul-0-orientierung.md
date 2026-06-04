# Modul 0 – Orientierung

## Lernziele

Nach diesem Modul kannst du:

- Das Azure Portal sicher navigieren
- Den Unterschied zwischen Subscription, Resource Group und Resource erklären
- Eine eigene Resource Group anlegen

---

## Was ist Azure?

Du kennst Hyper-V: du hast einen physischen Server, darauf läuft Hyper-V, und darauf erstellst du virtuelle Maschinen. Das alles liegt bei euch im Büro oder Serverraum.

**Azure ist dasselbe Prinzip – nur betreibt Microsoft die Hardware für dich.**

| Bei euch (On-Prem) | In Azure |
|--------------------|----------|
| Physischer Server im Serverraum | Microsofts Rechenzentrum (Region) |
| Hyper-V Hypervisor | Azure Compute |
| Virtuelle Maschine | Azure Virtual Machine |
| Netzwerk-Switch / VLAN | Azure Virtual Network (VNet) |
| Lokale Festplatte / NAS | Azure Storage |
| Active Directory | Microsoft Entra ID |
| Firewall / ACLs | Network Security Group (NSG) |

Der größte Unterschied zu Hyper-V: In Azure zahlst du nur für das, was du gerade nutzt – **Stunde für Stunde**. Wenn du eine VM ausschaltest, zahlst du fast nichts mehr. Es gibt keine Hardware-Investition im Voraus.

---

## Das Azure Portal

Das Azure Portal ist die grafische Oberfläche für alles in Azure. Du erreichst es unter:

**[portal.azure.com](https://portal.azure.com)**

### Beim ersten Login

1. Öffne [portal.azure.com](https://portal.azure.com) im Browser
2. Melde dich mit den Zugangsdaten an, die du von deinem Betreuer erhalten hast
3. Bestätige ggf. die Multi-Faktor-Authentifizierung (Authenticator-App oder SMS)

### Übersicht des Portals

Nach dem Login siehst du die **Startseite (Home)**. Die wichtigsten Bereiche:

- **Suchleiste oben** (Tastenkürzel: `G` + `/`): Finde jeden Azure-Dienst oder jede Ressource
- **Linkes Menü**: Favoriten und zuletzt genutzte Dienste
- **Hauptbereich**: Zuletzt verwendete Ressourcen und eine Kachel-Übersicht
- **Notifications** (Glocken-Symbol oben rechts): Status von laufenden Deployments

!!! tip "Tipp: Immer die Suchleiste nutzen"
    Azure hat über 200 verschiedene Dienste. Statt durch Menüs zu klicken, nutze **immer** die Suchleiste oben. Tippe z.B. "Virtual machines" oder "Storage accounts" – du findest den Dienst sofort.

---

## Die drei wichtigsten Konzepte

Bevor du anfängst, musst du drei Begriffe verstehen. Diese Hierarchie ist die Grundlage für alles in Azure.

### 1. Subscription (Abonnement)

Eine Subscription ist wie ein **Abrechnungskonto bei Azure**. Alle Kosten werden dieser Subscription in Rechnung gestellt. In einem Unternehmen kann es mehrere Subscriptions geben, z.B. eine für Entwicklung und eine für Produktion.

Für dieses Training nutzt du eine Subscription, die bereits vorbereitet wurde.

### 2. Resource Group (Ressourcengruppe)

Eine Resource Group ist ein **Ordner für deine Azure-Ressourcen**. Alles, was du heute baust, kommt in eine Resource Group. Das hat zwei große Vorteile:

- Du siehst auf einen Blick, welche Ressourcen zusammengehören
- Du kannst mit einem Klick alle Ressourcen auf einmal löschen (wichtig am Ende!)

### 3. Resource (Ressource)

Eine Resource ist ein **konkreter Azure-Dienst**: eine VM, ein Storage Account, eine Datenbank, eine KI-API usw. Jede Resource liegt in genau einer Resource Group.

```
Subscription  (Abrechnungskonto)
└── Resource Group: rg-praktikum  (dein "Ordner" für heute)
    ├── Virtual Machine: vm-training
    ├── Storage Account: stpraktikum123
    └── Computer Vision: cv-praktikum
```

### Region

Azure hat weltweit über 60 **Regionen** – das sind physische Rechenzentren. Beim Erstellen einer Ressource wählst du immer eine Region. Wähle eine, die nah bei dir liegt:

- **Germany West Central** (Frankfurt) – empfohlen für Deutschland
- **West Europe** (Amsterdam) – gut erreichbar, viele Dienste verfügbar

!!! warning "Alle Ressourcen in die gleiche Region"
    Erstelle heute alle Ressourcen in derselben Region. Das vermeidet unnötige Übertragungskosten und sorgt für die beste Performance.

---

## Aufgabe: Deine erste Resource Group erstellen

Jetzt legst du deinen "Arbeitsordner" für den heutigen Tag an.

### Schritt 1: Resource Groups öffnen

1. Klicke in der **Suchleiste** oben und tippe `Resource groups`
2. Klicke auf den Treffer **Resource groups** (Kategorie: Services)
3. Du siehst eine Liste aller Resource Groups in der Subscription

### Schritt 2: Neue Resource Group erstellen

1. Klicke oben links auf **+ Create**
2. Fülle das Formular aus:

| Feld | Wert |
|------|------|
| Subscription | die bereitgestellte Subscription (bereits ausgewählt) |
| Resource group | `rg-praktikum` |
| Region | `(Europe) Germany West Central` oder `(Europe) West Europe` |

3. Klicke auf **Review + create**
4. Überprüfe die Angaben kurz – alles richtig?
5. Klicke auf **Create**

### Schritt 3: Überprüfen

Nach wenigen Sekunden erscheint oben rechts eine Erfolgsmeldung. Klicke auf **Go to resource group** (oder suche erneut nach `rg-praktikum`).

Du siehst deine leere Resource Group. Oben links steht der Name, rechts davon die Region und die Subscription.

!!! success "Geschafft!"
    Du hast deine erste Azure Resource Group erstellt. Ab jetzt kommen alle Ressourcen des heutigen Tages hierhin.

---

Weiter zu [Modul 1 – Erste VM](modul-1-vm.md) →
