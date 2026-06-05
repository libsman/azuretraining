# Modul 44 – Aufräumen: Lernpfad 7

Am Ende von Lernpfad 7 hast du folgende Ressourcen erstellt:

| Ressource | Name | Modul |
|-----------|------|-------|
| Resource Group | `rg-monitoring` | Modul 40 |
| Log Analytics Workspace | `law-training` | Modul 40 |
| Application Insights | `ai-training` | Modul 41 |
| Availability Test | `homepage-ping` | Modul 41 |
| Action Group | `ag-email-training` | Modul 40 |
| Metric Alerts | `5xx-errors-alert`, `slow-responses` | Modul 40/41 |
| Microsoft Sentinel | aktiviert auf `law-training` | Modul 43 |
| Analytics Rules | Custom Rules | Modul 43 |

Außerdem konfiguriert (keine eigene Ressource, aber rückgängig machen):

| Konfiguration | Wo | Modul |
|--------------|-----|-------|
| Diagnose-Einstellungen | App Service aus LP1 | Modul 40 |
| Data Connectors | Sentinel | Modul 43 |

---

## Schritt 1: Microsoft Sentinel entfernen

!!! warning "Sentinel zuerst deaktivieren"
    Sentinel muss vor dem Löschen des Log Analytics Workspace deaktiviert werden.

1. **Microsoft Sentinel** → dein Workspace
2. **Settings** → **Remove Microsoft Sentinel**
3. Bestätige mit **Remove**

Oder per CLI:

```bash
az sentinel onboarding-state delete \
  --resource-group rg-monitoring \
  --workspace-name law-training \
  --name default
```

---

## Schritt 2: Diagnose-Einstellungen entfernen

Entferne die Verbindung vom App Service zum Log Analytics Workspace:

```bash
# Diagnose-Einstellung der App Service entfernen
APP_ID=$(az webapp show --name DEIN-APP-NAME --resource-group rg-aztraining --query id -o tsv)

az monitor diagnostic-settings delete \
  --name "appservice-to-law" \
  --resource $APP_ID
```

Im Portal: App Service → **Diagnostic settings** → Zeile anklicken → **Delete**

---

## Schritt 3: Availability Tests und Alerts löschen

```bash
# Availability Tests (im App Insights)
az monitor app-insights web-test delete \
  --resource-group rg-monitoring \
  --name homepage-ping

# Alle Metric Alerts löschen
az monitor metrics alert delete --name "5xx-errors-alert" --resource-group rg-monitoring
az monitor metrics alert delete --name "slow-responses" --resource-group rg-monitoring 2>/dev/null || true

# Log-basierte Alerts löschen
az monitor scheduled-query delete --name "failed-logins-alert" --resource-group rg-monitoring 2>/dev/null || true
```

---

## Schritt 4: Resource Group löschen

```bash
az group delete --name rg-monitoring --yes --no-wait
```

Das löscht: Log Analytics Workspace, Application Insights, Application Insights Availability Tests, Action Groups und alle weiteren Ressourcen in `rg-monitoring`.

!!! info "Defender for Cloud bleibt aktiv"
    Microsoft Defender for Cloud (CSPM Free Tier) ist immer aktiv für deine Subscription – ohne Kosten und ohne eigene Ressourcen. Es gibt nichts zu löschen.

---

## Was du in Lernpfad 7 gelernt hast

| Modul | Thema | Wichtigstes Konzept |
|-------|-------|-------------------|
| 40 | Azure Monitor & Log Analytics | KQL-Abfragen, zentrales Log-Repository, Alerts |
| 41 | Application Insights | APM, SDK-Integration, Request/Exception-Tracking |
| 42 | Defender for Cloud | CSPM, Secure Score, Security Alerts, JIT |
| 43 | Microsoft Sentinel | Cloud-SIEM, Analytics Rules, Incident Response |

---

!!! success "Lernpfad 7 abgeschlossen!"
    Du kannst jetzt Azure-Ressourcen überwachen, Anomalien mit KQL-Abfragen finden, Web-App-Telemetrie auswerten und aktive Bedrohungen mit Microsoft Sentinel erkennen. Das sind Fähigkeiten die direkt im Berufsalltag als Cloud-Administrator oder Security-Engineer gefragt sind.

---

Weiter zu [Modul 45 – Abschlussprojekt 1: Dreischichtige Web-App](modul-45-projekt-webapp.md) →
