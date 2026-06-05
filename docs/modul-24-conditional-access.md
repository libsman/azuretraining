# Modul 24 – Conditional Access: Bedingter Zugriff und MFA

## Lernziele

Nach diesem Modul kannst du:

- Das Konzept von Conditional Access (Zero Trust) erklären
- Den Unterschied zwischen MFA-erzwingen und Conditional Access beschreiben
- Eine einfache Conditional Access-Policy im Portal erstellen und analysieren
- Security Defaults als Mindestschutz für Tenants erklären
- Named Locations und Sign-in Risk als Policy-Bedingungen beschreiben

---

## Hintergrund: Zero Trust – "Never Trust, Always Verify"

**On-Prem-Vergleich:** Im klassischen Sicherheitsmodell war das Unternehmensnetzwerk ein "sicheres Inneres" (Büro, VPN) und das Internet "gefährlich außen". Wer im Büronetz war, durfte auf Ressourcen zugreifen. **Zero Trust** dreht das um: Jeder Zugriff wird geprüft – egal ob aus dem Büro, aus dem Homeoffice oder vom Mobilgerät.

Microsoft beschreibt Zero Trust mit drei Prinzipien:

1. **Verify explicitly** – Immer authentifizieren und autorisieren, alle verfügbaren Datenpunkte nutzen
2. **Use least privilege access** – Nur minimale Rechte, Just-in-Time-Zugriff
3. **Assume breach** – Immer davon ausgehen, dass Angreifer schon im Netz sind → Segmentierung, Monitoring

**Conditional Access** ist das technische Werkzeug dafür in Entra ID.

---

## Was ist Conditional Access?

Conditional Access ist ein **Policy-Engine**: "Wenn Bedingung X zutrifft, dann Anforderung Y."

**Bedingungen (Conditions):**

| Signal | Beispiel |
|--------|---------|
| Benutzer / Gruppe | Nur für Admins |
| Anwendung | Nur beim Zugriff auf Azure Portal |
| Standort | Nur aus Deutschland |
| Gerätezustand | Nur Intune-verwaltete Geräte |
| Sign-in Risk | Wenn Login-Risikolevel "High" |
| Client App | Nur moderne Auth-Clients |

**Aktionen (Grant/Block):**

- Zugriff blockieren
- Zugriff erlauben wenn MFA bestanden
- Zugriff erlauben wenn Gerät compliant
- Zugriff erlauben wenn Terms of Use akzeptiert

!!! info "Entra ID P1 benötigt"
    Conditional Access-Richtlinien benötigen **Microsoft Entra ID P1** (oder P2). In vielen Microsoft 365 Business Premium / E3-Lizenzen ist P1 enthalten. Mit einem reinen Azure Free Account steht nur die grundlegende Version zur Verfügung.
    
    Für dieses Modul: schaue dir die Oberfläche an und erstelle eine Richtlinie im **Report-only**-Modus (kein Einfluss auf echte Logins).

---

## Security Defaults: Mindestschutz ohne P1

Falls kein P1 vorhanden ist, bietet Microsoft **Security Defaults** – ein vordefiniertes Sicherheitspaket:

- MFA für alle Admins erzwingen
- MFA für alle Benutzer registrieren
- Ältere Authentifizierungsprotokolle (Legacy Auth) blockieren
- Privilegierte Aktionen immer mit MFA schützen

### Security Defaults prüfen

1. **Microsoft Entra ID** → **Properties** (ganz unten im linken Menü)
2. Scrolle nach unten: **Manage Security defaults**
3. Du siehst ob Security Defaults aktiviert oder deaktiviert sind

!!! warning "Security Defaults vs. Conditional Access"
    Beides gleichzeitig zu aktivieren ist nicht möglich (bzw. führt zu Konflikten). Wenn du Conditional Access nutzt, deaktivierst du Security Defaults – dafür musst du aber eigene Richtlinien erstellen die denselben Schutz bieten.

---

## MFA für Benutzer konfigurieren

### Per-User MFA (Legacy-Methode, für kleine Tenants)

1. **Microsoft Entra ID** → **Users** → **All users**
2. Klicke oben auf **Per-user MFA** (Link erscheint im oberen Bereich)
3. Du siehst die MFA-Status aller Benutzer: Disabled / Enabled / Enforced
4. Wähle einen Benutzer → klicke rechts auf **Enable**

Der Benutzer wird beim nächsten Login aufgefordert MFA einzurichten (Authenticator App, SMS oder Anruf).

!!! info "Per-user MFA vs. Conditional Access MFA"
    - **Per-user MFA**: einfach, aber unflexibel – MFA immer, egal von wo oder womit
    - **Conditional Access MFA**: flexibel – MFA nur unter bestimmten Bedingungen (z.B. nicht im Firmennetz)
    
    Für neue Deployments: Conditional Access bevorzugen.

---

## Erste Conditional Access-Policy (Report-only)

