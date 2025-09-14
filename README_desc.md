# Azure Service Bus – In-Depth Guide

This repository contains notes, summaries, and examples from **Alan Smith’s Pluralsight course “Microsoft Azure Service Bus In-Depth”** along with companion slide decks. It explores **messaging concepts, patterns, and implementation details** for building distributed, reliable, and scalable applications with **Azure Service Bus**.

---

## 📖 Course Overview
- Delivered by: **Alan Smith**, Active Solution, Stockholm, Sweden  
- Format: Deep-dive with demos and real scenarios  
- Requirements:  
  - Working knowledge of **C#** and **Visual Studio**  
  - Access to an **Azure subscription** or Service Bus namespace  

The course covers:
- Understanding **Azure messaging services**
- Service Bus entities: **namespaces, queues, topics, subscriptions**
- Sending, receiving, and processing messages
- Advanced features: **duplicate detection, sessions, dead-lettering**
- Publish/subscribe routing and correlation
- Security with **Shared Access Signatures (SAS)**
- Real-world usage scenarios and performance tuning

---

## 🧩 Modules

### 1. Understanding Azure Service Bus
- Azure provides multiple messaging services: **Service Bus, Event Hubs, Event Grid, Relay, Storage Queues**.
- Service Bus supports:
  - **Queues** → point-to-point messaging (FIFO)
  - **Topics & Subscriptions** → publish/subscribe messaging
- Features:
  - Dead-lettering
  - Duplicate detection
  - Sessions
  - Scheduled delivery & message expiration
  - AMQP & HTTP protocols
- SDK: **Azure.Messaging.ServiceBus** (recommended) replaces **Microsoft.Azure.ServiceBus** (deprecated).

---

### 2. Service Bus Scenarios & Pricing
- **Global Azure Racing Game**:  
  - Lap times & telemetry sent to Service Bus Topics, processed by worker roles, stored in Table Storage.
- **Website Read Model**:  
  - On-prem SQL DB → updates pushed asynchronously via Service Bus → content published to Azure Blob Storage.
- **Pricing**:
  - **Basic** → limited (queues only, ~\$0.05 per million ops).
  - **Standard** → queues + topics, ~\$10/month base + ops charges.
  - **Premium** → dedicated resources, predictable latency, higher cost (~\$680/month per messaging unit).

---

### 3. Managing Namespaces & Artifacts
- **Namespace**: globally unique container for queues/topics/subscriptions.
- Management options:
  - **Azure Portal** → easiest for demos
  - **SDK (ServiceBusAdministrationClient)** → programmatic management
  - **CLI / PowerShell** → automation  
    ```bash
    az servicebus namespace create --resource-group RG --name sbdemo01 --location westeurope
    az servicebus queue create --name orders --namespace-name sbdemo01 --resource-group RG
    ```

---

### 4. Creating & Sending Messages
- Message = **Header (properties)** + **Body (binary/JSON/text)**.
- Common properties: `MessageId`, `SessionId`, `CorrelationId`, `ContentType`, `TimeToLive`, `ReplyTo`.
- Serialization: typically **JSON** (`Newtonsoft.Json`) for loose coupling.
- Sending:
  - `ServiceBusSender.SendMessageAsync(message)` for single messages.
  - `ServiceBusSender.SendMessagesAsync(batch)` for higher throughput.

---

### 5. Receiving & Processing Messages
- Modes:
  - **Receive & Delete** → at most once, possible loss.
  - **Peek-Lock** → at least once, supports complete/abandon/defer/dead-letter.
- **ServiceBusProcessor** supports efficient message pumps with constrained concurrency.
- **Duplicate detection** (set `RequiresDuplicateDetection = true`) prevents reprocessing.

---

### 6. Publish/Subscribe & Routing
- **Topics & Subscriptions** → route messages to multiple receivers.
- Subscription filters:
  - **SQL filters** (rich comparisons, e.g. `Region = 'USA' AND Items > 30`).
  - **Correlation filters** (equals only, faster).
- **Wire Tap pattern**: add a monitoring subscription to inspect all messages.

---

### 7. Message Correlation & Sessions
- **CorrelationId**: used for routing/filtering.  
- **SessionId**: used for correlating related messages across queues.
- **Request/Response** with sessions:
  - Requests sent with `ReplyToSessionId`
  - Responses sent with matching `SessionId`
  - Enables async two-way messaging.

---

### 8. Handling Errors & Poison Messages
- **Transient faults**: retry with backoff.
- **Dead-letter queues (DLQ)**:
  - Automatic (max delivery count, expiration, routing failures)
  - Explicit (`DeadLetterMessageAsync(reason, description)`)
- **Poison messages**: should be explicitly dead-lettered with diagnostic info.

---

### 9. Securing Service Bus Entities
- Default `RootManageSharedAccessKey` has full permissions (avoid in production).
- Use **Shared Access Signatures (SAS)**:
  - Assign to namespace, queue, or topic
  - Claims: **Send**, **Listen**, **Manage**
  - Supports primary/secondary key rotation

---

## 🔑 Key Takeaways
- Azure Service Bus = **enterprise-class messaging PaaS**.
- Use **Queues** for 1-to-1, **Topics/Subscriptions** for 1-to-many.
- Always plan for **errors, retries, and DLQs** in production.
- Choose **Standard** tier for most workloads; **Premium** for predictable performance.
- Use **granular SAS policies**, not the root key, in production.

---

## 📂 Repository Structure
```
/slides
   understanding-azure-service-bus-slides.pdf
   managing-service-bus-namespaces-and-artifacts-slides.pdf
   azure-service-bus-scenarios-slides.pdf
   creating-and-sending-messages-slides.pdf
   receiving-and-processing-messages-slides.pdf
   using-publish-subscribe-to-route-messages-and-correlating-messages-slides.pdf
   message-correlation-slides.pdf
   handling-poison-messages-dead-lettering-messages-and-handling-errors-slides.pdf
   securing-service-bus-entities-slides.pdf
/transcript
   MSAzureSBTranscript.txt
README.md  ← (this file)
```

---

## 📚 References
- Course: *Microsoft Azure Service Bus In-Depth* – Alan Smith (Pluralsight)
- [Azure.Messaging.ServiceBus SDK on GitHub](https://github.com/Azure/azure-sdk-for-net/tree/main/sdk/servicebus/Azure.Messaging.ServiceBus)
- [Enterprise Integration Patterns](https://www.enterpriseintegrationpatterns.com/)
- [Azure Service Bus Documentation](https://learn.microsoft.com/azure/service-bus-messaging/)
