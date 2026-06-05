# Modul 39 – Aufräumen: Lernpfad 6

Am Ende von Lernpfad 6 hast du folgende Ressourcen erstellt:

| Ressource | Name | Modul |
|-----------|------|-------|
| Resource Group | `rg-devops` | Modul 33 |
| App Service Plan | `plan-devops` | Modul 33/34 |
| App Service | `app-devops-XXXX` | Modul 33/34 |
| Storage Account (ARM) | `starm-XXXX` | Modul 35 |
| App Service Plan (Bicep) | `bicep-plan` | Modul 36 |
| App Service (Bicep) | `bicep-app-XXXX` | Modul 36 |
| Key Vault (Bicep) | `kv-bicep-XXXX` | Modul 36 |
| Resource Group (Terraform) | `rg-terraform-training` | Modul 37 |
| Storage Account (Terraform) | `sttfXXXX` | Modul 37 |
| App Service (Terraform) | `app-tf-XXXX` | Modul 37 |
| Policy Assignment | `no-public-storage` | Modul 38 |
| Resource Locks | `protect-rg` | Modul 38 |

---

## Reihenfolge beachten

!!! warning "Locks vor Resource Group löschen"
    Resource Locks verhindern das Löschen. Lösche sie zuerst – sonst schlägt die Löschung der Resource Group fehl.

---

## Schritt 1: Policy Assignments entfernen

```bash
# Alle Policy Assignments in der Resource Group anzeigen
az policy assignment list \
  --resource-group rg-devops \
  --query "[].name" -o table

# Assignments löschen
az policy assignment delete --name "no-public-storage" --resource-group rg-devops
az policy assignment delete --name "require-env-tag" --resource-group rg-devops 2>/dev/null || true

# Custom Policy Definition löschen
az policy definition delete --name "deny-public-blob-access" 2>/dev/null || true
```

---

## Schritt 2: Resource Locks entfernen

```bash
# Alle Locks in der Resource Group anzeigen
az lock list --resource-group rg-devops --query "[].name" -o table

# Lock löschen
az lock delete --name "protect-rg" --resource-group rg-devops
```

---

## Schritt 3: Terraform-Ressourcen aufräumen

Im Terraform-Projektordner:

```bash
cd terraform-training

# Alle Terraform-Ressourcen löschen
terraform destroy -auto-approve
```

Das löscht `rg-terraform-training` mit allem darin (Storage, App Service, Key Vault falls erstellt).

---

## Schritt 4: Bicep/ARM-Ressourcen löschen

```bash
# Resource Group rg-devops komplett löschen
az group delete --name rg-devops --yes --no-wait
```

!!! info "--no-wait"
    Mit `--no-wait` wartet die CLI nicht auf Abschluss. Die Löschung läuft im Hintergrund weiter (5–10 Minuten).

---

## Schritt 5: Azure DevOps-Projekt löschen

1. Öffne **dev.azure.com** → deine Organisation
2. Projekteinstellungen (unten links: ⚙️ **Project settings**)
3. **Overview** → ganz unten: **Delete project**
4. Projektname eingeben → **Delete**

!!! warning "Azure DevOps-Organisation bleibt bestehen"
    Die Organisation selbst (`dev.azure.com/dein-name`) bleibt bestehen. Das Free Tier von Azure DevOps (5 Benutzer, unbegrenzte öffentliche Projekte) hat keine Kosten – du kannst sie behalten.

---

## Schritt 6: GitHub-Repository löschen (optional)

1. Öffne dein GitHub-Repository aus Modul 34
2. **Settings** → ganz unten: **Delete this repository**
3. Bestätige mit dem Repository-Namen

---

## Was du in Lernpfad 6 gelernt hast

| Modul | Thema | Wichtigstes Konzept |
|-------|-------|-------------------|
| 33 | Azure DevOps | Boards, Repos, Pipelines – alles in einem |
| 34 | GitHub Actions | `.github/workflows/*.yml` – CI/CD direkt aus GitHub |
| 35 | ARM Templates | JSON-basiertes IaC – Grundlage für alles Weitere |
| 36 | Bicep | Modernes IaC für Azure – kompiliert zu ARM |
| 37 | Terraform | Multi-Cloud IaC – State-Management, HCL-Syntax |
| 38 | Azure Policy & Locks | Compliance automatisieren, Löschen verhindern |

---

!!! success "Lernpfad 6 abgeschlossen!"
    Du hast die wichtigsten DevOps-Werkzeuge und IaC-Technologien kennengelernt. Mit Azure DevOps oder GitHub Actions kannst du Code automatisch testen und deployen. Mit Bicep oder Terraform kannst du Infrastruktur versionieren und reproduzierbar deployen.

---

Weiter zu [Modul 40 – Azure Monitor & Log Analytics](modul-40-log-analytics.md) →
