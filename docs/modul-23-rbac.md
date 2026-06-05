# Modul 23 – Azure RBAC vertieft: Eigene Rollen und Scopes

## Lernziele

Nach diesem Modul kannst du:

- Die RBAC-Scope-Hierarchie (Management Group → Subscription → Resource Group → Ressource) erklären
- Bestehende Rollenzuweisungen im Portal auslesen und analysieren
- Eigene Custom Roles erstellen wenn Built-in Roles nicht passen
- Den Unterschied zwischen Azure RBAC und Entra ID-Rollen (Directory Roles) erklären
- Privilege Access Reviews als Sicherheitspraxis beschreiben

---

## Hintergrund: Was ist RBAC?

**On-Prem-Vergleich:** Im Active Directory gibt es Gruppen und Berechtigungen: Wer darf auf welchen Fileserver-Ordner zugreifen, wer kann GPOs ändern, wer ist Domain Admin. Das funktioniert über Gruppenrichtlinien und ACLs.

In Azure heißt das Konzept **RBAC (Role-Based Access Control)**: Wer darf auf welcher Azure-Ressource was tun?

**Drei Grundelemente:**

```
WER      +    WELCHE ROLLE    +    AUF WELCHEM SCOPE
(Identität)   (Berechtigung)       (Ressourcenebene)

Beispiel:
Anna     +    Reader           +    Resource Group "rg-aztraining"
```

---

## Die Scope-Hierarchie

Azure RBAC hat vier Ebenen – Berechtigungen vererben sich nach unten:

```
Management Group       (Organisationsebene – mehrere Subscriptions)
    └── Subscription   (Abrechnungseinheit)
            └── Resource Group   (logische Containergruppe)
                    └── Ressource   (einzelne VM, Storage, etc.)
```

**Vererbungsprinzip:** Eine Rolle auf Subscription-Ebene gilt automatisch für alle Resource Groups und Ressourcen darin.

**Best Practice:** Rollen so tief wie möglich vergeben (**Least Privilege**). Wenn jemand nur Zugriff auf eine Resource Group braucht, keine Subscription-Rolle vergeben.

!!! tip "Scope in der Praxis"
    In kleinen Umgebungen (1 Subscription, 2–3 Resource Groups) ist das einfach. In Unternehmen mit 10+ Subscriptions und hunderten Resource Groups wird RBAC über Management Groups und Azure Policy gesteuert – das ist Modul 23 already Level "Senior Cloud Engineer".

---

## Bestehende Rollenzuweisungen anzeigen

### Auf einer Resource Group

1. Navigiere zu einer Resource Group (z.B. `rg-aztraining`)
2. Links im Menü: **Access control (IAM)**
3. Tab **Role assignments**

Du siehst: Wer hat welche Rolle auf dieser Ressource? Die Spalten sind:
- **Name**: Benutzer, Gruppe oder Service Principal
- **Type**: User / Group / Service Principal / Managed Identity
- **Role**: die zugewiesene Rolle
- **Scope**: auf welchem Scope die Rolle vergeben wurde (ggf. höher als diese RG)

!!! info "Inherited vs. Direct"
    Rollen können direkt oder geerbt sein. Wenn du auf Subscription-Ebene `Owner` bist, erscheint das auch bei jeder Resource Group als Rolle – als "Inherited".

### Alle eigenen Rollen anzeigen

```bash
# Alle Rollenzuweisungen für den eingeloggten Benutzer
az role assignment list --assignee $(az account show --query user.name -o tsv) --all --output table
```

---

## Built-in Roles – die wichtigsten im Überblick

Azure hat über 100 vordefinierte Rollen. Diese kennst du am häufigsten:

| Rolle | Beschreibung | Typischer Use Case |
|-------|-------------|-------------------|
| **Owner** | Alles, inkl. RBAC verwalten | Subscription-Besitzer |
| **Contributor** | Ressourcen erstellen/ändern/löschen, aber kein RBAC | Entwickler, DevOps |
| **Reader** | Alles lesen, nichts ändern | Auditoren, Monitoring-Tools |
| **User Access Administrator** | Nur RBAC-Zuweisungen verwalten | Zugriffsverwaltung delegieren |
| **Storage Blob Data Contributor** | Blobs lesen/schreiben | Apps mit Managed Identity |
| **Key Vault Secrets User** | Secrets lesen | Apps die Secrets aus KV lesen |
| **Virtual Machine Contributor** | VMs verwalten (nicht Netzwerk) | VM-Admins |

!!! warning "Owner-Rolle sparsam vergeben"
    `Owner` darf auch RBAC verwalten – also weitere Personen zu Ownern machen. In Produktion: `Contributor` für Entwickler, `Owner` nur für Cloud-Admins.

---

## Custom Role erstellen

Wenn keine Built-in Role passt, kannst du eigene Rollen definieren. Beispiel: du willst dass jemand VMs **starten und stoppen** darf, aber nicht erstellen oder löschen.

### Schritt 1: Permissions recherchieren

