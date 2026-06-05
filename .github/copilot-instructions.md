# Copilot Instructions – Azure Einstiegstraining

Dieses Repository ist eine MkDocs-Material-Dokumentation für ein selbstgeführtes Azure-Training.
Zielgruppe: IT-Profis im Microsoft-Umfeld (Windows-Admins, Azubis, Praktikanten) ohne Azure-Erfahrung.
Sprache: **Deutsch** (duzen, direkte Ansprache, keine Fachbegriffe ohne Erklärung).

---

## Projektstruktur

```
mkdocs.yml          ← Navigation, Theme, Plugins
README.md           ← Repo-Übersicht (Modul-Tabelle + Quickstart)
docs/
  index.md          ← Willkommensseite (Tabellen: Was du baust + Modulübersicht)
  modul-0-*.md      ← Modul-Dateien (fortlaufend nummeriert)
  modul-N-*.md
.github/
  workflows/deploy.yml   ← GitHub Pages Deploy via mkdocs gh-deploy
  copilot-instructions.md
```

---

## Konventionen für neue Module

### 1. Dateiname

```
docs/modul-{N}-{kurzname}.md
```

Beispiel: `docs/modul-8-keyvault.md`

Keine Umlaute im Dateinamen außer wenn bereits etabliert (Ausnahme: `modul-7-aufräumen.md`).

### 2. Datei-Inhalt (Pflichtstruktur)

```markdown
# Modul {N} – {Titel}

## Lernziele

Nach diesem Modul kannst du:

- ...
- ...

---

## Hintergrund: {Konzept erklären}

On-Prem-Vergleich wo sinnvoll (Hyper-V, lokaler Server, Active Directory, etc.)

---

## {Hauptteil: Schritt-für-Schritt im Azure Portal}

### Schritt 1: ...
### Schritt 2: ...

---

## Challenge

!!! question "Challenge: ..."
    ...

??? success "Hinweis"
    ...

---

Weiter zu [Modul {N+1} – {Titel}](modul-{N+1}-{kurzname}.md) →
```

### 3. Admonition-Typen (einheitlich verwenden)

| Typ | Wann |
|-----|------|
| `!!! tip` | Nützliche Hinweise, Abkürzungen |
| `!!! info` | Hintergrundinformation, Free Tier, Kosten |
| `!!! warning` | Wichtige Warnung (Kosten, Datenverlust, Sicherheit) |
| `!!! success` | Erfolgsbestätigung am Ende eines Abschnitts |
| `!!! question` | Challenge-Aufgabe |
| `??? success "Hinweis"` | Aufklappbarer Lösungshinweis zur Challenge |

### 4. Konfigurationstabellen

Immer als Markdown-Tabelle mit `Feld | Wert`:

```markdown
| Feld | Wert |
|------|------|
| Resource group | `rg-aztraining` |
| Region | `West Europe` |
```

Ressource-Namen immer in Backticks: `` `rg-aztraining` ``, `` `vm-training` ``

---

## Checkliste beim Hinzufügen eines neuen Moduls

Wenn ein neues Modul `modul-{N}-{name}.md` erstellt wird, **immer alle folgenden Dateien aktualisieren**:

### mkdocs.yml – nav-Abschnitt

Die nav-Struktur verwendet **Lernpfad-Gruppen**: Willkommen (direkte Seite) + Lernpfade (aufklappbare Gruppen in der Sidebar). Alle Module eines Lernpfads kommen als Unterelemente der jeweiligen Lernpfad-Gruppe.

```yaml
nav:
  - Willkommen: index.md
  - Lernpfad 1 – Grundlagen:
    - Modul 0 – Orientierung: modul-0-orientierung.md
    # ... bestehende Module ...
    - Modul {N} – {Titel}: modul-{N}-{kurzname}.md
    - Modul 7 – Aufräumen: modul-7-aufräumen.md   # immer letztes Modul im Lernpfad
  - Lernpfad 2 – {Thema}:
    - Modul 8 – ...
    # ...
    - Modul {M} – Aufräumen: modul-{M}-aufräumen.md
```

