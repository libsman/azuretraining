# Modul 4 – Monitoring & Kosten

## Lernziele

Nach diesem Modul kannst du:

- Deine aktuellen Azure-Kosten in der Cost Analysis einsehen
- Einen Budget-Alert einrichten, der dich per E-Mail warnt
- Ressourcen mit Tags beschriften
- VM-Metriken (CPU, Netzwerk) in einem Diagramm ansehen

---

## Hintergrund: Warum Monitoring in der Cloud wichtig ist

On-Prem läuft ein Server so lange bis er kaputt geht – egal ob du ihn benutzt oder nicht, die Kosten (Strom, Miete, Abschreibung) fallen immer an.

In der Cloud ist das anders: **Du zahlst nur für das, was gerade läuft.** Das ist ein Vorteil – aber auch eine Gefahr. Wenn du vergisst, eine VM auszuschalten oder ein teuren Dienst versehentlich startest, läuft die Uhr weiter.

Gutes Cloud-Monitoring bedeutet:

- **Transparenz**: Ich weiß, was gerade läuft und was es kostet
- **Frühwarnung**: Ich bekomme eine Benachrichtigung bevor die Kosten aus dem Ruder laufen
- **Ordnung**: Ich kann Ressourcen bestimmten Projekten oder Teams zuordnen

---

## Cost Analysis: Was kostet mein Training?

### Schritt 1: Cost Management öffnen

1. Tippe in der Suchleiste **`Cost Management`** und klicke auf den Dienst
2. Klicke links im Menü auf **Cost analysis**

### Schritt 2: Kosten ansehen

Du siehst ein Diagramm mit den Kosten der aktuellen Periode.

1. Stelle oben rechts den Zeitraum auf **This month**
2. Klicke auf **Group by** → wähle **Service name**

Jetzt siehst du aufgeschlüsselt, welcher Azure-Dienst wie viel kostet.

!!! info "Heute noch fast 0 €"
    Da du heute alles frisch erstellt hast, sind die Kosten bisher minimal. Erst nach 24h werden die Ressourcen sichtbar im Cost Dashboard. Ein Standard_B1s VM kostet ca. **0,009 €/Stunde** – das sind weniger als 10 Cent pro Tag.

---

## Budget Alert einrichten

Ein Budget Alert schickt dir automatisch eine E-Mail, wenn deine Ausgaben einen Schwellenwert erreichen. Das ist dein Sicherheitsnetz.

### Schritt 1: Budgets öffnen

1. Bleibe im **Cost Management** Bereich
2. Klicke links auf **Budgets**
3. Klicke oben auf **+ Add**

### Schritt 2: Budget konfigurieren

| Feld | Wert |
|------|------|
| Name | `budget-praktikum` |
| Reset period | `Monthly` |
| Creation date | (heute, bereits eingetragen) |
| Expiration date | (Ende nächsten Monats) |
| Amount | `20` |

Klicke auf **Next: Alerts >**

### Schritt 3: Alert-Bedingung hinzufügen

1. Klicke auf **+ Add alert condition**
2. Konfiguriere:

| Feld | Wert |
|------|------|
| Alert condition | `Actual` |
| % of budget | `80` |
| Alert recipients (email) | deine E-Mail-Adresse |

3. Klicke auf **Create**

!!! success "Sicherheitsnetz aktiv"
    Du bekommst ab jetzt eine E-Mail sobald 80% von 20 € (= 16 €) verbraucht sind. So kannst du keine Überraschungen erleben.

---

## Tags: Ressourcen beschriften

Tags sind Schlüssel-Wert-Paare, die du an Azure-Ressourcen anhängen kannst. Sie helfen dir:

- Ressourcen nach Projekt oder Team zu gruppieren
- Kosten nach Tags aufzuschlüsseln (z.B. "Was hat Projekt X insgesamt gekostet?")
- Ordnung zu halten wenn man viele Ressourcen hat

