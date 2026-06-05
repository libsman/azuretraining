# Modul 42 – Microsoft Defender for Cloud: Security Posture und Alerts

## Lernziele

Nach diesem Modul kannst du:

- Microsoft Defender for Cloud als Cloud Security Posture Management (CSPM) erklären
- Den **Secure Score** interpretieren und Empfehlungen abarbeiten
- Defender Plans (CSPM kostenlos vs. kostenpflichtige Workload-Schutzpläne) unterscheiden
- Security Alerts verstehen und auf Incidents reagieren
- Just-In-Time (JIT) VM-Zugriff aktivieren
- Regulatory Compliance-Dashboards nutzen (ISO 27001, CIS Benchmarks)

---

## Hintergrund: Security Posture Management

**On-Prem-Vergleich:** On-Premises gibt es Vulnerability Scanner wie Nessus oder OpenVAS die prüfen ob Server gepatcht und korrekt konfiguriert sind. Ergebnisse kommen als Berichte – die Behebung ist manuelle Arbeit. In Azure übernimmt **Microsoft Defender for Cloud** diese Rolle automatisch und kontinuierlich.

**Defender for Cloud hat zwei Hauptbereiche:**

| Bereich | Beschreibung |
|---------|-------------|
| **CSPM** (Cloud Security Posture Management) | Prüft Konfiguration, gibt Empfehlungen, berechnet Secure Score |
| **CWP** (Cloud Workload Protection) | Erkennt aktive Angriffe, schützt VMs/Container/Datenbanken in Echtzeit |

Der **Secure Score** ist eine Prozentzahl (0–100%) die zeigt wie gut deine Azure-Umgebung konfiguriert ist. Jede umgesetzte Empfehlung erhöht den Score.

!!! info "Was ist kostenlos?"
    **CSPM Foundational** (Secure Score + Empfehlungen) ist kostenlos für alle Azure-Subscriptions. Die erweiterten Defender-Pläne (für Server, Datenbanken, Container usw.) kosten extra (z. B. Defender for Servers ca. 0,02 €/Server-Stunde).

---

## Schritt 1: Defender for Cloud öffnen

1. Suche im Portal nach **Microsoft Defender for Cloud**
2. Du siehst sofort dein **Secure Score** – dieser ist bereits aktiv, ohne dass du etwas konfigurieren musst

!!! tip "Secure Score ohne eigene Ressourcen"
    Auch wenn du nur wenige Ressourcen hast, findest du hier sinnvolle Empfehlungen. Defender scannt die gesamte Subscription.

---

## Schritt 2: Secure Score und Empfehlungen

### Secure Score verstehen

Der Secure Score basiert auf **Sicherheitskontrollen** (Control Groups). Jede Kontrolle enthält mehrere Empfehlungen:

- Kontrolle komplett erfüllt → volle Punkte
- Kontrolle teilweise erfüllt → anteilige Punkte
- Kontrolle gar nicht erfüllt → 0 Punkte

### Empfehlungen abarbeiten

1. **Defender for Cloud** → **Recommendations**
2. Sortiere nach **Severity** (Critical, High, Medium, Low)
3. Klicke auf eine Empfehlung → du siehst:
   - Warum sie wichtig ist
   - Welche Ressourcen betroffen sind
   - Einen **Quick Fix**-Button (wenn verfügbar)

**Typische einfache Empfehlungen für das Training:**

| Empfehlung | Fix |
|------------|-----|
| MFA sollte für Accounts mit Owner-Rechten aktiviert sein | MFA im Entra-Account aktivieren |
| Storage Accounts sollten HTTPS erzwingen | `az storage account update --https-only true` |
| App Service sollte nur HTTPS nutzen | Im App Service: TLS/SSL Settings → HTTPS Only: On |
| VMs sollten System Updates installiert haben | Azure Update Manager aktivieren |

---

## Schritt 3: Security Alerts

Security Alerts werden ausgelöst wenn Defender for Cloud verdächtiges Verhalten erkennt:

1. **Defender for Cloud** → **Security alerts**
2. Jeder Alert hat:
   - **Severity** (Informational, Low, Medium, High)
   - **Affected resource**
   - **Description**: was wurde erkannt
   - **Remediation steps**: wie beheben