```bash
# Alle Actions für Virtual Machines anzeigen
az provider operation show --namespace Microsoft.Compute \
  --query "resourceTypes[?name=='virtualMachines'].operations[].name" \
  -o tsv | grep -i "start\|deallocate\|powerOff"
```

Du findest:
- `Microsoft.Compute/virtualMachines/start/action`
- `Microsoft.Compute/virtualMachines/deallocate/action`
- `Microsoft.Compute/virtualMachines/powerOff/action`
- `Microsoft.Compute/virtualMachines/read`

### Schritt 2: JSON-Definition erstellen

Öffne in der Cloud Shell:

```bash
code vm-operator-role.json
```

```json
{
  "Name": "VM Operator",
  "Description": "Darf VMs starten, stoppen und den Status lesen. Keine Erstellungs- oder Löschrechte.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Resources/subscriptions/resourceGroups/read"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/DEINE-SUBSCRIPTION-ID"
  ]
}
```

Subscription-ID holen:

```bash
az account show --query id -o tsv
```

Trage sie in `AssignableScopes` ein (ersetze `DEINE-SUBSCRIPTION-ID`).

### Schritt 3: Custom Role erstellen

```bash
az role definition create --role-definition vm-operator-role.json
```

### Schritt 4: Rolle zuweisen

```bash
# Rolle dem Testbenutzer zuweisen (auf Resource Group)
az role assignment create \
  --assignee testuser@XXXX.onmicrosoft.com \
  --role "VM Operator" \
  --resource-group rg-aztraining
```

Die Rolle erscheint jetzt in **Access control (IAM)** → **Role assignments** der Resource Group.

### Custom Role löschen (Aufräumen)

```bash
az role definition delete --name "VM Operator"
```

---

## Azure RBAC vs. Entra ID-Rollen

Ein häufiger Verwechslungspunkt:

| | Azure RBAC | Entra ID Directory Roles |
|--|-----------|--------------------------|
| **Steuert Zugriff auf** | Azure-Ressourcen (VMs, Storage, ...) | Entra ID selbst (Benutzer verwalten, Apps registrieren) |
| **Beispiele** | Owner, Contributor, Reader | Global Administrator, User Administrator |
| **Vergeben in** | Azure Portal → IAM | Entra ID-Portal → Roles |
| **Scope** | Subscription / RG / Ressource | Tenant-weit (oder Administrative Unit) |

**Faustregel:** 
- Wer Azure-Ressourcen verwalten soll → **Azure RBAC**
- Wer Entra ID-Benutzer oder -Apps verwalten soll → **Entra ID Directory Role**

---

## Access Review: Wer hat noch Zugang?

In Unternehmen wächst RBAC schnell unkontrolliert. Access Reviews (benötigt **Entra ID P2**) erlauben periodische Überprüfungen: jeder Manager bestätigt ob seine Mitarbeiter noch die Zugriffsrechte brauchen.

Auch ohne P2 kannst du manuell prüfen:

```bash
# Alle Rollenzuweisungen in einer Subscription ausgeben
az role assignment list --all --output table \
  --query "[].{Benutzer:principalName, Rolle:roleDefinitionName, Scope:scope}"
```

```bash
# Besonders kritisch: wer hat Owner-Rechte?
az role assignment list --all \
  --query "[?roleDefinitionName=='Owner'].{Benutzer:principalName, Scope:scope}" \
  --output table
```

---

## Challenge

!!! question "Challenge: Eigene Rolle für Storage"
    Erstelle eine Custom Role `Storage Readonly Plus` die folgendes darf:
    
    - Blobs lesen (`Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read`)
    - Container auflisten (`Microsoft.Storage/storageAccounts/blobServices/containers/read`)
    - Storage Account Metadaten lesen (`Microsoft.Storage/storageAccounts/read`)
    - **Aber nicht**: Blobs schreiben, löschen, oder Storage Account ändern
    
    Weise die Rolle dem Testbenutzer auf dem Storage Account aus Modul 22 zu.

??? success "Hinweis"
    ```json
    {
      "Name": "Storage Readonly Plus",
      "Description": "Liest Blobs und listet Container auf",
      "Actions": [
        "Microsoft.Storage/storageAccounts/read",
        "Microsoft.Storage/storageAccounts/blobServices/read",
        "Microsoft.Storage/storageAccounts/blobServices/containers/read",
        "Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read"
      ],
      "NotActions": [],
      "AssignableScopes": ["/subscriptions/DEINE-SUBSCRIPTION-ID"]
    }
    ```
    
    Dann: `az role definition create --role-definition storage-readonly-plus.json`
    
    Und zuweisen: `az role assignment create --assignee testuser@... --role "Storage Readonly Plus" --scope /subscriptions/.../resourceGroups/.../providers/Microsoft.Storage/storageAccounts/staztraining-XXXX`

---

Weiter zu [Modul 24 – Conditional Access: Bedingter Zugriff und MFA](modul-24-conditional-access.md) →
