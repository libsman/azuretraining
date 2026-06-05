# Modul 21 – Microsoft Entra ID: Benutzer, Gruppen und Organisationsstruktur

## Lernziele

Nach diesem Modul kannst du:

- Den Unterschied zwischen Microsoft Entra ID und Active Directory erklären
- Benutzer und Gruppen im Entra ID-Portal anlegen und verwalten
- Gastbenutzer einladen (B2B-Zusammenarbeit)
- Den Zusammenhang zwischen Entra ID, Azure-Subscription und RBAC verstehen
- Administrative Units als Organisationsstruktur beschreiben

---

## Hintergrund: Active Directory vs. Microsoft Entra ID

**On-Prem-Vergleich:** In fast jedem Windows-Unternehmen gibt es ein **Active Directory (AD)** – den Verzeichnisdienst für Benutzer, Computer, Gruppen und Gruppenrichtlinien. Domäne joinen, Logins über Kerberos, `gpupdate /force` – das kennt jeder Windows-Admin.

**Microsoft Entra ID** (früher: Azure Active Directory / Azure AD) ist das Cloud-Äquivalent – aber kein direktes Upgrade:

| | Active Directory (on-prem) | Microsoft Entra ID (Cloud) |
|--|---------------------------|---------------------------|
| Protokolle | Kerberos, NTLM, LDAP | OAuth 2.0, OpenID Connect, SAML |
| Domäne | `firma.local` | `firma.onmicrosoft.com` |
| Zugriff auf | Windows-PCs, Server, Fileserver | Cloud-Apps, Azure, Microsoft 365 |
| Gruppenrichtlinien | ✅ GPO | ❌ (dafür: Intune/Autopilot) |
| Geräteverwaltung | Domänen-Join | Entra-Join / Hybrid-Join |
| Preis | Inklusive in Windows Server | Free-Tier + Premium P1/P2 |

!!! info "Umbenennung: Azure AD → Entra ID"
    Microsoft hat 2023 **Azure Active Directory** in **Microsoft Entra ID** umbenannt. Im Portal und in vielen Dokumentationen findest du noch beide Namen – sie bezeichnen denselben Dienst. Die URL `aad.portal.azure.com` leitet auf das neue Entra ID-Portal weiter.

**Was Entra ID für Azure bedeutet:**

Jede Azure-Subscription ist einem Entra ID-Tenant zugeordnet. Wenn du dich am Azure Portal anmeldest, authentifizierst du dich gegen Entra ID. RBAC-Rollen (wer darf was in Azure) werden auf Entra-Identitäten vergeben.

---

## Entra ID-Portal öffnen

### Option A: Über das Azure Portal

1. Tippe in der Suchleiste **`Microsoft Entra ID`** und klicke auf den Dienst
2. Du siehst das Overview-Dashboard deines Tenants

### Option B: Direktlink