**Wichtig – MkDocs Material Features:** Die folgenden Features sind aktiv (und sollen NICHT geändert werden):
- `navigation.footer` – zeigt Vorherige/Nächste-Seite am Seitenende
- `navigation.top` – Zurück-nach-oben-Button
- `toc.follow` – Inhaltsverzeichnis folgt der Scrollposition
- `search.suggest`, `search.highlight`, `content.code.copy` etc.

**Nicht verwenden:** `navigation.tabs` und `navigation.sections` – diese verursachen entweder Overflow in der Tab-Leiste oder auto-generierte Listen im Body.

Neues Modul **vor** dem letzten Aufräumen-Modul des aktuellen Lernpfads einfügen. Jeder Lernpfad hat sein eigenes Aufräumen-Modul als letztes.

### docs/index.md – zwei Stellen

**1. Tabelle "Was du in Lernpfad 1 lernst" (Emoji + Was + Wo)**

```markdown
| 🔑 | {Kurzbeschreibung was gebaut wird} | {wo es erreichbar ist} |
```

**2. Tabelle "Übersicht der Module"**

```markdown
| [Modul {N}](modul-{N}-{kurzname}.md) | {Thema} | {Dauer} Min |
```

Neues Modul vor Modul 7 einfügen.

### README.md – Modul-Tabelle

```markdown
| {N} | {Thema kurz} |
```

Vor Modul 7 einfügen.

### Vorheriges Modul – "Weiter zu"-Link

Am Ende von Modul `{N-1}` den Link anpassen:

```markdown
Weiter zu [Modul {N} – {Titel}](modul-{N}-{kurzname}.md) →
```

### Letztes Modul des Lernpfads (Aufräumen) – Ressourcentabelle

In `modul-{M}-aufräumen.md` die Tabelle am Anfang um die neue Ressource ergänzen.
Ebenso in der Abschluss-Tabelle "Was du gebaut hast" am Ende.

---

## Stil-Regeln

- **Duzen**: "du kannst", "du siehst", "klicke auf"
- **On-Prem-Vergleich**: Immer früh im Modul erklären was das Äquivalent on-prem ist
- **Free Tier bevorzugen**: Wo möglich Free/kostenlose SKUs nutzen und mit `!!! info` erklären
- **Kosten nennen**: Jede kostenpflichtige Ressource mit ca. Preis erwähnen
- **Cloud Shell**: Kein lokales Setup – alles über Azure Cloud Shell (Bash)
- **Python**: Wenn Code nötig, Python bevorzugen (passt zum bereits etablierten Modul 3)
- **Keine englischen Begriffe ohne Erklärung**: Erste Verwendung mit Klammer erklären, z.B. "PaaS (Platform as a Service)"

---

## Themenideen für weitere Module (vollständiger Trainingsplan)

Siehe [docs/roadmap.md](../docs/roadmap.md) für den vollständigen Plan. Kurzfassung:

| Lernpfad | Module | Thema |
|----------|--------|-------|
| LP 3 | 15–20 | Datenbanken (Azure SQL, Cosmos DB, PostgreSQL, Redis) |
| LP 4 | 21–25 | Identity & Access (Entra ID, Managed Identity, RBAC, Conditional Access) |
| LP 5 | 26–32 | Container (Docker, ACR, ACI, Container Apps, AKS) |
| LP 6 | 33–39 | DevOps & IaC (Azure DevOps, GitHub Actions, ARM, Bicep, Terraform) |
| LP 7 | 40–44 | Monitoring & Security (Log Analytics, App Insights, Defender, Policy) |
| LP 8 | 45–47 | Abschlussprojekte (3-Tier-App, Event-Driven, Container+CI/CD) |
