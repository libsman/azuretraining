# Modul 20 – Aufräumen: Lernpfad 3 abschließen

## Was du in Lernpfad 3 gebaut hast

| Ressource | Typ | Modul | Kosten/Monat |
|-----------|-----|-------|-------------|
| `rg-datenbanken` | Resource Group | alle | kostenlos |
| `sql-aztraining-XXXX` | Azure SQL Server (logisch) | 15 | kostenlos |
| `db-aztraining` | Azure SQL Database (Serverless) | 15 | ~0–3 € |
| `cosmos-aztraining-XXXX` | Azure Cosmos DB (Serverless) | 16 | ~0–1 € |
| `psql-aztraining-XXXX` | PostgreSQL Flexible Server | 17 | ~10 €/Monat |
| `redis-aztraining-XXXX` | Azure Cache for Redis (C0 Basic) | 18 | ~14 €/Monat |

!!! warning "Wichtig: Kosten stoppen"
    **PostgreSQL Flexible Server** (~10 €/Monat) und **Redis C0** (~14 €/Monat) laufen kontinuierlich. Wenn du nicht weitermachst, lösche die Resource Group oder stoppe die Dienste.
    
    Azure SQL Serverless pausiert automatisch nach Inaktivität → 0 € wenn nicht genutzt.  
    Cosmos DB Serverless → 0 € wenn nicht genutzt.

---

## Lernpfad 3 abschließen: Resource Group löschen

### Option A: Resource Group komplett löschen (empfohlen)

```bash
az group delete --name rg-datenbanken --yes --no-wait
```

Das löscht alle Ressourcen in `rg-datenbanken` asynchron. Dauert 5–15 Minuten.

### Option B: Einzelne Ressourcen stoppen

Wenn du die Ressourcen behalten willst aber Kosten sparen möchtest:

**PostgreSQL Flexible Server stoppen:**

1. `psql-aztraining-XXXX` → **Overview** → **Stop** oben in der Menüleiste
2. Gestoppter Server: 0 € für Compute, aber Storage (~2 €/Monat) läuft weiter
3. Azure stoppt den Server nach 7 Tagen automatisch-restart – dann manuell wieder stoppen

**Redis löschen (kein "Stop"):**

Redis Cache hat keine Stop-Funktion. Entweder komplett löschen oder weiterlaufen lassen.

```bash
az redis delete --name redis-aztraining-XXXX --resource-group rg-datenbanken --yes
```

### Option C: Nur PostgreSQL und Redis löschen

```bash
# PostgreSQL löschen
az postgres flexible-server delete \
  --name psql-aztraining-XXXX \
  --resource-group rg-datenbanken \
  --yes

# Redis löschen
az redis delete \
  --name redis-aztraining-XXXX \
  --resource-group rg-datenbanken \
  --yes
```

SQL und Cosmos DB (Serverless) können bleiben – kosten nichts wenn inaktiv.

---

## Löschung bestätigen

```bash
# Verbleibende Ressourcen prüfen
az resource list --resource-group rg-datenbanken --output table
```

Falls die Resource Group komplett gelöscht ist:

```bash
az group list --output table | findstr datenbanken
# Keine Ausgabe = Resource Group ist weg
```

---

## Was du in Lernpfad 3 gelernt hast

| Modul | Thema | Schlüsselkonzept |
|-------|-------|-----------------|
| 15 – Azure SQL | Managed relationale DB | Serverless, Free Tier, Query Editor, pyodbc |
| 16 – Cosmos DB | NoSQL Dokumentendb | Partition Key, Serverless RU/s, Change Feed |
| 17 – PostgreSQL | Open-Source DB als Service | Flexible Server, psql, Schemas, Trigger |
| 18 – Redis Cache | In-Memory Key-Value Store | Cache-Aside, TTL, Rate Limiting, Session Store |
| 19 – App + DB | Sichere Verbindung | Managed Identity, Key Vault Reference, Health Check |

### Konzepte die du jetzt kennst

**Datenbankentscheidungen:**

- ✅ Du weißt wann SQL (strukturierte Daten, ACID) vs. NoSQL (flexibles Schema, horizontale Skalierung) sinnvoll ist
- ✅ Du kennst drei Azure DB-Dienste: SQL Database, Cosmos DB, PostgreSQL
- ✅ Du verstehst Managed Service Vorteile: kein Patching, automatische Backups, integriertes HA

**Sicherheit:**

- ✅ Kein Passwort im Code – Managed Identity + Key Vault References
- ✅ Firewall-Regeln für Datenbankzugriff
- ✅ SSL/TLS für alle Datenbankverbindungen

**Performance:**

- ✅ Cache-Aside Pattern mit Redis
- ✅ Serverless-Tarife für unregelmäßige Last (SQL Serverless, Cosmos DB Serverless)

---

## Vergleich: Welche Datenbank für welchen Anwendungsfall?

| Anwendungsfall | Empfehlung |
|---------------|-----------|
| .NET-App mit fixem Schema, Transaktionen | Azure SQL Database |
| Migration von SQL Server on-premises | SQL Managed Instance |
| Produktkatalog mit variablen Attributen | Cosmos DB (NoSQL) |
| Open-Source-Stack, PostgreSQL-Erweiterungen | Azure DB for PostgreSQL |
| Session-Daten, API-Caching, Rate Limiting | Azure Cache for Redis |
| Kombination aus DB + Cache | PostgreSQL/SQL + Redis |

---

## Lernpfad 3 – Checkliste

Bevor du weitermachst, stelle sicher dass du folgende Dinge kannst:

- [ ] Azure SQL Database (Serverless) erstellen und Query Editor nutzen
- [ ] Cosmos DB-Container mit sinnvollem Partition Key anlegen
- [ ] PostgreSQL Flexible Server mit psql verbinden und Schemas anlegen
- [ ] Redis HSET/GET/TTL in der Console ausführen
- [ ] Cache-Aside Pattern erklären und implementieren
- [ ] Verbindungsstring sicher in Key Vault speichern und per App Setting referenzieren

---

Weiter zu [Lernpfad 4 – Identity & Access Management](modul-21-entra.md) →

!!! info "Lernpfad 4 – Identity & Access"
    In Lernpfad 4 geht es um Microsoft Entra ID (ehemals Azure AD), Managed Identities, RBAC und Conditional Access – also wie du steuert *wer* auf *was* in Azure zugreifen darf.