In echten Unternehmen sind Tags Pflicht – die IT-Abteilung kann so sehen, welches Team welche Kosten verursacht.

### Schritt 1: Resource Group öffnen

1. Suche im Portal nach `rg-praktikum` und klicke auf die Resource Group
2. Klicke links im Menü auf **Tags**

### Schritt 2: Tags hinzufügen

Klicke jeweils in die Felder Name und Wert und füge folgende Tags hinzu:

| Name | Wert |
|------|------|
| `owner` | dein Name |
| `projekt` | `azure-training` |
| `umgebung` | `dev` |

Klicke auf **Apply**.

!!! tip "Tags auf Resource Group = Tags auf allen Ressourcen"
    Wenn du Tags auf einer Resource Group setzt, werden sie in Cost Analysis auf alle enthaltenen Ressourcen angewendet. Du musst nicht jede VM, jeden Storage Account etc. einzeln taggen.

---

## VM-Metriken: CPU und Netzwerk live sehen

Azure sammelt automatisch Metriken für alle laufenden Ressourcen: CPU-Auslastung, Speicher, Netzwerkdurchsatz, Festplatten-I/O und mehr.

### Schritt 1: Zur VM navigieren

1. Suche im Portal nach `vm-training` und klicke auf deine VM
2. Links im Menü: **Monitoring** → **Metrics**

### Schritt 2: Metriken hinzufügen

1. Im Diagramm-Editor:
    - **Scope**: `vm-training` (bereits ausgewählt)
    - **Metric Namespace**: `Virtual Machine Host`
    - **Metric**: wähle `Percentage CPU`
2. Du siehst ein Diagramm der CPU-Auslastung der letzten Zeit

Füge weitere Metriken hinzu:

1. Klicke auf **+ Add metric**
2. Wähle `Network In Total` → Add
3. Klicke nochmal auf **+ Add metric**
4. Wähle `Network Out Total` → Add

Jetzt siehst du drei Kurven im selben Diagramm.

### Schritt 3: CPU-Alert erstellen

Jetzt richten wir einen automatischen Alert ein, der sich meldet wenn die VM unter Last kommt.

1. Klicke oben im Metrics-Bereich auf **+ New alert rule**
2. Das Formular öffnet sich, die Ressource `vm-training` ist bereits eingetragen
3. Klicke auf **+ Add condition**
4. Wähle **Percentage CPU** aus der Liste
5. Scrolle im Konfigurations-Bereich nach unten und stelle ein:

| Feld | Wert |
|------|------|
| Threshold type | `Static` |
| Aggregation type | `Average` |
| Operator | `Greater than` |
| Threshold value | `80` |

6. Klicke auf **Done**

**Action Group erstellen** (wer bekommt die E-Mail?):

7. Scrolle nach unten zum Abschnitt **Actions**
8. Klicke auf **+ Create action group**
9. Fülle aus:

| Feld | Wert |
|------|------|
| Action group name | `ag-praktikum` |
| Display name | `praktikum` |
| Resource group | `rg-praktikum` |

10. Klicke auf den Tab **Notifications**
11. Wähle bei **Notification type**: `Email/SMS message/Push/Voice`
12. Trage deine E-Mail-Adresse ein und klicke **OK**
13. Gib einen Namen ein, z.B. `email-benachrichtigung`
14. Klicke durch bis **Review + create** → **Create**

**Alert Rule abschließen:**

15. Zurück im Alert Rule Formular: trage unter **Alert rule name** ein: `alert-hohe-cpu`
16. Klicke auf **Create**

!!! success "Alles überwacht!"
    Du bekommst jetzt eine E-Mail wenn die CPU deiner VM über 80% steigt. In echten Produktionsumgebungen würde man auf solche Alerts z.B. automatisch weitere VMs starten (Auto-Scaling).

---

Weiter zu [Modul 5 – Azure App Service](modul-5-appservice.md) →
