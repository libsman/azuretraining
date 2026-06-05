# Modul 17 – Azure Database for PostgreSQL: Open-Source-DB als Service

## Lernziele

Nach diesem Modul kannst du:

- Azure Database for PostgreSQL Flexible Server erstellen und konfigurieren
- Dich über die Cloud Shell mit `psql` mit der Datenbank verbinden
- SQL-Schemas, Tabellen und Daten in PostgreSQL anlegen
- Den Unterschied zwischen PostgreSQL und Azure SQL einschätzen
- Verbindungsstrings für Anwendungen aus dem Portal kopieren

---

## Hintergrund: PostgreSQL als Managed Service

**On-Prem-Vergleich:** Wer PostgreSQL kennt, weiß den Aufwand: `apt install postgresql`, `pg_hba.conf` bearbeiten, Benutzer anlegen, Backups mit `pg_dump` einrichten, Patches manuell einspielen, Replikation konfigurieren. **Azure Database for PostgreSQL** übernimmt das alles – du hast sofort eine fertige PostgreSQL-Instanz.

**Warum PostgreSQL statt Azure SQL?**

| | Azure SQL | Azure DB for PostgreSQL |
|--|-----------|------------------------|
| SQL-Dialekt | T-SQL (Microsoft) | Standard-SQL + PostgreSQL-Erweiterungen |
| Lizenz | Proprietär | Open Source (PostgreSQL) |
| Erweiterungen | Eingeschränkt | 50+ Extensions (PostGIS, pg_trgm, uuid-ossp...) |
| Hersteller-Unabhängigkeit | Microsoft-Ökosystem | Open Source, portabel |
| Preis | Ab 0 € (Free Tier) | Ab ~10 €/Monat (Burstable B1ms) |
| Eignet sich für | .NET/Azure-native Apps | Linux/Open-Source-Stacks, Migration von On-Prem Postgres |

!!! info "Deployment-Optionen"
    Azure bietet zwei Varianten: **Flexible Server** (empfohlen, neuere Generation, mehr Kontrolle, Hochverfügbarkeit) und den älteren Single Server (auslaufend). Wir nutzen Flexible Server.

---

## Flexible Server erstellen

### Schritt 1: Dienst öffnen

1. Suche im Portal nach **`Azure Database for PostgreSQL flexible servers`** → **+ Create**

### Schritt 2: Basics konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-datenbanken` |
| Server name | `psql-aztraining-XXXX` (weltweit eindeutig) |
| Region | `West Europe` |
| PostgreSQL version | `16` (aktuell) |
| Workload type | `Development` |

**Compute + Storage:**

Klicke auf **Configure server**:

| Feld | Wert |
|------|------|
| Compute tier | `Burstable` |
| Compute size | `Standard_B1ms` (1 vCore, 2 GB RAM) |
| Storage size | `32 GB` |
| Performance tier | `P4` |

!!! info "Burstable B1ms – ~10 €/Monat"
    Die kleinste Flex-Server-Option. "Burstable" bedeutet: Normalerweise wenig CPU, bei Bedarf kurze Bursts auf 100%. Perfekt für Entwicklungs- und Trainingszwecke.

**Authentication:**

| Feld | Wert |
|------|------|
| Authentication method | `PostgreSQL authentication only` |
| Admin username | `pgadmin` |
| Password | Sicheres Passwort (notiere es!) |

### Schritt 3: Networking

Wechsle zum Tab **Networking**:

| Feld | Wert |
|------|------|
| Connectivity method | `Public access (allowed IP addresses)` |
| Allow public access from any... | ❌ Nicht aktivieren |
| + Add current client IP address | ✅ Aktivieren |

!!! warning "Firewall wichtig"
    Nur mit eingetragener IP-Adresse kannst du dich verbinden. Bei wechselnder IP (Home Office, WLAN) musst du die Firewall-Regel aktualisieren: Server → **Networking** → neue Regel hinzufügen.

### Schritt 4: Review + Create

**Review + create** → **Create** (~3–5 Minuten)

---

## Verbindung über Cloud Shell

### psql in der Cloud Shell

1. Öffne die **Cloud Shell** im Portal (oben rechts `>_`)
2. Stelle sicher, dass du **Bash** verwendest

```bash
# Verbinden (Servername aus dem Portal kopieren)
psql "host=psql-aztraining-XXXX.postgres.database.azure.com \
      port=5432 \
      dbname=postgres \
      user=pgadmin \
      sslmode=require"
```

Nach Eingabe des Passworts bist du im PostgreSQL-Prompt `postgres=#`.

!!! tip "SSL Pflicht"
    Azure Database for PostgreSQL erfordert SSL/TLS. Ohne `sslmode=require` wird die Verbindung abgelehnt.

### Neue Datenbank anlegen

```sql
-- Datenbank für unser Projekt erstellen
CREATE DATABASE inventar;

-- Zur neuen Datenbank wechseln
\c inventar

-- Bestätigung: Prompt ändert sich zu "inventar=#"
```

---

## Schema und Tabellen erstellen

```sql
-- Schema (Namespace) erstellen
CREATE SCHEMA lager;

-- Tabelle: Kategorien
CREATE TABLE lager.kategorien (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(50) NOT NULL UNIQUE,
    beschreibung TEXT
);

-- Tabelle: Artikel
CREATE TABLE lager.artikel (
    id           SERIAL PRIMARY KEY,
    kategorie_id INT REFERENCES lager.kategorien(id),
    bezeichnung  VARCHAR(100) NOT NULL,
    ean          VARCHAR(13) UNIQUE,
    bestand      INT DEFAULT 0 CHECK (bestand >= 0),
    preis        NUMERIC(10,2),
    erstellt_am  TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Index für häufige Suche
CREATE INDEX idx_artikel_kategorie ON lager.artikel(kategorie_id);
```

