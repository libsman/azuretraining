# Modul 43 – Microsoft Sentinel: SIEM und Security-Events auswerten

## Lernziele

Nach diesem Modul kannst du:

- Microsoft Sentinel als Cloud-SIEM erklären und von Defender for Cloud abgrenzen
- Sentinel auf einem Log Analytics Workspace aktivieren
- Data Connectors für Azure Activity und Microsoft Entra ID einrichten
- Eingebaute Analytics Rules aktivieren um Bedrohungen automatisch zu erkennen
- Incidents untersuchen und im Incident-Dashboard navigieren
- Eine eigene Analytics Rule per KQL schreiben
- Hunting Queries für proaktive Bedrohungssuche nutzen

---

## Hintergrund: Was ist ein SIEM?

**On-Prem-Vergleich:** Große Unternehmen setzen SIEM-Lösungen (Security Information and Event Management) wie **Splunk**, **IBM QRadar** oder **ArcSight** ein. Diese sammeln Logs aus allen Quellen, korrelieren Ereignisse und schlagen Alarm bei verdächtigem Verhalten. Ein klassisches SIEM kostet Millionen €, braucht ein spezialisiertes Team und ist schwierig zu skalieren.

**Microsoft Sentinel** ist ein cloud-natives SIEM das auf dem Log Analytics Workspace aufbaut. Du bezahlst pro GB aufgenommener Daten – kein Server, keine Lizenz, automatisch skalierend.

**Sentinel vs. Defender for Cloud:**

| | Defender for Cloud | Microsoft Sentinel |
|--|-------------------|-------------------|
| Fokus | Azure-Ressourcen konfigurieren und schützen | Alle Logs korrelieren, Bedrohungen erkennen |
| Datenquellen | Azure-Ressourcen | Azure + On-Premises + AWS + Office 365 + ... |
| Use Case | "Sind meine Ressourcen sicher konfiguriert?" | "Werde ich gerade angegriffen?" |
| SIEM/SOAR | Nein | Ja (Incidents, Playbooks, Automation) |

!!! info "Kosten von Microsoft Sentinel"
    Sentinel berechnet ca. 2,46 €/GB für Daten die über die freien 5 GB des Log Analytics Workspace hinausgehen. Für das Training entstehen kaum Kosten wenn du Azure Activity Logs und Entra ID-Logs einbindest (niedriges Volumen).

---

## Schritt 1: Microsoft Sentinel aktivieren

Sentinel wird direkt auf einem Log Analytics Workspace aktiviert:

1. Suche nach **Microsoft Sentinel** im Portal
2. **+ Create** → wähle `law-training` als Workspace
3. **Add Microsoft Sentinel**

!!! tip "Bestehenden Workspace nutzen"
    Du verwendest den `law-training`-Workspace aus Modul 40. Alle Logs die du dort bereits konfiguriert hast stehen automatisch in Sentinel zur Verfügung.

---

## Schritt 2: Data Connectors einrichten

Data Connectors verbinden Datenquellen mit Sentinel:

1. **Sentinel** → **Data connectors**
2. Du siehst hunderte verfügbarer Connectors (Azure, AWS, Office 365, Palo Alto, ...)

### Azure Activity-Connector aktivieren

1. Suche nach **Azure Activity** → **Open connector page**
2. Klicke **Launch Azure Policy Assignment wizard**
3. Scope: deine Subscription
4. **Review + create** → **Create**

### Microsoft Entra ID-Connector aktivieren

1. Suche nach **Microsoft Entra ID** → **Open connector page**
2. Aktiviere **Sign-in Logs** und **Audit Logs**
3. Klicke **Connect**

!!! warning "Entra ID Connector benötigt P1/P2 Lizenz für Sign-in Logs"
    Sign-in Logs erfordern eine Entra ID P1 oder P2 Lizenz. Mit einem Free Tier Entra ID sind nur Audit Logs verfügbar. Das ist für das Training ausreichend.

---

## Schritt 3: Analytics Rules aktivieren

Analytics Rules überprüfen eingehende Logs und erstellen Incidents wenn Bedingungen zutreffen:

1. **Sentinel** → **Analytics**
2. Du siehst zwei Kategorien:
   - **Rule templates**: Eingebaute Regeln von Microsoft
   - **Active rules**: Von dir aktivierte Regeln

### Eingebaute Rule aktivieren

1. Klicke auf **Rule templates**
2. Filter nach **Data source: Azure Activity**
3. Suche nach **"Suspicious number of resource creation or deployment activities"**
4. Klicke **Create rule**

| Feld | Wert |
|------|------|
| Rule name | (vorausgefüllt) |
| Status | `Enabled` |
| Query scheduling | `Every 5 hours` |
| Lookup data from last | `1 day` |

5. **Next: Automated response** → **Next: Review** → **Create**

---

