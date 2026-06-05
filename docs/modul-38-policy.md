# Modul 38 – Azure Policy & Resource Locks: Compliance automatisieren

## Lernziele

Nach diesem Modul kannst du:

- Azure Policy als Compliance-Werkzeug erklären und von RBAC abgrenzen
- Eingebaute Policy-Definitionen im Portal finden und zuweisen
- Eine eigene Policy-Definition schreiben (Deny + Audit-Effekte)
- Policy Initiatives (Gruppen von Policies) verstehen
- Resource Locks nutzen um versehentliches Löschen zu verhindern
- Den Unterschied zwischen Policy und Resource Lock beschreiben

---

## Hintergrund: Governance in Azure

**On-Prem-Vergleich:** In Active Directory gibt es Gruppenrichtlinien (GPOs) die erzwingen wie Computer und Benutzer konfiguriert sind – z.B. "Bildschirmschoner nach 10 Minuten", "Kein USB erlaubt". Azure Policy ist das Äquivalent für Azure-Ressourcen: "Alle Storage Accounts müssen verschlüsselt sein", "VMs dürfen nur in West Europe erstellt werden".

**Azure Policy vs. RBAC:**

| | Azure Policy | Azure RBAC |
|--|-------------|-----------|
| Kontrolliert | **Was** Ressourcen konfiguriert sein müssen | **Wer** Ressourcen verwalten darf |
| Verhindert | Falsch konfigurierte Ressourcen | Unbefugten Zugriff |
| Beispiel | "Storage Accounts müssen HTTPS haben" | "Anna darf keine VMs erstellen" |
| Wirkung | Auf Ressourcen-Eigenschaften | Auf Aktionen des Benutzers |

Beide ergänzen sich: RBAC steuert wer etwas tun darf, Policy steuert wie es konfiguriert sein muss.

---

## Policy-Konzepte

**Policy-Effekte (was passiert bei Verstoß):**

| Effekt | Beschreibung |
|--------|-------------|
| `Deny` | Ressource wird **nicht** erstellt/geändert |
| `Audit` | Ressource wird erstellt aber als **non-compliant markiert** |
| `Modify` | Ressource wird **automatisch angepasst** |
| `DeployIfNotExists` | Zugehörige Ressource wird **automatisch deployt** |
| `Disabled` | Policy ist deaktiviert |

**Policy-Struktur:**
1. **Policy Definition**: Was wird geprüft? Was passiert bei Verstoß?
2. **Policy Assignment**: Auf welchen Scope wird die Policy angewendet?
3. **Policy Initiative**: Gruppe von mehreren Policy Definitions

---

## Eingebaute Policies zuweisen

Azure hat hunderte vordefinierte Policies. Diese zuzuweisen ist der schnellste Weg:

### Im Portal

1. Suche nach **Policy** im Azure Portal
2. Links: **Definitions** – alle verfügbaren Policies anzeigen
3. Suche nach "storage https" – du findest **"Secure transfer to storage accounts should be enabled"**
4. Klicke auf die Policy → **Assign**

| Feld | Wert |
|------|------|
| Scope | deine Subscription oder Resource Group |
| Policy definition | (vorausgewählt) |
| Effect | Audit |
| Assignment name | `Storage HTTPS erzwingen` |

5. Klicke **Review + create** → **Create**

### Per CLI

```bash
# Alle eingebauten Policies mit "tag" im Namen suchen
az policy definition list \
  --query "[?policyType=='BuiltIn' && contains(displayName,'tag')].{Name:displayName, ID:name}" \
  --output table | head -20

# Policy zuweisen (Audit: alle Ressourcen ohne bestimmten Tag melden)
az policy assignment create \
  --name "require-environment-tag" \
  --display-name "Environment-Tag erforderlich" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-devops" \
  --params '{"tagName": {"value": "environment"}}'
```

---

## Compliance-Status prüfen

Nach der Zuweisung dauert es 5–15 Minuten bis Azure alle Ressourcen auswertet:

1. **Policy** → **Compliance**
2. Du siehst alle Assignments und wie viele Ressourcen compliant/non-compliant sind
3. Klicke auf eine Assignment um Details zu sehen (welche Ressourcen verstoßen?)

```bash
# Compliance per CLI prüfen
az policy state list \
  --resource-group rg-devops \
  --query "[?complianceState=='NonCompliant'].{Ressource:resourceId, Policy:policyDefinitionName}" \
  --output table
```

