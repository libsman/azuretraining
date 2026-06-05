# Modul 18 – Azure Cache for Redis: In-Memory-Cache für schnelle Apps

## Lernziele

Nach diesem Modul kannst du:

- Azure Cache for Redis erstellen und konfigurieren
- Den Unterschied zwischen Cache und Datenbank erklären
- Dich über die Redis Console im Portal mit dem Cache verbinden
- Das Cache-Aside-Pattern in Python implementieren
- Session-Daten, Token und häufig abgefragte Daten sinnvoll cachen

---

## Hintergrund: Warum In-Memory-Cache?

**On-Prem-Vergleich:** Viele On-Prem-Systeme nutzen lokale Caches (in-process Dictionary, Memcached) oder einen eigenen Redis-Server. Das Problem: Bei mehreren App-Instanzen (Scale-Out) haben alle eine eigene Kopie der Daten – inkonsistent. Ein **zentraler Cache-Service** löst das.

**Was ist Redis?**

Redis (Remote Dictionary Server) ist eine extrem schnelle In-Memory-Key-Value-Datenbank. Typische Antwortzeiten: **<1 Millisekunde** – verglichen mit 5–50 ms für eine SQL-Abfrage.

**Typische Anwendungsfälle:**

| Use Case | Beispiel |
|----------|---------|
| **Query-Cache** | Teures SQL-Ergebnis 5 Min. cachen |
| **Session-Store** | User-Sitzungsdaten für stateless Web Apps |
| **Rate Limiting** | API-Aufrufe pro Minute zählen und begrenzen |
| **Leaderboard** | Ranking-Listen mit Sorted Sets |
| **Pub/Sub** | Nachrichten zwischen Diensten verteilen |
| **Job Queue** | Worker-Tasks in Liste einreihen |

**Redis im Vergleich:**

| | Redis (Cache) | Azure SQL / PostgreSQL (DB) |
|--|---------------|----------------------------|
| Speicherort | RAM | SSD/HDD |
| Antwortzeit | < 1 ms | 5–50 ms |
| Datenpersistenz | Optional (flüchtig) | Dauerhaft |
| Datenstruktur | Key-Value, Listen, Sets, Sorted Sets | Tabellen mit Relationen |
| Kosten (Basic C0) | ~14 €/Monat | Variabel |

---

## Azure Cache for Redis erstellen

### Schritt 1: Dienst öffnen

1. Suche im Portal nach **`Azure Cache for Redis`** → **+ Create**

### Schritt 2: Konfigurieren

| Feld | Wert |
|------|------|
| Subscription | deine Subscription |
| Resource group | `rg-datenbanken` |
| DNS name | `redis-aztraining-XXXX` (weltweit eindeutig) |
| Location | `West Europe` |
| Cache type | **C0 Basic** (250 MB) |

!!! info "Cache Tier-Übersicht"
    | Tier | RAM | Preis | Geeignet für |
    |------|-----|-------|-------------|
    | **C0 Basic** | 250 MB | ~14 €/Monat | Test, Entwicklung |
    | C1 Standard | 1 GB | ~50 €/Monat | Produktion mit Replikation |
    | C2 Premium | 6 GB | ~200 €/Monat | Persistence, VNet, Cluster |
    
    Basic hat **keine Replikation** und **kein SLA** – ausschließlich für Entwicklung.

### Schritt 3: Review + Create

**Review + create** → **Create** (~5–10 Minuten – Redis-Provisioning dauert etwas länger)

---

## Erste Schritte: Redis Console

### Redis Console im Portal

1. Gehe zu `redis-aztraining-XXXX`
2. Klicke links im Menü auf **Console**
3. Die Console öffnet sich unten – warte bis "Connected" erscheint

### Grundlegende Redis-Befehle

```redis
# String setzen und auslesen
SET benutzer:1001 "Anna Müller"
GET benutzer:1001

# Mit Ablaufzeit (TTL) setzen – 60 Sekunden
SET session:abc123 "user_id=1001" EX 60
TTL session:abc123

# Ablaufzeit prüfen (nach einigen Sekunden)
TTL session:abc123
```

```redis
# Hash – mehrere Felder unter einem Key
HSET produkt:001 name "ThinkPad X1" preis "1299.99" lager "15"
HGET produkt:001 name
HGETALL produkt:001

# Inkrementieren (z.B. für Zähler/Rate Limiting)
SET api_aufrufe:heute 0
INCR api_aufrufe:heute
INCR api_aufrufe:heute
INCR api_aufrufe:heute
GET api_aufrufe:heute
```

```redis
# Liste – für Queues und History
LPUSH verlauf:user1001 "Seite-Home"
LPUSH verlauf:user1001 "Seite-Produkte"
LPUSH verlauf:user1001 "Seite-Warenkorb"
LRANGE verlauf:user1001 0 -1

# Alle Keys auflisten
KEYS *

# Key löschen
DEL benutzer:1001
```

!!! success "Redis Console ausprobiert!"
    Du hast die grundlegenden Datenstrukturen von Redis kennengelernt: Strings, Hashes, Listen.

---

## Cache-Aside Pattern in Python

Das häufigste Muster beim Caching: **"Schau erst im Cache nach, dann in der DB"**.

```
Anfrage → Cache vorhanden? → JA → Direkt zurückgeben (schnell)
                           → NEIN → DB abfragen → In Cache speichern → Zurückgeben
```

