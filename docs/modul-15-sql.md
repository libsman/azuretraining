# Modul 15 – Azure SQL Database: Managed Datenbank in der Cloud

## Lernziele

Nach diesem Modul kannst du:

- Eine Azure SQL Database (Managed Service) erstellen und konfigurieren
- Den Unterschied zwischen Azure SQL, SQL Managed Instance und SQL Server auf VM erklären
- Die Datenbank über das Azure Portal und die Query-Editor-Oberfläche abfragen
- Verbindungsstrings für Python-Anwendungen nutzen
- Firewall-Regeln für den Datenbankzugriff konfigurieren

---

## Hintergrund: Datenbanken in der Cloud

**On-Prem-Vergleich:** Ein SQL-Server on-premises bedeutet: Lizenz kaufen oder verlängern, Windows Server installieren, SQL Server installieren, Patching, Backups konfigurieren, Monitoring einrichten, Storage planen. Azure SQL Database übernimmt das alles – du bekommst eine Datenbank, die sofort fertig ist.

**Azure SQL: Drei Deployment-Optionen im Vergleich**

| Option | Kontrolle | Wartung | Kosten | Empfehlung |
|--------|-----------|---------|--------|------------|
| **SQL Database** (PaaS) | Nur Daten und Schema | Microsoft | Ab 0 € (Free) | ✅ Neuanfänger, einfache Apps |
| **SQL Managed Instance** | + SQL Agent, CLR | Microsoft | ~300 €/Monat | Migrations-Szenario on-prem→Azure |
| **SQL Server auf VM** | Volles OS + SQL | Du | VM-Kosten + Lizenz | Maximale Kontrolle nötig |

Für dieses Training: **SQL Database** – kostenloser Tarif, kein Setup, direkt nutzbar.

!!! info "Free Tier für Azure SQL"
    Azure SQL Database hat seit 2023 einen echten **Free Tier**: 32 GB Speicher, 100.000 vCore-Sekunden/Monat – ausreichend für Lernprojekte. **Nur eine Free-Database pro Subscription.**

---

## SQL Server und Datenbank erstellen

In Azure SQL ist ein **Server** der logische Container für eine oder mehrere **Databases**. Der Server definiert Region, Admin-Login und Firewall-Regeln.

### Schritt 1: Neue Resource Group

1. Navigiere zu **Resource groups** → **+ Create**

| Feld | Wert |
|------|------|
| Resource group | `rg-datenbanken` |
| Region | `West Europe` |

### Schritt 2: Azure SQL Datenbank erstellen

1. Suche im Portal nach **`SQL databases`** → **+ Create**

**Basics:**

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-datenbanken` |
| Database name | `db-aztraining` |
| Server | **Create new** |

**Neuen Server erstellen** (erscheint als Seitenbereich):

| Feld | Wert |
|------|------|
| Server name | `sql-aztraining-XXXX` (weltweit eindeutig) |
| Location | `West Europe` |
| Authentication method | `Use SQL authentication` |
| Server admin login | `sqladmin` |
| Password | Sicheres Passwort (notiere es!) |

Klicke **OK**.

Zurück im Hauptformular:

| Feld | Wert |
|------|------|
| Want to use SQL elastic pool? | `No` |
| Workload environment | `Development` |
| Compute + storage | Klicke **Configure database** |

**Compute + storage konfigurieren:**

1. Wähle **General Purpose** → scrollen nach unten
2. Wähle **Serverless** als Compute tier
3. Aktiviere **Free offer** falls verfügbar (erscheint wenn noch keine Free DB in der Subscription)
4. Falls kein Free Tier: wähle **Serverless**, 1 vCore minimum → ~3 €/Monat

Klicke **Apply** → zurück im Hauptformular.

!!! info "Serverless vs. Provisioned"
    - **Serverless**: DB pausiert automatisch nach Inaktivität → 0 Kosten wenn nicht genutzt. Erster Aufruf nach Pause: ~30 Sekunden Aufwärmzeit
    - **Provisioned**: Läuft immer, sofortige Reaktionszeit – für Produktion

**Backup redundancy:** `Locally-redundant backup storage` (günstiger für Test)

### Schritt 3: Networking

Wechsle zum Tab **Networking**:

| Feld | Wert |
|------|------|
| Connectivity method | `Public endpoint` |
| Allow Azure services... | `Yes` |
| Add current client IP | `Yes` |

!!! warning "Public Endpoint für dieses Training"
    Im Produktionsbetrieb würde man Private Endpoints nutzen (Modul 13). Für dieses Training erlauben wir Public Access mit deiner IP.

### Schritt 4: Review + Create

**Review + create** → **Create** (Deployment dauert ~2–3 Minuten)

---

## Erste Abfragen im Portal

### Query Editor öffnen

1. Gehe zur erstellten Datenbank `db-aztraining`
2. Klicke links im Menü auf **Query editor (preview)**
3. Melde dich an:
   - Login: `sqladmin`
   - Password: dein Passwort

### Tabelle erstellen und befüllen

Füge ins Abfragefenster ein und klicke **Run**:

```sql
CREATE TABLE mitarbeiter (
    id          INT IDENTITY(1,1) PRIMARY KEY,
    name        NVARCHAR(100)  NOT NULL,
    abteilung   NVARCHAR(50),
    eingestellt DATE           NOT NULL DEFAULT GETDATE()
);