### Daten einfügen

```sql
-- Kategorien befüllen
INSERT INTO lager.kategorien (name, beschreibung) VALUES
    ('Hardware',    'Computer, Peripherie'),
    ('Software',    'Lizenzen und Abonnements'),
    ('Verbrauch',   'Kabel, Adapter, Verbrauchsmaterial');

-- Artikel befüllen
INSERT INTO lager.artikel (kategorie_id, bezeichnung, ean, bestand, preis) VALUES
    (1, 'Tastatur USB',        '4001234567890', 23,  29.99),
    (1, 'Maus kabellos',       '4001234567891', 15,  39.99),
    (1, 'Monitor 27 Zoll',     '4001234567892',  4, 289.00),
    (2, 'Office 365 Lizenz',   NULL,             50,  99.00),
    (3, 'USB-C Kabel 1m',      '4001234567893', 200,  7.99),
    (3, 'HDMI Adapter',        '4001234567894',  88, 12.50);
```

### Abfragen

```sql
-- JOIN: Artikel mit Kategorienamen
SELECT
    k.name         AS kategorie,
    a.bezeichnung,
    a.bestand,
    a.preis
FROM lager.artikel a
JOIN lager.kategorien k ON a.kategorie_id = k.id
ORDER BY k.name, a.bezeichnung;

-- Aggregation: Gesamtbestand und Wert pro Kategorie
SELECT
    k.name                              AS kategorie,
    COUNT(a.id)                         AS artikel_anzahl,
    SUM(a.bestand)                      AS gesamtbestand,
    ROUND(SUM(a.bestand * a.preis), 2)  AS lagerwert_eur
FROM lager.kategorien k
LEFT JOIN lager.artikel a ON k.id = a.kategorie_id
GROUP BY k.name
ORDER BY lagerwert_eur DESC NULLS LAST;
```

!!! success "PostgreSQL-Standard-SQL"
    `SERIAL`, `NUMERIC`, `WITH TIME ZONE`, `LEFT JOIN ... NULLS LAST` – das ist Standard-PostgreSQL-SQL, portierbar zu jeder anderen PostgreSQL-Installation.

---

## Nützliche psql-Befehle

```sql
-- Alle Tabellen im Schema anzeigen
\dt lager.*

-- Struktur einer Tabelle anzeigen
\d lager.artikel

-- Alle Datenbanken
\l

-- Datenbank wechseln
\c inventar

-- Verbindung trennen
\q
```

---

## Verbindungsstring für Anwendungen

1. Gehe zu `psql-aztraining-XXXX` → **Connection strings**
2. Tab **Python (psycopg2)** zeigt den fertigen String

Beispiel Python-Verbindung (Cloud Shell):

```bash
pip install psycopg2-binary --quiet
```

```python
import psycopg2

conn = psycopg2.connect(
    host="psql-aztraining-XXXX.postgres.database.azure.com",
    port=5432,
    dbname="inventar",
    user="pgadmin",
    password="DEIN_PASSWORT",
    sslmode="require"
)

cursor = conn.cursor()
cursor.execute("""
    SELECT k.name, a.bezeichnung, a.bestand
    FROM lager.artikel a
    JOIN lager.kategorien k ON a.kategorie_id = k.id
    ORDER BY a.bestand DESC
""")
for row in cursor.fetchall():
    print(f"{row[0]:12} | {row[1]:25} | {row[2]:>5} Stk.")

cursor.close()
conn.close()
```

---

## Challenge

!!! question "Challenge: View + Trigger"
    PostgreSQL unterstützt **Trigger** – Funktionen die automatisch bei Datenänderungen ausgelöst werden.
    
    1. Erstelle eine Tabelle `lager.bestandslog` mit den Feldern: `id` (SERIAL), `artikel_id` (INT), `aktion` (VARCHAR), `alt_bestand` (INT), `neu_bestand` (INT), `geaendert_am` (TIMESTAMPTZ DEFAULT NOW())
    2. Erstelle eine Trigger-Funktion die bei jedem UPDATE auf `lager.artikel` den alten und neuen Bestand in `lager.bestandslog` schreibt
    3. Aktualisiere den Bestand eines Artikels und prüfe ob der Eintrag im Log erscheint

??? success "Hinweis"
    ```sql
    CREATE TABLE lager.bestandslog (
        id          SERIAL PRIMARY KEY,
        artikel_id  INT,
        aktion      VARCHAR(10),
        alt_bestand INT,
        neu_bestand INT,
        geaendert_am TIMESTAMPTZ DEFAULT NOW()
    );
    
    CREATE OR REPLACE FUNCTION lager.log_bestandsaenderung()
    RETURNS TRIGGER AS $$
    BEGIN
        IF OLD.bestand <> NEW.bestand THEN
            INSERT INTO lager.bestandslog
                (artikel_id, aktion, alt_bestand, neu_bestand)
            VALUES (NEW.id, 'UPDATE', OLD.bestand, NEW.bestand);
        END IF;
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    
    CREATE TRIGGER trg_bestandslog
    AFTER UPDATE ON lager.artikel
    FOR EACH ROW EXECUTE FUNCTION lager.log_bestandsaenderung();
    
    -- Test:
    UPDATE lager.artikel SET bestand = 20 WHERE bezeichnung = 'Maus kabellos';
    SELECT * FROM lager.bestandslog;
    ```

---

Weiter zu [Modul 18 – Azure Cache for Redis: In-Memory-Cache](modul-18-redis.md) →
