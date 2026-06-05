# Modul 49 – Azure Backup & Recovery

## Lernziele

Nach diesem Modul kannst du:

- Einen Recovery Services Vault erstellen und konfigurieren
- Eine Azure VM mit einer Backup-Richtlinie sichern
- Ein sofortiges Backup auslösen und den Status überwachen
- Einzelne Dateien aus einem Backup wiederherstellen (File Recovery)
- Backup-Kosten einschätzen

---

## Hintergrund: Backup in der Cloud

On-Premises kennst du Tools wie Veeam, Windows Server Backup, DPM oder Acronis. Backup-Jobs laufen nachts, Tapes werden rotiert, Restore-Tests oft vergessen.

**Azure Backup** ist der native Azure-Backup-Dienst: kein eigener Backup-Server, keine Medien, kein zusätzliches Lizenzmodell. Er sichert VMs, Dateien, SQL-Datenbanken, Blobs und mehr – alles zentral über den **Recovery Services Vault**.

| On-Prem | Azure |
|---------|-------|
| Backup-Server (Veeam, DPM) | Recovery Services Vault |
| Backup-Job | Backup Policy |
| Backup-Speicher / Tapes | Vault Storage (GRS oder LRS) |
| Restore-Assistent | File Recovery / VM-Restore im Portal |
| RPO/RTO manuell konfigurieren | Policy: täglich, wöchentlich, monatlich |

```
Azure VM
   │ Backup Extension (automatisch installiert)
   ▼
Recovery Services Vault
   ├── Backup Policy (Schedule + Retention)
   ├── Recovery Points (täglich / wöchentlich / monatlich / jährlich)
   └── Restore:
       ├── File Recovery (einzelne Dateien aus Snapshot mounten)
       ├── VM-Restore (neue VM aus Backup erstellen)
       └── Disk Restore (nur die Disk wiederherstellen)
```

!!! info "Kosten"
    Azure Backup berechnet: **pro geschützter VM** ~5 €/Monat (Instanzgebühr) + **Speicher** (~0,02 €/GiB/Monat für LRS). Für das Training entsteht wenig Kosten, wenn du schnell aufräumst.

---

## Schritt 1: Test-VM erstellen

Für dieses Modul erstellst du eine kleine VM, die wir sichern und aus der wir Dateien wiederherstellen.

```bash
# Ressourcengruppe aus Modul 48 wiederverwenden
# Falls noch nicht vorhanden:
az group create --name rg-lp9 --location westeurope

# Kleine Test-VM erstellen (Ubuntu, B1s = günstigste Option)
az vm create \
  --resource-group rg-lp9 \
  --name vm-backup-test \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --no-wait

echo "VM wird erstellt..."
```

Erstelle in der Zwischenzeit eine Testdatei, die wir später "versehentlich löschen" und wiederherstellen:

```bash
# Warten bis VM fertig ist
az vm wait --resource-group rg-lp9 --name vm-backup-test --created

# IP-Adresse holen und per SSH verbinden
VM_IP=$(az vm show -d --resource-group rg-lp9 --name vm-backup-test --query publicIps -o tsv)
ssh azureuser@$VM_IP "echo 'Wichtige Produktionsdaten!' > ~/wichtige-datei.txt && ls ~"
```

---

## Schritt 2: Recovery Services Vault erstellen

```bash
az backup vault create \
  --resource-group rg-lp9 \
  --name vault-lp9 \
  --location westeurope

echo "Vault erstellt: vault-lp9"
```

**Im Portal:** Suche nach "Recovery Services vaults" → `vault-lp9` → du siehst das Dashboard mit 0 Backup-Items.

### Redundanz konfigurieren

```bash
# Für das Training: LRS (günstiger), Produktion: GRS (georedundant)
az backup vault backup-properties set \
  --resource-group rg-lp9 \
  --name vault-lp9 \
  --backup-storage-redundancy LocallyRedundant
```

!!! info "GRS vs. LRS"
    **GRS (Geo-Redundant Storage)** repliziert Backups in eine zweite Region – bei Rechenzentrumsausfall bist du geschützt. **LRS** bleibt in einer Region, ist aber ~50 % günstiger. Für Produktion empfiehlt sich GRS oder GZRS.

---

## Schritt 3: Backup Policy erstellen

Eine Policy legt fest: **wann** gesichert wird und **wie lange** Recovery Points aufbewahrt werden.

```bash
# Standard-Policy anzeigen (Default für Azure VMs)
az backup policy show \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --name DefaultPolicy
```

Für das Training nutzen wir die DefaultPolicy (täglich 00:00 UTC, 30 Tage Retention). In der Produktion würdest du eine eigene Policy erstellen:

