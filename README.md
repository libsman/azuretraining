# Azure Einstiegstraining

Selbstgeführtes Azure-Training für IT-Profis im Microsoft-Umfeld – Azubis, Praktikanten und Windows-Admins.

📖 **Dokumentation:** https://libsman.github.io/azuretraining

## Lernpfade

| Lernpfad | Module | Thema |
|----------|--------|-------|
| LP 1 | 0–7 | Grundlagen (VM, Storage, KI, App Service, Functions) |
| LP 2 | 8–14 | Netzwerk & Sicherheit (VNet, NSG, Bastion, Load Balancer, Key Vault) |
| LP 3 | 15–20 | Datenbanken (SQL, Cosmos DB, PostgreSQL, Redis) |
| LP 4 | 21–25 | Identity & Access (Entra ID, Managed Identity, RBAC, Conditional Access) |
| LP 5 | 26–32 | Container (Docker, ACR, ACI, Container Apps, AKS) |
| LP 6 | 33–39 | DevOps & IaC (Azure DevOps, GitHub Actions, ARM, Bicep, Terraform, Policy) |
| LP 7 | 40–44 | Monitoring & Security (Log Analytics, App Insights, Defender for Cloud, Sentinel) |
| LP 8 | 45–47 | Abschlussprojekte (3-Tier-App, Event-Driven, Container + CI/CD) |
| LP 9 | 48–51 | Hybrid & Windows Admin (Azure Files, Backup, Azure Virtual Desktop) |

## Alle Module

### Lernpfad 1 – Grundlagen

| Modul | Thema |
|-------|-------|
| 0 | Orientierung: Portal, Konzepte, Resource Group |
| 1 | Erste VM: Ubuntu + nginx Webserver |
| 2 | Storage: Statische Website ohne Server |
| 3 | Azure KI: Bilderkennung mit Computer Vision |
| 4 | Monitoring: Kosten, Alerts, Tags |
| 5 | App Service: Web App ohne VM deployen (PaaS) |
| 6 | Azure Functions: Serverless Computing |
| 7 | Aufräumen Lernpfad 1 |

### Lernpfad 2 – Netzwerk & Sicherheit

| Modul | Thema |
|-------|-------|
| 8 | Virtual Network: Subnetze und Adressräume |
| 9 | Network Security Groups: Traffic filtern |
| 10 | Azure Bastion: Sicherer VM-Zugriff ohne Public IP |
| 11 | Load Balancer: Traffic auf mehrere Server verteilen |
| 12 | Key Vault: Secrets und API-Keys sicher speichern |
| 13 | Private Endpoints: Dienste intern erreichbar machen |
| 14 | Aufräumen Lernpfad 2 |

### Lernpfad 3 – Datenbanken

| Modul | Thema |
|-------|-------|
| 15 | Azure SQL Database: Managed Datenbank erstellen und abfragen |
| 16 | Azure Cosmos DB: NoSQL-Datenbank für flexible Daten |
| 17 | Azure Database for PostgreSQL: Open-Source-DB als Service |
| 18 | Azure Cache for Redis: In-Memory-Cache für schnelle Apps |
| 19 | App + Datenbank sicher verbinden |
| 20 | Aufräumen Lernpfad 3 |

### Lernpfad 4 – Identity & Access

| Modul | Thema |
|-------|-------|
| 21 | Microsoft Entra ID: Benutzer, Gruppen, B2B |
| 22 | Managed Identity: Apps ohne Passwörter |
| 23 | Azure RBAC vertieft: Custom Roles und Scopes |
| 24 | Conditional Access: Bedingter Zugriff und MFA |
| 25 | Aufräumen Lernpfad 4 |

### Lernpfad 5 – Container

| Modul | Thema |
|-------|-------|
| 26 | Docker Grundlagen: Images, Container, Dockerfile |
| 27 | Azure Container Registry: Eigene Images speichern |
| 28 | Azure Container Instances: Container ohne Server |
| 29 | Azure Container Apps: Serverless Container mit Autoscaling |
| 30 | AKS Überblick: Kubernetes-Konzepte und Cluster erstellen |
| 31 | AKS Workload: Deployen, skalieren, Rolling Updates |
| 32 | Aufräumen Lernpfad 5 |

### Lernpfad 6 – DevOps & IaC

| Modul | Thema |
|-------|-------|
| 33 | Azure DevOps: Repos, Boards und YAML-Pipelines |
| 34 | GitHub Actions: CI/CD direkt aus GitHub nach Azure |
| 35 | ARM Templates: Infrastruktur als JSON |
| 36 | Bicep: Modernes IaC für Azure |
| 37 | Terraform: Plattformunabhängiges IaC |
| 38 | Azure Policy & Resource Locks: Compliance automatisieren |
| 39 | Aufräumen Lernpfad 6 |

### Lernpfad 7 – Monitoring & Security

| Modul | Thema |
|-------|-------|
| 40 | Azure Monitor & Log Analytics: Logs zentral auswerten und KQL |
| 41 | Application Insights: Telemetrie für Web-Apps |
| 42 | Microsoft Defender for Cloud: Security Posture und Alerts |
| 43 | Microsoft Sentinel: SIEM und Incident Response |
| 44 | Aufräumen Lernpfad 7 |

### Lernpfad 8 – Abschlussprojekte

| Modul | Thema |
|-------|-------|
| 45 | Abschlussprojekt 1: Dreischichtige Web-App (App Service + SQL + Key Vault) |
| 46 | Abschlussprojekt 2: Event-Driven Architektur (Functions + Service Bus + Cosmos DB) |
| 47 | Abschlussprojekt 3: Container-App mit CI/CD-Pipeline (ACR + Container Apps + GitHub Actions) |

### Lernpfad 9 – Hybrid & Windows Admin

| Modul | Thema |
|-------|-------|
| 48 | Azure Files & File Sync: Dateiserver in der Cloud |
| 49 | Azure Backup & Recovery: Recovery Services Vault, VM-Backup, File Recovery |
| 50 | Azure Virtual Desktop: Host Pools, Session Hosts, RemoteApp |
| 51 | Aufräumen Lernpfad 9 |

## Lokale Vorschau

```powershell
pip install -r requirements.txt
python -m mkdocs serve
```

Oder kürzer (falls `mkdocs` im PATH ist):

```powershell
mkdocs serve
```

Oeffne http://localhost:8000 im Browser.

## GitHub Pages Deployment

Beim Push auf `main` wird die Dokumentation automatisch über GitHub Actions gebaut und auf GitHub Pages veröffentlicht.

> **Einmalig nötig:** Unter Settings → Pages → Build and deployment → "Deploy from a branch" → `gh-pages` / `/ (root)` auswählen.
