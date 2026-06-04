# Azure Einstiegstraining

Praxis-Einführung in Microsoft Azure für Einsteiger mit Hyper-V / On-Prem Erfahrung.

📖 **Dokumentation:** https://libsman.github.io/azuretraining

## Inhalt

| Modul | Thema |
|-------|-------|
| 0 | Orientierung: Portal, Konzepte, Resource Group |
| 1 | Erste VM: Ubuntu + nginx Webserver |
| 2 | Storage: Statische Website ohne Server |
| 3 | Azure KI: Bilderkennung mit Computer Vision |
| 4 | Monitoring: Kosten, Alerts, Tags |

## Lokale Vorschau

```bash
pip install -r requirements.txt
mkdocs serve
```

Öffne http://localhost:8000 im Browser.

## GitHub Pages Deployment

Beim Push auf `main` wird die Dokumentation automatisch über GitHub Actions gebaut und auf GitHub Pages veröffentlicht.

> **Einmalig nötig:** Unter Settings → Pages → Build and deployment → "Deploy from a branch" → `gh-pages` / `/ (root)` auswählen.