```bash
# Beispiel: Eigene Policy mit wöchentlichem Backup (JSON-Datei nötig)
# Im Portal geht das komfortabler:
# vault-lp9 → Backup policies → Add → Azure Virtual Machine
```

---

## Schritt 4: VM-Backup aktivieren

```bash
# VM-Backup aktivieren (nutzt DefaultPolicy)
az backup protection enable-for-vm \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --vm vm-backup-test \
  --policy-name DefaultPolicy

echo "Backup aktiviert – erste Sicherung läuft beim nächsten geplanten Zeitpunkt"
```

---

## Schritt 5: Sofort-Backup auslösen

Ein erster Backup-Job läuft automatisch, wenn du den Schutz aktivierst. Du kannst ihn auch manuell anstoßen:

```bash
# Manuellen Backup-Job starten
az backup protection backup-now \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --container-name "iaasvmcontainer;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --item-name "vm;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --backup-management-type AzureIaasVM \
  --retain-until $(date -d "+30 days" +%Y-%m-%d)
```

!!! tip "Einfacher im Portal"
    Portal → `vault-lp9` → Backup items → Azure Virtual Machines → `vm-backup-test` → **Backup now**. Dann unter **Backup Jobs** den Fortschritt beobachten. Ein erster Backup dauert ca. 5–15 Minuten.

### Backup-Status überwachen

```bash
# Job-Status abfragen
az backup job list \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --output table
```

---

## Schritt 6: File Recovery (Dateien aus Backup wiederherstellen)

Jetzt simulieren wir einen Datei-Verlust und stellen gezielt eine Datei wieder her – ohne die ganze VM zu überschreiben.

```bash
# "Versehentliches" Löschen auf der VM simulieren
ssh azureuser@$VM_IP "rm ~/wichtige-datei.txt && echo 'Datei gelöscht!'"
```

**File Recovery im Portal:**

1. Portal → `vault-lp9` → Backup items → Azure Virtual Machines → `vm-backup-test`
2. **File Recovery** klicken
3. Recovery Point auswählen (letztes Backup)
4. **Download Executable** (Windows .exe oder Linux-Script)
5. Skript auf der Ziel-VM ausführen → mountet das Backup als temporäres Laufwerk
6. Gewünschte Dateien kopieren
7. **Unmount disks** klicken (wichtig! Gemountete Backups haben ein 12h-Timeout)

```bash
# Alternative: Direkte Dateiwiederherstellung per CLI
# (Komplexer – im Portal empfohlen für den Einstieg)

# Recovery Point anzeigen
az backup recoverypoint list \
  --resource-group rg-lp9 \
  --vault-name vault-lp9 \
  --container-name "iaasvmcontainer;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --item-name "vm;iaasvmcontainerv2;rg-lp9;vm-backup-test" \
  --backup-management-type AzureIaasVM \
  --workload-type VM \
  --output table
```

---

## Schritt 7: VM-Restore (optional)

Eine komplette VM aus dem Backup wiederherstellen:

```bash
# Im Portal: vault-lp9 → Backup items → vm-backup-test → Restore VM
# Optionen:
# - Create new VM (neue VM aus Backup-Image)
# - Replace existing (vorhandene VM überschreiben)
# - Restore disks (nur Disk, dann manuell anhängen)
```

!!! warning "Restore überschreibt Daten"
    "Replace existing" überschreibt die laufende VM! Für Produktionsumgebungen immer zuerst "Create new VM" wählen und testen.

---

## Challenge

!!! question "Challenge: Backup-Report"
    1. Öffne im Portal unter dem Vault: **Backup Reports** (Azure Monitor Workbooks)
    2. Warum sind dort noch keine Daten sichtbar?
    3. Aktiviere die Diagnoseeinstellungen des Vaults auf einen Log Analytics Workspace
    4. Welche Events werden dabei geloggt?

??? success "Hinweis"
    Backup Reports benötigen einen Log Analytics Workspace und ~24h Datensammlung.
    
    ```bash
    # Log Analytics Workspace erstellen
    az monitor log-analytics workspace create \
      --resource-group rg-lp9 \
      --workspace-name law-lp9 \
      --location westeurope
    
    # Diagnose für Vault aktivieren
    LAW_ID=$(az monitor log-analytics workspace show \
      --resource-group rg-lp9 \
      --workspace-name law-lp9 \
      --query id -o tsv)
    
    VAULT_ID=$(az backup vault show \
      --resource-group rg-lp9 \
      --name vault-lp9 \
      --query id -o tsv)
    
    az monitor diagnostic-settings create \
      --name vault-diag \
      --resource $VAULT_ID \
      --workspace $LAW_ID \
      --logs '[{"category": "AzureBackupReport","enabled": true}]'
    ```

---

Weiter zu [Modul 50 – Azure Virtual Desktop](modul-50-avd.md) →