---

## Eigene Policy schreiben

Erstelle `deny-public-storage.json` – verhindert Storage Accounts mit öffentlichem Blob-Zugriff:

```json
{
  "properties": {
    "displayName": "Öffentlichen Blob-Zugriff auf Storage Accounts verbieten",
    "description": "Verbietet das Erstellen von Storage Accounts mit allowBlobPublicAccess=true",
    "policyType": "Custom",
    "mode": "Indexed",
    "parameters": {},
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Storage/storageAccounts"
          },
          {
            "field": "Microsoft.Storage/storageAccounts/allowBlobPublicAccess",
            "equals": "true"
          }
        ]
      },
      "then": {
        "effect": "Deny"
      }
    }
  }
}
```

```bash
# Policy Definition erstellen
az policy definition create \
  --name "deny-public-blob-access" \
  --display-name "Öffentlichen Blob-Zugriff verbieten" \
  --rules deny-public-storage.json \
  --mode Indexed

# Policy zuweisen
az policy assignment create \
  --name "no-public-storage" \
  --policy "deny-public-blob-access" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/rg-devops"
```

Test: Versuche jetzt einen Storage Account mit `--allow-blob-public-access true` zu erstellen – du erhältst einen Fehler.

---

## Policy Initiative: Mehrere Policies bündeln

Eine **Initiative** (Policy Set) fasst mehrere Policies zusammen:

```bash
# Eingebaute Initiative: Azure Security Benchmark
az policy set-definition list \
  --query "[?policyType=='BuiltIn' && contains(displayName,'Benchmark')].displayName" \
  --output table
```

Initiativen zuweisen genauso wie einzelne Policies – aber du weist 50+ Policies mit einem Klick zu.

---

## Resource Locks: Löschen verhindern

Resource Locks sind ein anderes Konzept: Sie verhindern bestimmte Aktionen unabhängig von RBAC und Policy.

**Zwei Lock-Typen:**

| Typ | Was wird verhindert |
|-----|-------------------|
| `CanNotDelete` | Ressource kann nicht gelöscht werden (aber geändert) |
| `ReadOnly` | Ressource kann weder gelöscht noch geändert werden |

!!! warning "ReadOnly kann unerwartete Probleme verursachen"
    `ReadOnly` verhindert auch Operationen die intern Änderungen vornehmen – z.B. kann eine Storage Account mit `ReadOnly`-Lock keine neuen Blob-Schlüssel generieren. Vorsicht in Produktion.

### Lock setzen

```bash
# Löschen der gesamten Resource Group verhindern
az lock create \
  --name "protect-rg" \
  --resource-group rg-devops \
  --lock-type CanNotDelete \
  --notes "Schutz vor versehentlichem Löschen"
```

### Lock im Portal

1. Resource Group öffnen
2. Links: **Locks**
3. **+ Add** → Lock-Typ wählen → **OK**

### Lock löschen

```bash
az lock delete \
  --name "protect-rg" \
  --resource-group rg-devops
```

!!! info "Wer kann Locks löschen?"
    Locks können nur von Benutzern mit der Rolle `Owner` oder `User Access Administrator` (oder der spezifischen Permission `Microsoft.Authorization/locks/delete`) gelöscht werden – nicht von einfachen Contributors.

---

## Challenge

!!! question "Challenge: Tagging-Policy"
    Erstelle eine Policy die:
    
    - Alle neuen Ressourcen in `rg-devops` prüft
    - Non-compliant meldet (`Audit`) wenn das Tag `environment` fehlt
    - Weise sie der Resource Group zu
    - Erstelle eine Ressource ohne Tag und prüfe den Compliance-Status
    
    (Tipp: Suche in den eingebauten Policies nach "tag".)

??? success "Hinweis"
    ```bash
    # Eingebaute Policy-ID für "Require a tag on resources"
    az policy definition list \
      --query "[?displayName=='Require a tag on resources'].name" -o tsv
    
    # Dann Assignment erstellen:
    az policy assignment create \
      --name "require-env-tag" \
      --policy $(az policy definition list \
        --query "[?displayName=='Require a tag on resources'].name" -o tsv) \
      --scope "/subscriptions/.../resourceGroups/rg-devops" \
      --params '{"tagName": {"value": "environment"}}'
    ```

---

Weiter zu [Modul 39 – Aufräumen Lernpfad 6](modul-39-aufräumen.md) →