!!! info "Alerts im Free Tier"
    Im kostenlosen CSPM-Tier sind nur einige grundlegende Alerts verfügbar. Umfassender Schutz (z. B. gegen Malware auf VMs) erfordert die kostenpflichtigen Defender-Pläne.

---

## Schritt 4: Just-In-Time VM-Zugriff (JIT)

JIT ist eine der wirkungsvollsten Sicherheitsmaßnahmen: SSH/RDP-Ports werden standardmäßig **geschlossen** und nur auf Anfrage für eine begrenzte Zeit geöffnet.

**Voraussetzung:** Defender for Servers Plan (kostenpflichtig). Im Training erkläre ich das Konzept – du kannst es ohne Kosten nachvollziehen indem du dir die Konfiguration anschaust.

### Konzept

Ohne JIT: Port 22 (SSH) oder 3389 (RDP) ist dauerhaft offen → Brute-Force-Angriffe möglich.

Mit JIT:
```
Benutzer beantragt Zugriff → Defender öffnet Port für X Stunden nur für die IP des Benutzers → 
Port automatisch wieder geschlossen
```

### JIT aktivieren

1. **Defender for Cloud** → **Workload protections** → **Just-in-time VM access**
2. Klicke auf eine VM → **Enable JIT on 1 VM**
3. Konfiguriere erlaubte Ports (22, 3389) und maximale Zugriffszeit

### Zugriff anfordern

```bash
# Eigene öffentliche IP holen
MY_IP=$(curl -s https://api.ipify.org)

# JIT-Zugriff für 2 Stunden anfordern
az security jit-policy initiate \
  --resource-group rg-monitoring \
  --vm-name DEIN-VM-NAME \
  --ports "[{number:22,allowedSourceAddressPrefix:'$MY_IP',endTimeUtc:'$(date -u -d '+2 hours' +%Y-%m-%dT%H:%M:%SZ)'}]"
```

---

## Schritt 5: Regulatory Compliance

Defender for Cloud bewertet deine Umgebung gegen bekannte Standards:

1. **Defender for Cloud** → **Regulatory compliance**
2. Standardmäßig aktiviert: **Microsoft Cloud Security Benchmark (MCSB)**
3. Klicke auf einen Standard → du siehst welche Controls erfüllt/nicht erfüllt sind

Weitere Standards hinzufügen:

1. **Manage compliance policies** → **+ Add more standards**
2. Wähle z. B. **CIS Microsoft Azure Foundations Benchmark** (kostenlos)

!!! tip "Compliance-Report exportieren"
    **Regulatory compliance** → **Download report** → PDF-Bericht für Audits.

---

## Schritt 6: Defender-Pläne für die Subscription konfigurieren

Im Training nur anschauen – **nicht aktivieren** um Kosten zu vermeiden:

1. **Defender for Cloud** → **Environment settings** → deine Subscription
2. Du siehst alle verfügbaren Pläne:

| Plan | Schützt | Preis (ca.) |
|------|---------|-------------|
| Defender for Servers | VMs | 0,02 €/Std./Server |
| Defender for App Service | App Services | 15 €/Monat |
| Defender for SQL | Azure SQL | 15 €/Monat je DB-Server |
| Defender for Storage | Storage Accounts | 10 €/Monat |
| Defender for Containers | AKS/ACR | 7 €/vCore/Monat |

---

## Challenge

!!! question "Challenge: Secure Score um 5 Punkte erhöhen"
    1. Öffne **Recommendations** in Defender for Cloud
    2. Wähle 2–3 Empfehlungen mit **Quick Fix** die du in deiner Umgebung umsetzen kannst
    3. Führe den Quick Fix aus und beobachte wie der Secure Score steigt
    
    Dokumentiere: Welche Empfehlungen du umgesetzt hast und wie sich der Score verändert hat.

??? success "Hinweis"
    Gute Kandidaten für Quick Fixes ohne Kosten:
    - `Storage accounts should use customer-managed key` → Oder einfacher: `Secure transfer to storage accounts should be enabled` → Quick Fix: HTTPS aktivieren
    - `App Service apps should use the latest HTTP version` → Quick Fix direkt im Portal
    - `MFA should be enabled on accounts with write permissions` → In Entra ID → Security → MFA aktivieren

---

Weiter zu [Modul 43 – Microsoft Sentinel: SIEM und Security-Events auswerten](modul-43-sentinel.md) →