INSERT INTO mitarbeiter (name, abteilung, eingestellt) VALUES
    ('Anna Müller',    'IT-Infrastruktur', '2024-03-01'),
    ('Bernd Schmidt',  'Cloud Operations', '2023-11-15'),
    ('Clara Weber',    'IT-Infrastruktur', '2025-01-10'),
    ('David Hoffmann', 'DevOps',           '2024-07-22');

SELECT * FROM mitarbeiter ORDER BY eingestellt;
```

Du siehst die Ergebnistabelle direkt im Browser.

### Weitere Abfragen

```sql
-- Mitarbeiter pro Abteilung zählen
SELECT abteilung, COUNT(*) AS anzahl
FROM mitarbeiter
GROUP BY abteilung
ORDER BY anzahl DESC;

-- Nur IT-Infrastruktur
SELECT name, eingestellt
FROM mitarbeiter
WHERE abteilung = 'IT-Infrastruktur'
ORDER BY eingestellt;
```

!!! success "Managed SQL Datenbank läuft!"
    Kein SQL Server installiert, kein Betriebssystem konfiguriert. Du schreibst SQL – Azure kümmert sich um den Rest.

---

## Datenbank über Python abfragen

Öffne die Cloud Shell im Portal und installiere das nötige Paket:

```bash
pip install pyodbc --quiet
```

Erstelle eine neue Datei:

```bash
code sql_demo.py
```

Füge folgenden Code ein und ersetze die Platzhalter:

```python
import pyodbc

# Verbindungsstring – Werte aus dem Portal kopieren
SERVER   = "sql-aztraining-XXXX.database.windows.net"
DATABASE = "db-aztraining"
USERNAME = "sqladmin"
PASSWORD = "DEIN_PASSWORT"   # In Produktion: aus Key Vault (Modul 12)!

connection_string = (
    f"DRIVER={{ODBC Driver 18 for SQL Server}};"
    f"SERVER={SERVER};"
    f"DATABASE={DATABASE};"
    f"UID={USERNAME};"
    f"PWD={PASSWORD};"
    "Encrypt=yes;TrustServerCertificate=no;"
)

conn   = pyodbc.connect(connection_string)
cursor = conn.cursor()

# Alle Mitarbeiter abfragen
cursor.execute("SELECT id, name, abteilung, eingestellt FROM mitarbeiter ORDER BY name")
rows = cursor.fetchall()

print(f"{'ID':<4} {'Name':<20} {'Abteilung':<20} {'Eingestellt'}")
print("-" * 60)
for row in rows:
    print(f"{row.id:<4} {row.name:<20} {row.abteilung:<20} {str(row.eingestellt)}")

# Neuen Eintrag hinzufügen
cursor.execute(
    "INSERT INTO mitarbeiter (name, abteilung, eingestellt) VALUES (?, ?, ?)",
    ("Eva Fischer", "Cloud Security", "2026-06-05")
)
conn.commit()
print("\n✅ Neuer Mitarbeiter eingetragen.")

cursor.close()
conn.close()
```

Ausführen:

```bash
python sql_demo.py
```

!!! tip "Verbindungsstring aus dem Portal"
    `db-aztraining` → **Connection strings** → Tab **ODBC** – dort findest du den fertigen Connection String mit deinem Server-Namen.

---

## Automatische Backups und Point-in-Time-Restore

Azure SQL erstellt automatisch:

- **Vollbackup** wöchentlich
- **Differenzielles Backup** alle 12 Stunden
- **Transaktionslog-Backup** alle 5–10 Minuten

Das bedeutet: du kannst die Datenbank auf **jede beliebige Minute** der letzten 7 Tage (Standard-Tier) zurücksetzen – ohne eigene Backup-Lösung.

**Wo zu sehen:** `db-aztraining` → **Restore** oben in der Menüleiste.

---

## Challenge

!!! question "Challenge: Ansicht erstellen"
    Erstelle eine SQL-**View** `v_it_mitarbeiter` die alle Mitarbeiter aus der Abteilung `IT-Infrastruktur` zurückgibt, sortiert nach Einstellungsdatum. Frage die View anschließend im Query Editor ab.

??? success "Hinweis"
    ```sql
    CREATE VIEW v_it_mitarbeiter AS
    SELECT id, name, eingestellt
    FROM mitarbeiter
    WHERE abteilung = 'IT-Infrastruktur';
    
    -- View abfragen:
    SELECT * FROM v_it_mitarbeiter ORDER BY eingestellt;
    ```

---

Weiter zu [Modul 16 – Azure Cosmos DB: NoSQL-Datenbank](modul-16-cosmosdb.md) →