### Verbindungsdetails holen

1. Gehe zu `redis-aztraining-XXXX` → **Access keys**
2. Kopiere **Primary connection string** (Format: `redis-aztraining-XXXX.redis.cache.windows.net:6380,password=...,ssl=True`)

### Python-Demo (Cloud Shell)

```bash
pip install redis --quiet
```

```bash
code redis_demo.py
```

```python
import redis
import json
import time

# Verbindung (Connection String aus dem Portal)
HOST     = "redis-aztraining-XXXX.redis.cache.windows.net"
PORT     = 6380
PASSWORD = "DEIN_ACCESS_KEY"

r = redis.Redis(
    host=HOST,
    port=PORT,
    password=PASSWORD,
    ssl=True,
    decode_responses=True
)

# Verbindungstest
r.ping()
print("✅ Redis-Verbindung erfolgreich")


def simuliere_db_abfrage(produkt_id: str) -> dict:
    """Simuliert eine langsame Datenbankabfrage (500ms)."""
    print(f"  🗄️  DB-Abfrage für Produkt {produkt_id}...")
    time.sleep(0.5)  # Simulierte DB-Latenz
    return {
        "id": produkt_id,
        "name": "ThinkPad X1 Carbon",
        "preis": 1299.99,
        "lager": 15
    }


def get_produkt(produkt_id: str) -> dict:
    """Cache-Aside Pattern: erst Cache, dann DB."""
    cache_key = f"produkt:{produkt_id}"

    # 1. Cache prüfen
    cached = r.get(cache_key)
    if cached:
        print(f"  ⚡ Cache-HIT  für {cache_key}")
        return json.loads(cached)

    # 2. Kein Cache-Hit → DB abfragen
    print(f"  💤 Cache-MISS für {cache_key}")
    daten = simuliere_db_abfrage(produkt_id)

    # 3. Ergebnis im Cache speichern (5 Minuten TTL)
    r.setex(cache_key, 300, json.dumps(daten))
    return daten


# Demo
print("\n--- Erster Aufruf (kein Cache) ---")
start = time.time()
produkt = get_produkt("prod-001")
print(f"  Dauer: {(time.time()-start)*1000:.0f} ms | {produkt['name']}")

print("\n--- Zweiter Aufruf (aus Cache) ---")
start = time.time()
produkt = get_produkt("prod-001")
print(f"  Dauer: {(time.time()-start)*1000:.0f} ms | {produkt['name']}")

print("\n--- TTL prüfen ---")
ttl = r.ttl("produkt:prod-001")
print(f"  Cache läuft ab in: {ttl} Sekunden")

# Cache-Invalierung bei Änderung
print("\n--- Cache-Invalierung ---")
r.delete("produkt:prod-001")
print("  Cache-Eintrag gelöscht (z.B. nach Preisänderung)")
```

```bash
python redis_demo.py
```

Die Ausgabe zeigt deutlich: erster Aufruf ~500 ms, zweiter Aufruf < 5 ms.

---

## Rate Limiting mit Redis

Ein praktisches Beispiel: API-Rate-Limiting mit Redis INCR + EXPIRE.

```python
def check_rate_limit(user_id: str, max_anfragen: int = 5, zeitfenster: int = 60) -> bool:
    """
    Gibt True zurück wenn die Anfrage erlaubt ist,
    False wenn das Limit überschritten wurde.
    """
    key = f"ratelimit:{user_id}:{int(time.time() // zeitfenster)}"
    
    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, zeitfenster)
    ergebnis = pipe.execute()
    
    anfragen_count = ergebnis[0]
    erlaubt = anfragen_count <= max_anfragen
    print(f"  User {user_id}: {anfragen_count}/{max_anfragen} Anfragen → {'✅ OK' if erlaubt else '🚫 LIMIT'}")
    return erlaubt


print("\n--- Rate Limiting Demo ---")
for i in range(7):
    check_rate_limit("user-1001")
```

---

## Challenge

!!! question "Challenge: Session-Store"
    Implementiere einen einfachen Session-Store in Python:
    
    1. Funktion `erstelle_session(user_id, data)` → erzeugt einen zufälligen Session-Token (UUID), speichert `data` als JSON mit 30 Minuten TTL
    2. Funktion `hole_session(token)` → gibt die Session-Daten zurück oder `None` wenn abgelaufen
    3. Funktion `loesche_session(token)` → Logout
    
    Teste mit einem simulierten Login-Flow.

??? success "Hinweis"
    ```python
    import uuid
    import json
    
    def erstelle_session(user_id: str, data: dict) -> str:
        token = str(uuid.uuid4())
        session_data = {"user_id": user_id, **data}
        r.setex(f"session:{token}", 1800, json.dumps(session_data))
        return token
    
    def hole_session(token: str) -> dict | None:
        raw = r.get(f"session:{token}")
        return json.loads(raw) if raw else None
    
    def loesche_session(token: str) -> None:
        r.delete(f"session:{token}")
    
    # Test:
    token = erstelle_session("user-1001", {"name": "Anna Müller", "rolle": "admin"})
    print(f"Token: {token}")
    print(f"Session: {hole_session(token)}")
    loesche_session(token)
    print(f"Nach Logout: {hole_session(token)}")  # None
    ```

---

Weiter zu [Modul 19 – App + Datenbank sicher verbinden](modul-19-app-datenbank.md) →
