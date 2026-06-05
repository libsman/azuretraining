# Modul 51 – Aufräumen Lernpfad 9

## Was du in Lernpfad 9 gebaut hast

| Ressource | Typ | Modul |
|-----------|-----|-------|
| `rg-lp9` | Resource Group | alle Module |
| `stalp9XXXXX` | Storage Account | Modul 48 |
| `company-data` | Azure File Share | Modul 48 |
| `sync-lp9` | Storage Sync Service | Modul 48 |
| `vault-lp9` | Recovery Services Vault | Modul 49 |
| `vm-backup-test` | Virtual Machine | Modul 49 |
| `law-lp9` | Log Analytics Workspace | Modul 49 |
| `hp-training` | AVD Host Pool | Modul 50 |
| `ag-desktop` | AVD Application Group | Modul 50 |
| `ws-training` | AVD Workspace | Modul 50 |
| `sp-training` | AVD Scaling Plan | Modul 50 |

---

## Aufräumen: Ressourcen löschen

!!! warning "Recovery Services Vault zuerst deaktivieren"
    Ein Vault mit aktiven Backups kann nicht einfach gelöscht werden. Du musst zuerst den Backup-Schutz deaktivieren und alle Backup-Daten löschen.

### Schritt 1: Backup-Schutz deaktivieren

```bash
# Backup-Schutz der VM deaktivieren UND Daten löschen
az backup protection disable \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --container-name "iaasvmcontainer;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --item-name "vm;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --delete-backup-data true \
  --yes
```

!!! tip "Falls der Container-Name unbekannt ist"
    ```bash
    az backup container list \
      --resource-group rg-lp9 \
      --vault-name vault-lp9 \
      --backup-management-type AzureIaasVM \
      --output table
    ```

### Schritt 2: Alle Ressourcen löschen

Wenn Vault und Backup-Daten bereinigt sind, löschst du einfach die gesamte Resource Group:

```bash
az group delete --name rg-lp9 --yes --no-wait
echo "Ressourcengruppe rg-lp9 wird gelöscht (läuft im Hintergrund)"
```

!!! info "AVD Session Hosts werden mitgelöscht"
    Die Session-Host-VMs liegen in `rg-lp9` – sie werden zusammen mit allem anderen gelöscht.

### Schritt 3: Entra ID-Registrierung bereinigen

AVD-Session Hosts, die per Entra ID gejoint wurden, hinterlassen einen Eintrag im Entra Portal:

1. Portal → **Microsoft Entra ID** → **Devices**
2. Suche nach `vm-backup-test` oder dem Hostnamen deiner Session-Host-VM
3. Gerät auswählen → **Delete**

### Schritt 4: Service Principal bereinigen (falls erstellt)

Falls du in Modul 50 einen Service Principal für den AVD-Scaling Plan erstellt hast:

```bash
# Service Principals deiner AVD-Ressourcen anzeigen
az ad sp list --display-name "AVD" --query "[].{Name:displayName, Id:id}" -o table

# Gezielt löschen
# az ad sp delete --id <ID>
```

---

## Was du in Lernpfad 9 gelernt hast

✅ **Azure Files**: SMB-Freigaben in der Cloud erstellen, einbinden und mit Snapshots sichern  
✅ **Azure File Sync**: Lokale Windows-Dateiserver transparent mit Azure synchronisieren  
✅ **Azure Backup**: Recovery Services Vault, Backup Policies, VM-Backup aktivieren  
✅ **File Recovery**: Einzelne Dateien aus einem Backup wiederherstellen ohne VM-Neuinstallation  
✅ **Azure Virtual Desktop**: Host Pools, Session Hosts, Application Groups, Workspaces  
✅ **AVD RemoteApp**: Einzelne Anwendungen statt ganzen Desktops streamen  

Du hast damit die wichtigsten Dienste kennengelernt, die Windows-Admins beim Übergang in die Cloud begegnen – von der Freigabe über das Backup bis zum virtuellen Desktop.

---

## Was du insgesamt gelernt hast

Du hast das komplette Azure Einstiegstraining mit 51 Modulen in 9 Lernpfaden abgeschlossen:

| Lernpfad | Themen |
|----------|--------|
| LP 1 – Grundlagen | VM, Storage, KI, App Service, Functions |
| LP 2 – Netzwerk & Sicherheit | VNet, NSG, Bastion, Load Balancer, Key Vault, Private Endpoints |
| LP 3 – Datenbanken | Azure SQL, Cosmos DB, PostgreSQL, Redis |
| LP 4 – Identity & Access | Entra ID, Managed Identity, RBAC, Conditional Access |
| LP 5 – Container | Docker, ACR, ACI, Container Apps, AKS |
| LP 6 – DevOps & IaC | Azure DevOps, GitHub Actions, ARM, Bicep, Terraform, Policy |
| LP 7 – Monitoring & Security | Log Analytics, App Insights, Defender for Cloud, Sentinel |
| LP 8 – Abschlussprojekte | 3-Tier-App, Event-Driven, Container + CI/CD |
| LP 9 – Hybrid & Windows Admin | Azure Files, File Sync, Backup, Recovery, AVD |

!!! tip "Nächste Schritte"
    - [AZ-900: Azure Fundamentals](https://learn.microsoft.com/de-de/certifications/azure-fundamentals/) – Grundlagenzertifizierung
    - [AZ-104: Azure Administrator](https://learn.microsoft.com/de-de/certifications/azure-administrator/) – für Windows-Admins sehr empfehlenswert (AVD, Backup, Files sind Prüfungsthemen!)
    - [AZ-140: Azure Virtual Desktop Specialty](https://learn.microsoft.com/de-de/certifications/azure-virtual-desktop-specialty/) – dedizierte AVD-Zertifizierung
