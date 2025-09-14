
# Azure Service Bus – In‑Depth Guide (with Python Samples)

This repository contains notes, summaries, and runnable **Python** examples derived from the course transcript and slide decks attached to this project. It covers **queues**, **topics & subscriptions**, **filters**, **sessions & correlation**, **duplicate detection**, **dead‑lettering**, and **SAS security**.

> Sources: Course transcript and slides bundled with this repo. See inline refs.

---

## Table of Contents

1. [What is Azure Service Bus?](#what-is-azure-service-bus)
2. [Prerequisites](#prerequisites)
3. [Quick Start: Send & Receive with a Queue](#quick-start-send--receive-with-a-queue)
4. [Managing Entities (namespaces, queues, topics, subscriptions)](#managing-entities-namespaces-queues-topics-subscriptions)
5. [Publish/Subscribe with Filters](#publishsubscribe-with-filters)
6. [Message Sessions & Request/Response](#message-sessions--requestresponse)
7. [Error Handling, Retries & Dead-lettering](#error-handling-retries--dead-lettering)
8. [Duplicate Detection](#duplicate-detection)
9. [Security with SAS](#security-with-sas)
10. [CLI Cheatsheet](#cli-cheatsheet)
11. [References](#references)

---

## What is Azure Service Bus?

**Azure Service Bus** is a fully-managed enterprise messaging broker offering **durable queues** (point‑to‑point) and **topics/subscriptions** (publish‑subscribe). It supports **AMQP** and **HTTP**, provides features like **sessions**, **dead‑letter queues**, **scheduled delivery**, **message deferral**, and **duplicate detection**. fileciteturn0file2L1-L1 fileciteturn0file2L3-L7

**Key SDK classes (Python equivalents of the .NET classes described in the course):**
- `ServiceBusClient` – top-level client for senders/receivers/processors. fileciteturn0file2L3-L7
- `ServiceBusAdministrationClient` – manage queues/topics/subscriptions. fileciteturn0file4L1-L1

---

## Prerequisites

- Python 3.9+
- Packages:
  ```bash
  pip install azure-servicebus==7.*
  ```
- Connection string in env var:
  - **`SB_CONN`** (namespace-level with appropriate claims)
- Example entity names:
  - Queue: **`orders`**
  - Topic: **`ordertopic`**
  - Subscriptions: **`allOrders`**, **`usaOrders`**, **`wiretap`**

> The course recommends using the modern SDKs and clarifies the role of admin vs data-plane clients. fileciteturn0file2L3-L7

---

## Quick Start: Send & Receive with a Queue

### 1) Send messages
```python
import os, json
from azure.servicebus import ServiceBusClient, ServiceBusMessage

CONN_STR = os.environ["SB_CONN"]
QUEUE = "orders"

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    sender = sb.get_queue_sender(queue_name=QUEUE)
    with sender:
        for i in range(3):
            body = {"orderId": f"ORD-{i}", "items": i+1}
            msg = ServiceBusMessage(json.dumps(body), content_type="application/json")
            sender.send_messages(msg)
        print("Sent 3 messages")
```

### 2) Receive (peek-lock) with a processor
```python
import os, json, threading
from azure.servicebus import ServiceBusClient, ServiceBusMessage

CONN_STR = os.environ["SB_CONN"]
QUEUE = "orders"

def on_message(message):
    data = json.loads(str(message))
    print("Processing:", data)
    message.complete()  # explicitly complete in peek-lock

def on_error(error):
    print("Error:", error)

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    processor = sb.get_queue_processor(
        queue_name=QUEUE,
        max_concurrent_calls=5,  # constrained concurrency
        auto_complete_messages=False
    )
    processor.add_message_handler(on_message)
    processor.add_error_handler(on_error)
    processor.start()
    try:
        threading.Event().wait(5)  # run briefly for demo
    finally:
        processor.stop()
        processor.close()
```
Peek‑lock enables **complete/abandon/dead‑letter/defer** semantics (at‑least‑once). Receive‑and‑delete is simpler but risks message loss (at‑most‑once). fileciteturn0file3L9-L11

---

## Managing Entities (namespaces, queues, topics, subscriptions)

```python
import os
from datetime import timedelta
from azure.servicebus.management import ServiceBusAdministrationClient, QueueDescription, TopicDescription, SubscriptionDescription

CONN_STR = os.environ["SB_CONN"]
admin = ServiceBusAdministrationClient.from_connection_string(CONN_STR)

# Create a queue with custom properties (e.g., 2-minute lock, DLQ on expiration)
if not admin.get_queue_runtime_properties.__self__.queue_exists("orders") if hasattr(admin, "get_queue_runtime_properties") else False:
    q = QueueDescription(
        name="orders",
        lock_duration=timedelta(minutes=2),
        dead_lettering_on_message_expiration=True,
        max_delivery_count=5
    )
    admin.create_queue(q)

# Create a topic and subscriptions
if not admin.topic_exists("ordertopic"):
    admin.create_topic(TopicDescription(name="ordertopic"))

if not admin.subscription_exists("ordertopic", "allOrders"):
    admin.create_subscription(SubscriptionDescription(topic_name="ordertopic", subscription_name="allOrders"))
```
- **Names must be unique within a namespace**; subscriptions live under topics. fileciteturn0file4L1-L1 fileciteturn0file3L1-L1

> You can do the same via CLI; see [CLI Cheatsheet](#cli-cheatsheet). fileciteturn0file3L3-L4

---

## Publish/Subscribe with Filters

### Send to a topic
```python
import os, json
from azure.servicebus import ServiceBusClient, ServiceBusMessage

CONN_STR = os.environ["SB_CONN"]
TOPIC = "ordertopic"

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    sender = sb.get_topic_sender(topic_name=TOPIC)
    with sender:
        order = {"orderId": "US-1001", "region": "USA", "items": 42, "loyalty": True}
        msg = ServiceBusMessage(json.dumps(order), content_type="application/json")
        # Promote properties for filtering (header props)
        msg.application_properties = {"Region": "USA", "Items": 42, "Loyalty": True}
        sender.send_messages(msg)
        print("Published order to topic")
```

### Create filtered subscriptions (SQL & Correlation)

```python
from azure.servicebus.management import (
    ServiceBusAdministrationClient,
    CreateRuleOptions, SqlRuleFilter, CorrelationRuleFilter
)

admin = ServiceBusAdministrationClient.from_connection_string(CONN_STR)

# USA-only subscription using SQL filter
if not admin.subscription_exists(TOPIC, "usaOrders"):
    admin.create_subscription(topic_name=TOPIC, subscription_name="usaOrders")
    admin.create_rule(
        TOPIC, "usaOrders",
        CreateRuleOptions(name="region", filter=SqlRuleFilter("Region = 'USA'"))
    )
    # remove default True filter
    admin.delete_rule(TOPIC, "usaOrders", "$Default")

# Wiretap subscription (no filter = everything)
if not admin.subscription_exists(TOPIC, "wiretap"):
    admin.create_subscription(topic_name=TOPIC, subscription_name="wiretap")
```

> SQL filters evaluate **promoted headers**; correlation filters do equality matches (faster) on standard headers. fileciteturn0file1L1-L1

---

## Message Sessions & Request/Response

**Remember:** `CorrelationId` is for efficient routing; **use `SessionId` to group related messages** and to implement async request/response. fileciteturn0file0L3-L1

### Session-enabled queue (sender):
```python
import os, json
from azure.servicebus import ServiceBusClient, ServiceBusMessage

CONN_STR = os.environ["SB_CONN"]
SESSION_QUEUE = "session-orders"
SESSION_ID = "sess-1234"

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    sender = sb.get_queue_sender(queue_name=SESSION_QUEUE)
    with sender:
        for n in range(3):
            msg = ServiceBusMessage(json.dumps({"n": n}), session_id=SESSION_ID)
            sender.send_messages(msg)
```

### Receive by session (receiver):
```python
from azure.servicebus import ServiceBusClient

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    # Accept a specific session (or use accept_next_session=True in some helpers)
    receiver = sb.get_queue_session_receiver(queue_name=SESSION_QUEUE, session_id=SESSION_ID)
    with receiver:
        for msg in receiver:
            print("Got in session:", receiver.session_id, str(msg))
            receiver.complete_message(msg)
```
> Enabling sessions requires creating the entity with `requires_session = True`; you then **set `session_id` on each message** and use a **session receiver** to process them. fileciteturn0file0L3-L1

### Request/Response with sessions (pattern)

- Request queue (no sessions)
- Response queue (sessions enabled)
- Request message sets `reply_to_session_id`; responder replies with `session_id=that value`. fileciteturn0file1L1-L1

```python
# Requester
REQ_Q, RESP_Q = "catalog-req", "catalog-resp"
with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    req = sb.get_queue_sender(REQ_Q)
    with req:
        reply_session = "resp-0001"
        m = ServiceBusMessage('{"productId": 706}', reply_to_session_id=reply_session)
        req.send_messages(m)

    # receive response from response queue session
    resp = sb.get_queue_session_receiver(RESP_Q, session_id=reply_session)
    with resp:
        msg = next(iter(resp))  # demo
        print("Response:", str(msg))
        resp.complete_message(msg)
```

---

## Error Handling, Retries & Dead-lettering

- Use **retry with backoff** for transient errors.
- **DLQ** is available per queue and per subscription; messages can be DLQ’d automatically (e.g., **max delivery count**, expiration, routing failure) or **explicitly** with a reason/description. fileciteturn0file2L1-L1

### Explicitly dead-letter a message
```python
from azure.servicebus import ServiceBusClient

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    receiver = sb.get_queue_receiver(queue_name="orders")
    with receiver:
        for msg in receiver:
            try:
                # your validation / processing
                raise ValueError("Invalid order data")
            except Exception as ex:
                receiver.dead_letter_message(
                    msg,
                    reason="Invalid order",
                    error_description=str(ex)
                )
```

### Read from DLQ
```python
from azure.servicebus import ServiceBusClient, ServiceBusSubQueue

with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    dlq = sb.get_queue_receiver(queue_name="orders", sub_queue=ServiceBusSubQueue.DEAD_LETTER)
    with dlq:
        for dead in dlq:
            print("DLQ:", dead.application_properties.get(b"DeadLetterReason"), str(dead))
            dlq.complete_message(dead)
```

---

## Duplicate Detection

Enable duplicate detection on an entity and ensure the **sender sets `message_id`**. Duplicate messages **are ignored** (not DLQ’d). fileciteturn0file3L9-L11

```python
# Create queue with duplicate detection
from datetime import timedelta
from azure.servicebus.management import ServiceBusAdministrationClient, QueueDescription

admin = ServiceBusAdministrationClient.from_connection_string(CONN_STR)
if not admin.queue_exists("rfid-checkout"):
    admin.create_queue(QueueDescription(
        name="rfid-checkout",
        requires_duplicate_detection=True,
        duplicate_detection_history_time_window=timedelta(minutes=60)
    ))

# Sender must set message_id explicitly
from azure.servicebus import ServiceBusClient, ServiceBusMessage
with ServiceBusClient.from_connection_string(CONN_STR) as sb:
    s = sb.get_queue_sender("rfid-checkout")
    with s:
        s.send_messages(ServiceBusMessage("payload", message_id="TAG-123"))
```

---

## Security with SAS

Use **SAS policies** with minimal claims (**Send**, **Listen**, **Manage**) scoped to the needed entity; **avoid using RootManageSharedAccessKey** in production. fileciteturn0file2L3-L7

---

## CLI Cheatsheet

```bash
# Create namespace (adjust RG/NAME/LOC)
az servicebus namespace create --resource-group RG --name sbdemo01 --location westeurope

# Create queue
az servicebus queue create --name orders --namespace-name sbdemo01 --resource-group RG

# Create topic and subscription
az servicebus topic create --name ordertopic --namespace-name sbdemo01 --resource-group RG
az servicebus topic subscription create --name allOrders --topic-name ordertopic --namespace-name sbdemo01 --resource-group RG
```
> The CLI commands follow a predictable pattern; use `az servicebus <entity> <verb>`. fileciteturn0file3L3-L4

---

## References

- **Admin & Data plane classes**: overview of Service Bus clients and roles. fileciteturn0file2L3-L7 fileciteturn0file4L1-L1
- **Sessions vs Correlation**: `SessionId` groups messages; `CorrelationId` aids routing/filters. fileciteturn0file0L3-L1
- **Filters & promoted headers**: SQL and Correlation filters. fileciteturn0file1L1-L1
- **Peek-lock semantics & duplicate detection**. fileciteturn0file3L9-L11
- **CLI management**. fileciteturn0file3L3-L4

---

### Notes
- Code targets `azure-servicebus` v7.*. Minor API names may differ across patch releases.
- Adjust concurrency and retries to match downstream capacity and SLOs.

