# Modul 16 – Azure Cosmos DB: NoSQL-Datenbank für flexible Daten

## Lernziele

Nach diesem Modul kannst du:

- Den Unterschied zwischen relationalen und NoSQL-Datenbanken erklären
- Einen Cosmos DB-Account mit NoSQL API erstellen
- Dokumente über das Data Explorer Portal erstellen und abfragen
- Cosmos DB mit Python über das SDK ansprechen
- Die wichtigsten Cosmos DB-Konzepte (Container, Partition Key, RU/s) beschreiben

---

## Hintergrund: Wann NoSQL, wann SQL?

**On-Prem-Vergleich:** In vielen On-Prem-Umgebungen gibt es neben SQL Server vielleicht MongoDB oder eine Key-Value-Datenbank wie Redis. Cosmos DB ist Microsofts Antwort auf moderne NoSQL-Anforderungen – vollständig verwaltet, global replizierbar, mit mehreren APIs (NoSQL, MongoDB, Cassandra, Gremlin).

**Relationale DB vs. NoSQL im Vergleich:**

| | Azure SQL (relational) | Cosmos DB (NoSQL) |
|--|----------------------|-------------------|
| Datenstruktur | Tabellen mit festem Schema | JSON-Dokumente, flexibles Schema |
| Abfragesprache | T-SQL | SQL-ähnlich (Core API) oder MongoDB-Syntax |
| Schema-Änderungen | ALTER TABLE nötig | Einfach – jedes Dokument kann anders aussehen |
| Skalierung | Vertikal (größere VM) | Horizontal (mehr Partitionen) |
| Transaktionen | ACID vollständig | Innerhalb einer Partition |
| Eignet sich für | Strukturierte Geschäftsdaten | Kataloge, Logs, Sessions, User-Profile |

**Wann Cosmos DB wählen?**

- Daten haben kein festes Schema (z.B. Produkte mit unterschiedlichen Attributen)
- Hohe Leserate nötig (weltweit, niedrige Latenz)
- JSON-Dokumente nativ speichern ohne ORM-Mapping
- Event-driven Architektur (Change Feed)

---

## Cosmos DB Account erstellen

### Schritt 1: Dienst öffnen

1. Suche im Portal nach **`Azure Cosmos DB`** → **+ Create**
2. Wähle **Azure Cosmos DB for NoSQL** → **Create**

!!! info "Welche API?"
    Cosmos DB unterstützt mehrere APIs: NoSQL (empfohlen, native), MongoDB, Apache Cassandra, Gremlin (Graphen), Table. Für neue Projekte: NoSQL API – beste Azure-Integration.

### Schritt 2: Konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-datenbanken` |
| Account Name | `cosmos-aztraining-XXXX` (weltweit eindeutig) |
| Location | `West Europe` |
| Capacity mode | `Serverless` |
| Apply Free Tier Discount | `Apply` (falls verfügbar) |

!!! info "Serverless vs. Provisioned Throughput"
    - **Serverless**: Du zahlst pro Request (ca. 0,25 € pro 1 Mio. Request Units). Ideal für unregelmäßige Last.
    - **Provisioned**: Feste RU/s reservieren. Für konstante Produktionslast.
    
    Für dieses Training: Serverless. Kosten: nahezu 0 €.

### Schritt 3: Review + Create

**Review + create** → **Create** (~3–5 Minuten)

---

## Datenbank und Container erstellen

1. Gehe zu `cosmos-aztraining-XXXX` → klicke links auf **Data Explorer**
2. Klicke oben auf **New Container**

| Feld | Wert |
|------|------|
| Database id | **Create new** → `shop` |
| Container id | `produkte` |
| Partition key | `/kategorie` |

Klicke **OK**.

!!! info "Was ist ein Partition Key?"
    Cosmos DB verteilt Daten auf physische Partitionen. Der **Partition Key** bestimmt, welches Feld für diese Verteilung genutzt wird. Wichtige Regeln:
    
    - Hohe Kardinalität wählen (viele verschiedene Werte) → gleichmäßige Verteilung
    - Häufige Abfragen sollten den Partition Key im Filter enthalten → günstigere Abfragen
    - ❌ Schlechter Partition Key: `/land` (nur 2–3 Werte, "hot partition")
    - ✅ Guter Partition Key: `/kategorie`, `/userId`, `/productId`

---

## Dokumente erstellen

### Über Data Explorer

1. Klicke links im Data Explorer auf **`shop` → `produkte` → Items**
2. Klicke oben auf **New Item**
3. Ersetze den Inhalt durch:

```json
{
  "id": "prod-001",
  "kategorie": "Laptop",
  "name": "ThinkPad X1 Carbon",
  "preis": 1299.99,
  "lager": 15,
  "hersteller": "Lenovo",
  "tags": ["business", "ultrabook", "13-zoll"]
}
```

Klicke **Save**. Erstelle noch zwei weitere Dokumente:

