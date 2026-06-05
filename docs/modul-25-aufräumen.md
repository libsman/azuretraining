# Modul 25 – Aufräumen: Lernpfad 4 abschließen

## Was du in Lernpfad 4 gebaut hast

| Ressource | Typ | Modul | Kosten/Monat |
|-----------|-----|-------|-------------|
| `rg-identity` | Resource Group | alle | kostenlos |
| `app-identity-XXXX` | App Service (Free F1) | 22 | 0 € |
| `staztraining-XXXX` | Storage Account | 22 | < 1 € |
| `id-shared-apps` | User-assigned Managed Identity | 22 | 0 € |
| Custom Role `VM Operator` | RBAC Custom Role | 23 | 0 € |
| Custom Role `Storage Readonly Plus` | RBAC Custom Role | 23 | 0 € |
| Testbenutzer `testuser@...` | Entra ID User | 21 | 0 € |
| Gruppe `grp-azure-leser` | Entra ID Group | 21 | 0 € |
| Conditional Access Policies | Entra ID Policy | 24 | 0 € |

!!! info "Entra ID-Objekte kosten nichts"
    Benutzer, Gruppen, Managed Identities und Conditional Access-Policies verursachen keine direkten Azure-Kosten (außer bei Entra ID P1/P2-Lizenzen). Du kannst sie länger behalten – aber für Ordnung trotzdem aufräumen.

---

## Azure-Ressourcen löschen

### Option A: Resource Group löschen

```bash
az group delete --name rg-identity --yes --no-wait
```

### Option B: Einzelne Ressourcen löschen

```bash
# Storage Account löschen
az storage account delete \
  --name staztraining-XXXX \
  --resource-group rg-identity \
  --yes

# App Service löschen
az webapp delete \
  --name app-identity-XXXX \
  --resource-group rg-identity

# User-assigned Managed Identity löschen
az identity delete \
  --name id-shared-apps \
  --resource-group rg-identity
```

---

## Custom Roles löschen

Custom Roles existieren auf Subscription-Ebene und werden nicht durch RG-Löschung entfernt:

```bash
az role definition delete --name "VM Operator"
az role definition delete --name "Storage Readonly Plus"
```

Überprüfen:

```bash
# Alle Custom Roles in der Subscription
az role definition list --custom-role-only true --output table
```

---

## Entra ID-Objekte aufräumen

### Testbenutzer löschen

1. **Microsoft Entra ID** → **Users** → **All users**
2. Klicke auf `Test User`
3. **Delete** oben → Bestätigen

!!! info "Soft Delete in Entra ID"
    Gelöschte Benutzer landen im **Recycle Bin** und können 30 Tage wiederhergestellt werden:
    **Users** → **Deleted users** → Benutzer auswählen → **Restore**

### Gruppen löschen

1. **Microsoft Entra ID** → **Groups** → **All groups**
2. Gruppe auswählen → **Delete** → Bestätigen

### Conditional Access Policies deaktivieren/löschen

1. **Microsoft Entra ID** → **Protection** → **Conditional Access** → **Policies**
2. Policy anklicken → **Delete** (oder auf **Off** setzen)

### Named Locations löschen

1. **Conditional Access** → **Named locations**
2. Location auswählen → **Delete**

---

## Was du in Lernpfad 4 gelernt hast

| Modul | Thema | Schlüsselkonzept |
|-------|-------|-----------------|
| 21 – Entra ID | Benutzer, Gruppen, B2B | Tenant, UPN, Security Groups, Gastbenutzer |
| 22 – Managed Identity | Apps ohne Passwörter | DefaultAzureCredential, System- vs. User-assigned |
| 23 – RBAC vertieft | Eigene Rollen & Scopes | Scope-Hierarchie, Custom Roles, Least Privilege |
| 24 – Conditional Access | Bedingter Zugriff | Zero Trust, MFA, Named Locations, Report-only |

### Konzepte die du jetzt kennst

**Identitäten in Azure:**
- ✅ Entra ID ist der Identitätsdienst für alle Azure-Zugriffe
- ✅ Managed Identities ersetzen Passwörter für Apps komplett
- ✅ `DefaultAzureCredential` funktioniert lokal (CLI) und in Azure (Managed Identity) ohne Codeänderung

**Zugriffssteuerung:**
- ✅ RBAC: Wer darf auf welcher Azure-Ressource was tun
- ✅ Scope-Hierarchie: Rollen vererben sich nach unten
- ✅ Least Privilege: immer minimale Rechte vergeben
- ✅ Custom Roles: wenn Built-in Roles nicht passen

**Zero Trust:**
- ✅ Security Defaults: Mindestschutz für jeden Tenant
- ✅ Conditional Access: Zugriff basierend auf Kontext (Standort, Gerät, Risiko)
- ✅ Report-only: Policies testen ohne Auswirkungen

---

## Lernpfad 4 – Checkliste

Bevor du weitermachst:

- [ ] Managed Identity auf einer App Service aktiviert und getestet
- [ ] RBAC-Rolle auf eine Ressource über das Portal vergeben
- [ ] Eine Custom Role erstellt und zugewiesen
- [ ] Den Unterschied zwischen Azure RBAC und Entra ID-Rollen erklären können
- [ ] Eine Conditional Access Policy im Report-only-Modus erstellt

---

Weiter zu [Lernpfad 5 – Container: Modul 26 – Docker Grundlagen](modul-26-docker.md) →

!!! info "Lernpfad 5 – Container"
    In Lernpfad 5 geht es um Container-Technologien: Docker, Azure Container Registry, Azure Container Instances, Container Apps und ein Einstieg in AKS (Kubernetes).