Öffne **[entra.microsoft.com](https://entra.microsoft.com)** – das eigenständige Entra Admin Center.

!!! info "Tenant und Tenantname"
    Dein Tenant hat einen eindeutigen Namen: `XXXX.onmicrosoft.com`. Im Overview unter **Overview** siehst du:
    - **Tenant ID** (UUID, z.B. `a1b2c3d4-...`)
    - **Primary domain** (z.B. `meinefirma.onmicrosoft.com`)
    - **Tenant type** (z.B. `Microsoft Entra ID`)

---

## Benutzer anlegen

### Schritt 1: Benutzerübersicht öffnen

1. Im Entra ID-Portal: links im Menü auf **Users** → **All users**
2. Du siehst deinen eigenen Account (und ggf. weitere Standard-Accounts)

### Schritt 2: Neuen Benutzer erstellen

Klicke oben auf **+ New user** → **Create new user**:

| Feld | Wert |
|------|------|
| User principal name | `testuser@XXXX.onmicrosoft.com` |
| Display name | `Test User` |
| Password | Auto-generate (notiere das Initialpasswort!) |

Klicke auf **Review + create** → **Create**.

!!! info "User Principal Name (UPN)"
    Der UPN ist der Benutzername im Cloud-Format: `vorname.nachname@domäne.com`. Er entspricht der E-Mail-Adresse und wird für alle Azure-Anmeldungen verwendet.

### Schritt 3: Benutzer anzeigen

1. Gehe zurück zu **All users**
2. Klicke auf `Test User` um das Profil zu öffnen
3. Schau dir die Tabs an: **Profile**, **Assigned roles**, **Groups**, **Applications**, **Devices**

---

## Gruppen anlegen und Mitglieder zuweisen

Gruppen sind in Entra ID (wie im AD) der zentrale Mechanismus um Berechtigungen zu verwalten: du gibst der Gruppe eine Rolle, und alle Mitglieder erben sie automatisch.

### Gruppe erstellen

1. Links im Menü: **Groups** → **All groups**
2. Klicke auf **+ New group**

| Feld | Wert |
|------|------|
| Group type | `Security` |
| Group name | `grp-azure-leser` |
| Group description | `Lesezugriff auf Azure-Ressourcen` |
| Membership type | `Assigned` |

Klicke auf **No members selected** → Suche nach `Test User` → Auswählen → **Create**.

!!! info "Security vs. Microsoft 365 Gruppen"
    | Typ | Wofür |
    |-----|-------|
    | **Security** | Azure RBAC, Berechtigungen, Conditional Access |
    | **Microsoft 365** | E-Mail-Verteiler, Teams, SharePoint |
    
    Für Azure-Berechtigungen immer **Security** wählen.

### Dynamische Mitgliedschaft (optional)

Statt Mitglieder manuell zuzuweisen, kann Entra ID Gruppen automatisch befüllen:

- Membership type: **Dynamic User**
- Regel: `user.department -eq "IT"`

Dann werden alle Benutzer mit Abteilung "IT" automatisch Mitglied. (**Benötigt Entra ID P1-Lizenz**)

---

## Gastbenutzer einladen (B2B)

B2B (Business-to-Business) ist das Entra-Konzept für externe Zusammenarbeit: du lädst jemanden mit seiner eigenen E-Mail-Adresse ein – er bekommt Zugriff auf deine Azure-Ressourcen, braucht keinen Account in deinem Tenant.

### Gastbenutzer hinzufügen

1. **Users** → **All users** → **+ New user** → **Invite external user**

| Feld | Wert |
|------|------|
| Email | eine externe E-Mail-Adresse |
| Display name | Name des Gastes |
| Message | optionale Nachricht in der Einladungsmail |

Klicke auf **Invite**.

Der Gast erhält eine E-Mail mit einem Link. Nach Annahme erscheint er in **All users** mit dem Typ `#EXT#` im UPN, z.B. `name_extern.com#EXT#@deinertenant.onmicrosoft.com`.

!!! tip "Gastbenutzer und Azure-Zugriff"
    Gastbenutzer können RBAC-Rollen bekommen wie normale Benutzer. Typisches Szenario: Externe Dienstleister bekommen Leserechte auf eine Resource Group – ohne eigenen Account im Unternehmen.

---

## Benutzer in Entra ID vs. RBAC in Azure

Ein häufiges Missverständnis: **Entra ID verwaltet Identitäten, RBAC verwaltet Berechtigungen auf Azure-Ressourcen.**

```
Entra ID                    Azure RBAC
─────────────────           ─────────────────────────────
Benutzer anlegen     →      Rolle vergeben (wer darf was)
Gruppen erstellen    →      Gruppe bekommt Rolle auf Scope
Passwort resettten   →      Kein Einfluss auf Azure-Rechte
```

Beispiel: Du kannst einen Benutzer in Entra ID anlegen – er hat noch **keinerlei** Zugriff auf Azure. Erst wenn du ihm eine RBAC-Rolle gibst (z.B. **Reader** auf eine Resource Group), kann er etwas sehen.

Das lernst du im Detail in Modul 23.

---

## Challenge

!!! question "Challenge: Gruppe mit RBAC-Rolle"
    1. Erstelle eine Gruppe `grp-rg-aztraining-reader`
    2. Füge deinen Testbenutzer `testuser@...` als Mitglied hinzu
    3. Navigiere zu einer Resource Group (z.B. `rg-aztraining` falls noch vorhanden, sonst erstelle eine neue)
    4. Weise der **Gruppe** (nicht dem einzelnen Benutzer!) die RBAC-Rolle **Reader** zu
    5. Wo im Portal machst du das?

??? success "Hinweis"
    RBAC-Rollen werden nicht in Entra ID vergeben, sondern direkt auf der Azure-Ressource:
    
    1. Resource Group öffnen → **Access control (IAM)** → **+ Add** → **Add role assignment**
    2. Role: `Reader`
    3. Members: **Select members** → Suche nach `grp-rg-aztraining-reader`
    4. **Review + assign**
    
    Jetzt hat jedes aktuelle und zukünftige Mitglied dieser Gruppe Leserechte auf die Resource Group.

---

Weiter zu [Modul 22 – Managed Identity: Apps ohne Passwörter authentifizieren](modul-22-managed-identity.md) →