## Schritt 4: Eigene Analytics Rule schreiben

Erstelle eine eigene KQL-basierte Rule die Incidents erzeugt:

1. **Analytics** → **+ Create** → **Scheduled query rule**

| Feld | Wert |
|------|------|
| Name | `Mehrere fehlgeschlagene Logins` |
| Description | `Alarm wenn 5+ fehlgeschlagene Logins in 10 Minuten` |
| Severity | `Medium` |

2. **Set rule logic** → KQL-Abfrage:

```kusto
AzureActivity
| where TimeGenerated > ago(10m)
| where ActivityStatusValue == "Failure"
| where OperationNameValue contains "MICROSOFT.AUTHORIZATION"
| summarize FailedAttempts = count() by Caller, bin(TimeGenerated, 10m)
| where FailedAttempts >= 5
| project TimeGenerated, Caller, FailedAttempts
```

3. **Query scheduling**: Run every `5 minutes`, Lookup last `10 minutes`
4. **Alert threshold**: Generate alert when number of query results is **greater than** `0`
5. **Next: Incident settings** → **Incident creation**: `Enabled`
6. **Next: Review** → **Create**

---

## Schritt 5: Incidents untersuchen

Wenn eine Analytics Rule auslöst, wird ein **Incident** erstellt:

1. **Sentinel** → **Incidents**
2. Jeder Incident hat:
   - **Severity** und **Status** (New/Active/Closed)
   - **Entities**: betroffene Benutzer, IPs, Ressourcen
   - **Evidence**: welche Alerts den Incident ausgelöst haben
3. Klicke auf einen Incident → **Investigate**

### Investigation Graph

Der Investigation Graph zeigt Zusammenhänge:
- Welcher Benutzer war betroffen?
- Von welcher IP kam der Angriff?
- Welche anderen Aktivitäten gab es von diesem Benutzer?

---

## Schritt 6: Hunting Queries – proaktive Bedrohungssuche

Hunting ist proaktive Suche nach Bedrohungen ohne eine Regel die automatisch auslöst:

1. **Sentinel** → **Hunting**
2. Microsoft liefert hunderte vorgefertigte Hunting Queries
3. Filter nach `Data source: AzureActivity`
4. Klicke auf eine Query → **Run Query**
5. Interessante Ergebnisse → **Add bookmark** → später als Incident weiterverfolgen

### Eigene Hunting Query erstellen

1. **+ New query**

```kusto
// Ressourcen die in kurzer Zeit erstellt und wieder gelöscht wurden (mögliche Test-Infrastruktur für Angriffe)
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue has_any ("Microsoft.Resources/deployments/write", "Microsoft.Resources/deployments/delete")
| summarize 
    Creates = countif(OperationNameValue contains "write"),
    Deletes = countif(OperationNameValue contains "delete")
  by Caller, ResourceGroup
| where Creates > 0 and Deletes > 0
| project Caller, ResourceGroup, Creates, Deletes
```

---

## Schritt 7: SOAR – Automation mit Playbooks

**SOAR** (Security Orchestration, Automation and Response) automatisiert die Reaktion auf Incidents. Playbooks sind Azure Logic Apps die automatisch ausgeführt werden:

Beispiel-Szenario: Wenn ein Incident mit "Brute Force" erkannt wird → automatisch den Benutzer in Entra ID blockieren.

1. **Sentinel** → **Automation** → **+ Create** → **Automation rule**
2. Condition: `Incident title contains "Brute Force"`
3. Action: `Run playbook` (vorher ein Logic App Playbook erstellen)

!!! tip "Playbook-Vorlagen"
    Unter **Playbooks** → **+ Create** → wähle eine Community-Vorlage. Microsoft stellt fertige Playbooks für häufige Szenarien bereit (Benutzer blockieren, Ticket in ServiceNow erstellen, Teams-Nachricht senden).

---

## Challenge

!!! question "Challenge: Eigene Threat Detection"
    1. Erstelle eine Analytics Rule die auslöst wenn eine neue Resource Group erstellt wird
    2. Erstelle dann tatsächlich eine Resource Group (`rg-test-sentinel`) und prüfe ob ein Incident entsteht
    3. Lösche die Resource Group und schließe den Incident mit Kommentar

??? success "Hinweis"
    ```kusto
    AzureActivity
    | where TimeGenerated > ago(5m)
    | where OperationNameValue == "MICROSOFT.RESOURCES/SUBSCRIPTIONS/RESOURCEGROUPS/WRITE"
    | where ActivityStatusValue == "Success"
    | project TimeGenerated, Caller, ResourceGroup, OperationNameValue
    ```
    
    Schedule: Every `5 minutes`, Lookup `5 minutes`, Threshold: `> 0`

---

Weiter zu [Modul 44 – Aufräumen Lernpfad 7](modul-44-aufräumen.md) →