```json
{
  "id": "prod-002",
  "kategorie": "Maus",
  "name": "MX Master 3",
  "preis": 79.99,
  "lager": 48,
  "hersteller": "Logitech",
  "tags": ["ergonomisch", "wireless"]
}
```

```json
{
  "id": "prod-003",
  "kategorie": "Laptop",
  "name": "Surface Laptop 5",
  "preis": 1199.00,
  "lager": 7,
  "hersteller": "Microsoft",
  "tags": ["business", "windows", "15-zoll"],
  "farbe": "Platin"
}
```

!!! success "Flexibles Schema in Aktion"
    `prod-003` hat ein zusätzliches Feld `farbe` das `prod-001` nicht hat. In SQL wäre das ein `ALTER TABLE`. In Cosmos DB ist jedes Dokument selbstständig.

---

## Abfragen im Data Explorer

1. Klicke auf **New SQL Query** (oben im Data Explorer)
2. Standard-Query ist `SELECT * FROM c` – führe sie aus

### Gefilterte Abfragen

```sql
-- Alle Laptops
SELECT * FROM c WHERE c.kategorie = "Laptop"

-- Nur Name und Preis, sortiert
SELECT c.name, c.preis FROM c ORDER BY c.preis ASC

-- Günstige Produkte
SELECT c.name, c.preis, c.kategorie
FROM c
WHERE c.preis < 100.00

-- ARRAY_CONTAINS: Produkte mit Tag "wireless"
SELECT c.name, c.tags
FROM c
WHERE ARRAY_CONTAINS(c.tags, "wireless")
```

!!! info "SQL in Cosmos DB"
    Die Abfragesprache sieht aus wie SQL, ist aber speziell für JSON-Dokumente. Statt Tabellen- und Spaltennamen nutzt du `c` als Alias für den Container und navigierst mit Punkt-Notation durch verschachtelte Felder.

---

## Cosmos DB mit Python abfragen

Öffne die Cloud Shell:

```bash
pip install azure-cosmos --quiet
```

Hole den Endpoint und Primary Key:

1. Gehe zu `cosmos-aztraining-XXXX` → **Keys**
2. Kopiere **URI** und **PRIMARY KEY**

```bash
code cosmos_demo.py
```

```python
from azure.cosmos import CosmosClient, PartitionKey

# Zugangsdaten (in Produktion: aus Key Vault – Modul 12)
ENDPOINT    = "https://cosmos-aztraining-XXXX.documents.azure.com:443/"
PRIMARY_KEY = "DEIN_PRIMARY_KEY"

client    = CosmosClient(ENDPOINT, credential=PRIMARY_KEY)
database  = client.get_database_client("shop")
container = database.get_container_client("produkte")

# Alle Produkte ausgeben
print("=== Alle Produkte ===")
for item in container.read_all_items():
    print(f"  [{item['kategorie']}] {item['name']} – {item['preis']} €")

# Gefiltert abfragen
print("\n=== Nur Laptops ===")
query = "SELECT c.name, c.preis FROM c WHERE c.kategorie = 'Laptop'"
for item in container.query_items(query=query, enable_cross_partition_query=True):
    print(f"  {item['name']:30} {item['preis']:>8.2f} €")

# Neues Dokument einfügen
neues_produkt = {
    "id": "prod-004",
    "kategorie": "Tastatur",
    "name": "MX Keys Mini",
    "preis": 99.99,
    "lager": 25,
    "hersteller": "Logitech"
}
container.create_item(body=neues_produkt)
print("\n✅ Produkt prod-004 eingefügt.")
```

```bash
python cosmos_demo.py
```

---

## Challenge

!!! question "Challenge: Change Feed erkunden"
    Cosmos DB hat einen **Change Feed** – ein Stream aller Änderungen in einem Container. Das ist die Grundlage für event-driven Architekturen (z.B. Azure Functions triggern wenn ein neues Dokument reinkommt).
    
    1. Gehe im Data Explorer zu **`shop` → `produkte` → Change Feed**
    2. Erstelle ein neues Dokument im Data Explorer
    3. Beobachte wie die Änderung im Change Feed erscheint
    
    Dokumentation: [Cosmos DB Change Feed](https://learn.microsoft.com/de-de/azure/cosmos-db/change-feed)

??? success "Hinweis"
    Der Change Feed zeigt jedes neu erstellte oder aktualisierte Dokument in der Reihenfolge der Änderung. Gelöschte Dokumente erscheinen nicht direkt (außer mit Soft Delete Pattern).
    
    In der Praxis verbindet man einen **Azure Functions Cosmos DB Trigger** mit dem Change Feed – dann wird bei jeder Änderung automatisch eine Function aufgerufen. Das kombiniert Modul 6 (Functions) mit diesem Modul.

---

Weiter zu [Modul 17 – Azure Database for PostgreSQL](modul-17-postgresql.md) →
