# Modul 46 – Abschlussprojekt 2: Event-Driven Architektur

## Projektübersicht

In diesem Abschlussprojekt baust du eine **event-getriebene Architektur**: Nachrichten werden über einen Service Bus gesendet, von Azure Functions verarbeitet und in Cosmos DB gespeichert. Dieses Muster findest du in Bestellsystemen, Benachrichtigungspipelines und IoT-Plattformen.

**Architektur:**

```
Producer (HTTP-Trigger Function)
    │  sendet Nachricht
    ▼
Azure Service Bus Queue
    │  löst aus
    ▼
Processor (Service Bus Trigger Function)
    │  speichert Ergebnis
    ▼
Azure Cosmos DB (NoSQL)
    │
Result API (HTTP-Trigger Function)
    │  liest Ergebnisse
    ▼
Client (curl / Browser)
```

**Was du baust:** Ein Auftragssystem. Der Producer nimmt Bestellungen entgegen, legt sie in den Service Bus, der Processor verarbeitet sie asynchron und speichert das Ergebnis in Cosmos DB. Die Result API gibt alle verarbeiteten Bestellungen zurück.

!!! info "Voraussetzungen"
    Lernpfad 1 (Modul 6 – Azure Functions) und Lernpfad 3 (Modul 16 – Cosmos DB) sollten abgeschlossen sein.

---

## Schritt 1: Infrastruktur erstellen

```bash
az group create --name rg-projekt2 --location westeurope

SUFFIX=$RANDOM

# Storage Account (für Azure Functions benötigt)
az storage account create \
  --name stfunc${SUFFIX} \
  --resource-group rg-projekt2 \
  --sku Standard_LRS \
  --allow-blob-public-access false

# Service Bus Namespace + Queue
az servicebus namespace create \
  --name sb-projekt2-${SUFFIX} \
  --resource-group rg-projekt2 \
  --sku Basic

SB_NAME=$(az servicebus namespace list --resource-group rg-projekt2 --query "[0].name" -o tsv)

az servicebus queue create \
  --namespace-name $SB_NAME \
  --resource-group rg-projekt2 \
  --name orders

# Cosmos DB (serverless für Training – kaum Kosten)
az cosmosdb create \
  --name cosmos-projekt2-${SUFFIX} \
  --resource-group rg-projekt2 \
  --locations regionName=westeurope \
  --capabilities EnableServerless

COSMOS_NAME=$(az cosmosdb list --resource-group rg-projekt2 --query "[0].name" -o tsv)

az cosmosdb sql database create \
  --account-name $COSMOS_NAME \
  --resource-group rg-projekt2 \
  --name ordersdb

az cosmosdb sql container create \
  --account-name $COSMOS_NAME \
  --resource-group rg-projekt2 \
  --database-name ordersdb \
  --name orders \
  --partition-key-path "/customerId"

# Function App
az functionapp create \
  --resource-group rg-projekt2 \
  --consumption-plan-location westeurope \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name func-projekt2-${SUFFIX} \
  --storage-account stfunc${SUFFIX} \
  --os-type linux

FUNC_APP=$(az functionapp list --resource-group rg-projekt2 --query "[0].name" -o tsv)
```

---

## Schritt 2: Verbindungsstrings konfigurieren

```bash
# Service Bus Connection String
SB_CONN=$(az servicebus namespace authorization-rule keys list \
  --namespace-name $SB_NAME \
  --resource-group rg-projekt2 \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv)

# Cosmos DB Connection String
COSMOS_CONN=$(az cosmosdb keys list \
  --name $COSMOS_NAME \
  --resource-group rg-projekt2 \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv)

# App Settings der Function App setzen
az functionapp config appsettings set \
  --name $FUNC_APP \
  --resource-group rg-projekt2 \
  --settings \
    "ServiceBusConnection=$SB_CONN" \
    "CosmosDBConnection=$COSMOS_CONN" \
    "COSMOS_DB=ordersdb" \
    "COSMOS_CONTAINER=orders" \
    "SERVICE_BUS_QUEUE=orders"
```

---

## Schritt 3: Azure Functions schreiben

Erstelle den Projektordner:

```bash
mkdir order-functions && cd order-functions
func init --python
```

### Function 1: HTTP-Trigger – Bestellung aufnehmen

```bash
func new --name PlaceOrder --template "HTTP trigger" --authlevel anonymous
```

`PlaceOrder/__init__.py`:
```python
import json
import uuid
import logging
import azure.functions as func
from azure.servicebus import ServiceBusClient, ServiceBusMessage
import os

def main(req: func.HttpRequest) -> func.HttpResponse:
    try:
        body = req.get_json()
    except ValueError:
        return func.HttpResponse("Invalid JSON", status_code=400)

    customer_id = body.get("customerId")
    items = body.get("items", [])

    if not customer_id or not items:
        return func.HttpResponse("customerId and items required", status_code=400)

    order = {
        "id": str(uuid.uuid4()),
        "customerId": customer_id,
        "items": items,
        "status": "pending"
    }

    sb_conn = os.environ["ServiceBusConnection"]
    queue_name = os.environ["SERVICE_BUS_QUEUE"]

    with ServiceBusClient.from_connection_string(sb_conn) as client:
        sender = client.get_queue_sender(queue_name=queue_name)
        with sender:
            sender.send_messages(ServiceBusMessage(json.dumps(order)))

    logging.info(f"Order {order['id']} queued for customer {customer_id}")
    return func.HttpResponse(
        json.dumps({"orderId": order["id"], "status": "queued"}),
        mimetype="application/json",
        status_code=202
    )
```