### Schritt 1: Policy-Bereich öffnen

1. **Microsoft Entra ID** → **Protection** → **Conditional Access**
2. Tab **Policies**

Du siehst eine Liste der vorhandenen Policies (meist leer in Test-Tenants).

### Schritt 2: Neue Policy erstellen

Klicke **+ New policy**. Gib der Policy einen Namen: `MFA für Azure Portal (Report-only)`

**Assignments – Users:**
- Klicke auf **Users** → **Include** → **Select users and groups**
- Wähle deinen Testbenutzer `testuser@...` (nicht deinen eigenen Admin-Account – Gefahr der Aussperrung!)

**Assignments – Target resources:**
- Klicke auf **Target resources** → **Include** → **Select resources**
- Suche nach **`Microsoft Azure Management`** und wähle es aus

Dies umfasst: Azure Portal, Azure CLI, Azure PowerShell, ARM-API.

**Access controls – Grant:**
- Klicke auf **Grant**
- Wähle **Require multifactor authentication**
- Klicke **Select**

**Enable policy:**
- Stelle sicher: **Report-only** (nicht "On"!)
- Klicke **Create**

!!! warning "Report-only ist sicher"
    Im **Report-only**-Modus wird die Policy ausgewertet aber nicht durchgesetzt. Du siehst im Sign-in Log was passiert wäre – ohne echte Auswirkungen. Das ist ideal zum Testen.

### Schritt 3: Ergebnis analysieren

Nachdem sich jemand einloggt, kannst du unter **Sign-in logs** sehen was die Policy ausgewertet hat:

1. **Microsoft Entra ID** → **Monitoring** → **Sign-in logs**
2. Klicke auf einen Login-Eintrag
3. Tab **Conditional Access** – du siehst welche Policies ausgewertet wurden und was das Ergebnis wäre

---

## Named Locations: Trusted IPs definieren

Conditional Access kann auf Basis des Standorts (IP-Adresse oder GPS-Koordinate) entscheiden.

### Unternehmens-IP als vertrauenswürdig markieren

1. **Microsoft Entra ID** → **Protection** → **Conditional Access** → **Named locations**
2. **+ IP ranges location**

| Feld | Wert |
|------|------|
| Name | `Büro Berlin` |
| Mark as trusted location | ✅ |
| IP ranges | deine externe IP-Adresse (CIDR, z.B. `203.0.113.0/24`) |

Jetzt kannst du in einer Conditional Access-Policy sagen: MFA erzwingen, AUSSER wenn der Login von der vertrauenswürdigen Location kommt.

```
WENN:  Benutzer: Alle
       App: Azure Portal
       Standort: NOT "Büro Berlin"
DANN:  MFA erzwingen
```

---

## Sign-in Risk: KI-basierter Schutz

Entra ID **Identity Protection** (benötigt P2) analysiert jeden Login mit Machine Learning:

- Unbekanntes Gerät
- Ungewöhnlicher Standort
- Login aus zwei Ländern in kurzer Zeit ("impossible travel")
- Credentials in geleakten Datensätzen gefunden

Das Ergebnis ist ein **Risk Level**: Low / Medium / High

In einer Conditional Access-Policy:
```
WENN: Sign-in risk: High
DANN: Zugriff blockieren
```

Oder:
```
WENN: User risk: Medium oder höher
DANN: Passwort-Reset erzwingen
```

!!! info "Identity Protection – Entra ID P2"
    Risk-basiertes Conditional Access benötigt Entra ID P2. Mit P1 nur: Block, MFA, compliant device. Mit P2: zusätzlich Risk Level als Bedingung.

---

## Challenge

!!! question "Challenge: Policy für externe Logins"
    Erstelle eine Conditional Access-Policy (Report-only) mit folgender Logik:
    
    - **Gilt für**: Alle Benutzer
    - **Bedingung**: Login NICHT von einer Named Location "Vertrauenswürdige IPs"
    - **Aktion**: MFA erzwingen
    
    Erstelle zuerst eine Named Location mit deiner aktuellen IP (findest du unter [ifconfig.me](https://ifconfig.me)). Aktiviere die Policy im Report-only-Modus und analysiere im Sign-in Log ob dein eigener Login die Policy ausgelöst hätte.

??? success "Hinweis"
    1. **Named locations** → **+ IP ranges location** → deine IP/32 → Mark as trusted
    2. **New policy** → Users: Alle → Target resources: Azure Management
    3. **Conditions** → **Locations** → Include: Any location / Exclude: deine Named Location
    4. **Grant** → Require MFA
    5. **Report-only** → Create
    
    Im Sign-in Log siehst du den Status `Conditional Access: Report-only: Would not apply` (weil du von der trusted IP kommst) oder `Would apply` (wäre von außen).

---

Weiter zu [Modul 25 – Aufräumen Lernpfad 4](modul-25-aufräumen.md) →