### Function 2: Service Bus Trigger – Bestellung verarbeiten

```bash
func new --name ProcessOrder --template "Azure Service Bus Queue trigger"
```

`ProcessOrder/__init__.py`:
```python
import json
import logging
import os
import azure.functions as func
from azure.cosmos import CosmosClient

def main(msg: func.ServiceBusMessage):
    order = json.loads(msg.get_body().decode("utf-8"))

    # Bestellung "verarbeiten" (hier: Status setzen)
    order["status"] = "processed"
    order["processedAt"] = __import__("datetime").datetime.utcnow().isoformat()

    # In Cosmos DB speichern
    cosmos_conn = os.environ["CosmosDBConnection"]
    db_name = os.environ["COSMOS_DB"]
    container_name = os.environ["COSMOS_CONTAINER"]

    client = CosmosClient.from_connection_string(cosmos_conn)
    db = client.get_database_client(db_name)
    container = db.get_container_client(container_name)
    container.upsert_item(order)

    logging.info(f"Order {order['id']} processed and saved to Cosmos DB")
```

`ProcessOrder/function.json`:
```json
{
  "scriptFile": "__init__.py",
  "bindings": [
    {
      "name": "msg",
      "type": "serviceBusTrigger",
      "direction": "in",
      "queueName": "orders",
      "connection": "ServiceBusConnection"
    }
  ]
}
```

### Function 3: HTTP-Trigger – Bestellungen abrufen

```bash
func new --name GetOrders --template "HTTP trigger" --authlevel anonymous
```

`GetOrders/__init__.py`:
```python
import json
import os
import azure.functions as func
from azure.cosmos import CosmosClient

def main(req: func.HttpRequest) -> func.HttpResponse:
    cosmos_conn = os.environ["CosmosDBConnection"]
    db_name = os.environ["COSMOS_DB"]
    container_name = os.environ["COSMOS_CONTAINER"]

    customer_id = req.params.get("customerId")

    client = CosmosClient.from_connection_string(cosmos_conn)
    db = client.get_database_client(db_name)
    container = db.get_container_client(container_name)

    if customer_id:
        query = f"SELECT * FROM c WHERE c.customerId = '{customer_id}' ORDER BY c._ts DESC"
    else:
        query = "SELECT * FROM c ORDER BY c._ts DESC OFFSET 0 LIMIT 50"

    items = list(container.query_items(query=query, enable_cross_partition_query=True))

    return func.HttpResponse(
        json.dumps(items),
        mimetype="application/json"
    )
```

`requirements.txt`:
```
azure-functions
azure-servicebus==7.12.1
azure-cosmos==4.7.0
```

---

## Schritt 4: Deployen und testen

```bash
# Pakete installieren
pip install -r requirements.txt

# Deployen
func azure functionapp publish $FUNC_APP

FUNC_URL="https://${FUNC_APP}.azurewebsites.net/api"

# Bestellung aufgeben
curl -X POST "$FUNC_URL/PlaceOrder" \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "customer-001",
    "items": [
      {"product": "Azure T-Shirt", "quantity": 2, "price": 29.99},
      {"product": "Cloud Mug", "quantity": 1, "price": 12.99}
    ]
  }'

# Kurz warten (Verarbeitung asynchron)
sleep 5

# Ergebnis prüfen
curl "$FUNC_URL/GetOrders?customerId=customer-001"
```

!!! success "Asynchrone Verarbeitung"
    Der `PlaceOrder`-Aufruf antwortet sofort mit `202 Accepted` – die eigentliche Verarbeitung passiert im Hintergrund. Das ist das Kernprinzip event-getriebener Architekturen: Entkopplung von Aufnahme und Verarbeitung.

---

## Schritt 5: Skalierung und Dead Letter Queue

Service Bus bietet automatisches Retry bei Fehlern:

```bash
# Dead Letter Queue prüfen (Nachrichten die 10x fehlgeschlagen sind)
az servicebus queue show \
  --namespace-name $SB_NAME \
  --resource-group rg-projekt2 \
  --name orders \
  --query "countDetails.deadLetterMessageCount"
```

Wenn du in `ProcessOrder` absichtlich einen Fehler auslöst, landen die Nachrichten nach 10 Versuchen in der Dead Letter Queue.

---

## Aufräumen

```bash
az group delete --name rg-projekt2 --yes --no-wait
```

---

## Challenge

!!! question "Challenge: E-Mail-Benachrichtigung"
    Ergänze die Architektur um eine Benachrichtigungsfunktion:
    
    1. Erstelle eine vierte Function `NotifyCustomer` die beim Schreiben in Cosmos DB triggert (Cosmos DB Change Feed Trigger)
    2. Sende eine einfache Log-Nachricht (oder E-Mail via SendGrid) wenn eine Bestellung verarbeitet wurde
    3. Teste mit einer neuen Bestellung

??? success "Hinweis"
    ```python
    # function.json für Cosmos DB Change Feed Trigger
    {
      "bindings": [{
        "name": "documents",
        "type": "cosmosDBTrigger",
        "direction": "in",
        "databaseName": "ordersdb",
        "containerName": "orders",
        "connection": "CosmosDBConnection",
        "leaseContainerName": "leases",
        "createLeaseContainerIfNotExists": true
      }]
    }
    ```

---

Weiter zu [Modul 47 – Abschlussprojekt 3: Container-App mit CI/CD-Pipeline](modul-47-projekt-cicd.md) →
