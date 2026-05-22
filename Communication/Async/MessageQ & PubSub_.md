# System Design: Sync vs Async, Message Queues & Pub/Sub

> **Sources:**
> - algomaster.io/learn/system-design/sync-vs-async-communication
> - algomaster.io/learn/system-design/message-queues
> - algomaster.io/learn/system-design/pub-sub

---

# Part 1: Synchronous vs Asynchronous Communication

## The Core Difference

| | Synchronous | Asynchronous |
|---|---|---|
| **Model** | Request → wait → response | Publish message → continue → receiver processes later |
| **Coupling** | Tight — caller must know callee | Loose — sender doesn't know receivers |
| **Blocking** | Caller is blocked during the call | Caller continues immediately |
| **Consistency** | Strong — both agree on outcome | Eventual — data catches up over time |
| **Error handling** | Natural — errors propagate up the call stack | Complex — must handle failures separately |

---

## Synchronous Communication

**How it works:**
```
1. Client sends request
2. Client waits (blocks)
3. Server processes
4. Server responds
5. Client continues
```

**Examples:** REST APIs, gRPC, database queries, GraphQL

### The Availability Problem

Each extra synchronous hop multiplies downtime:

```
Service A (99.9%) → Service B (99.9%) → Service C (99.9%)

Combined availability = 99.9% × 99.9% × 99.9% = 99.7%
                      = ~26 hours downtime/year  (vs ~8.7 hours each)
```

More services in a chain = worse overall availability.

### ✅ Use Synchronous When:
- User needs an **immediate answer** (login, account balance, search results)
- **Strong consistency** required (payment validation, inventory check before purchase)
- Operation is **fast** (< 100ms downstream)
- **Simple systems** — don't add async complexity you don't need

---

## Asynchronous Communication

**How it works:**
```
1. Sender publishes message to queue/topic
2. Sender gets ack that message is stored
3. Sender continues with other work  ← key difference
4. Receiver pulls message when ready
5. Receiver processes and acknowledges
```

**Real example — order placement:**
```
User clicks "Place Order"
  │
  ▼
API saves order → returns "Order Confirmed" to user immediately ✅
  │
  ▼ (async, in background)
  ├── Inventory Service reserves stock
  ├── Payment Service charges card
  ├── Email Service sends confirmation
  └── Analytics Service logs event

If Email Service is slow → user experience unaffected ✅
```

### ✅ Use Asynchronous When:
- Operation can be **deferred** (send email, generate report, encode video)
- Need **resilience** — retry later if downstream is down
- **Spiky traffic** — queue absorbs bursts, smooths load
- **Multiple services** need to react to the same event
- Operation is **slow** (PDF generation, batch processing, ML inference)

---

## Hybrid Approach (Production Standard)

```
User request  →  Synchronous path  →  Fast response to user
                       │
                       ▼ (async)
              Background processing  →  Resilient, scalable
```

Most real systems use both. User-facing = sync for speed. Background work = async for resilience.

---

## Common Patterns

| Pattern | Direction | Description | Use Case |
|---|---|---|---|
| **Request-Response** | ↔ | Client waits for reply | REST, gRPC, DB queries |
| **Fire and Forget** | → | Sender doesn't care about result | Logging, analytics, audit |
| **Request-Async Response** | → then ← later | Result returned on a different channel | Long-running jobs |
| **Publish-Subscribe** | → fanout | One event → many receivers | Notifications, event-driven arch |

---

# Part 2: Message Queues

## What is a Message Queue?

An **intermediary buffer** between producers and consumers. Producers push messages in; consumers pull and process them independently.

```
Producer ──► [Queue: msg1, msg2, msg3, ...] ──► Consumer(s)
```

> Key property: **each message is delivered to exactly one consumer** (point-to-point).

---

## Core Components

| Component | Role |
|---|---|
| **Producer** | Creates and sends messages to the queue |
| **Queue** | Stores messages durably until consumed |
| **Consumer** | Pulls and processes messages; sends ack when done |
| **Message Broker** | Manages the queue infrastructure (RabbitMQ, SQS, etc.) |

---

## How It Works

```
1. Producer sends message → Broker stores it durably
2. Consumer pulls message → Broker marks it "in-flight"
3. Consumer processes message
4. Consumer sends ACK → Broker deletes message
5. If no ACK within timeout → Broker redelivers to another consumer
```

**Visibility timeout:** Message is hidden from other consumers while being processed. If processing fails/times out, it becomes visible again for redelivery.

---

## Key Advantages

| Advantage | How |
|---|---|
| **Decoupling** | Producer and consumer don't need to be running simultaneously |
| **Load leveling** | Queue absorbs traffic spikes; consumers process at their own pace |
| **Scalability** | Add more consumers to increase throughput |
| **Resilience** | If consumer crashes, message stays in queue and gets redelivered |
| **Retry logic** | Failed messages can be retried with backoff |

---

## Best Practices

| Practice | Why |
|---|---|
| **Idempotent consumers** | Messages can be delivered more than once — processing twice must be safe |
| **Dead Letter Queue (DLQ)** | After max retries, move poison messages to DLQ for inspection |
| **Exponential backoff** | Retry 1s → 2s → 4s → 8s... avoids retry storms |
| **Message TTL** | Expire stale messages that are no longer relevant |
| **Monitor queue depth** | Growing backlog = consumer is falling behind; scale up |

---

## Popular Message Queue Systems

| System | Type | Persistence | Ordering | Best For |
|---|---|---|---|---|
| **RabbitMQ** | AMQP broker | Yes | Per-queue | Complex routing, flexible topologies |
| **AWS SQS** | Managed queue | Yes | Best-effort (FIFO option) | AWS-native, serverless, simple queues |
| **Apache Kafka** | Distributed log | Yes (configurable) | Per-partition | High-throughput streaming, replay |
| **Redis Streams** | In-memory log | Optional | Yes | Low-latency, lightweight pipelines |
| **Azure Service Bus** | Managed broker | Yes | Yes (sessions) | Azure-native, enterprise messaging |

---

# Part 3: Publish-Subscribe (Pub/Sub)

## What is Pub/Sub?

A messaging pattern where a **publisher sends a message to a topic, and all subscribers to that topic receive a copy**.

> Key difference from queues: **one message → all subscribers** (fanout), not one consumer.

```
Publisher ──► [Topic] ──► Subscriber A (gets a copy)
                     └──► Subscriber B (gets a copy)
                     └──► Subscriber C (gets a copy)
```

---

## Message Queue vs Pub/Sub

| Aspect | Message Queue | Pub/Sub |
|---|---|---|
| **Delivery** | One message → **one** consumer | One message → **all** subscribers |
| **Coupling** | Producer may know consumer | Publisher doesn't know subscribers |
| **Purpose** | Task distribution, work queues | Event notification, fanout |
| **Consumption** | Consuming removes the message | Each subscriber gets their own copy |

---

## Core Concepts

| Concept | Description |
|---|---|
| **Publisher** | Sends events to a topic; doesn't care who listens |
| **Topic** | Named channel (e.g. `orders`, `user-events`); receives messages and fans out |
| **Subscriber** | Registers interest in a topic; receives all messages on it |
| **Subscription** | Link between a subscriber and a topic; can have filters |
| **Message** | Payload + metadata (ID, timestamp, headers, attributes) |

---

## How Pub/Sub Works

### Push vs Pull Delivery

| | Push | Pull |
|---|---|---|
| **Who initiates** | Broker pushes to subscriber endpoint | Subscriber polls broker |
| **Latency** | Lower — immediate delivery | Higher — depends on poll interval |
| **Backpressure** | Harder — broker controls rate | Natural — subscriber pulls when ready |
| **Use case** | Real-time, webhooks | Batch processing, rate control |

---

## Message Fanout Patterns

### Simple Fanout
Every subscriber gets every message — used for broadcasting events.

### Filtered Fanout
Subscribers only receive messages matching their filter criteria:
```
Topic: order-events
  ├── Inventory Service  → filter: event_type = "OrderPlaced"
  ├── Notification Svc   → filter: event_type = "OrderShipped"
  └── Analytics Service  → filter: all events
```

### Fan-out to Queues *(Production Best Practice)*
Combine pub/sub fanout with per-consumer queues for reliability:
```
Topic ──► Sub A's Queue ──► Workers A (multiple instances)
      └──► Sub B's Queue ──► Workers B (multiple instances)
      └──► Sub C's Queue ──► Workers C (multiple instances)
```
Each service gets its own reliable queue. Pub/sub handles fanout; queues handle retry + backpressure + scaling.

---

## Subscriber Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Exclusive** | One subscriber per subscription | Stateful services |
| **Shared** | Multiple instances share a subscription (load balanced) | Scalable workers |
| **Durable** | Messages stored when subscriber is offline; delivered on reconnect | Critical processing |
| **Ephemeral** | Only delivered to currently connected subscribers; missed messages lost | Real-time feeds |

---

## Popular Pub/Sub Implementations

| System | Persistence | Replay | Ordering | Managed | Best For |
|---|---|---|---|---|---|
| **Apache Kafka** | ✅ Yes | ✅ Yes | Per-partition | Self/Managed | High-throughput event streaming |
| **AWS SNS** | ❌ No | ❌ No | ❌ No | ✅ Yes | Fan-out to SQS/Lambda/HTTP/email |
| **Google Pub/Sub** | ✅ Yes (7 days) | ✅ Yes (Seek) | Per-key | ✅ Yes | Global apps, GCP workloads |
| **Redis Pub/Sub** | ❌ No | ❌ No | ❌ No | Self | Real-time, cache invalidation |

---

## Design Considerations

### 1. At-Least-Once Delivery
Most pub/sub systems may deliver a message **more than once**. Subscribers **must be idempotent**.

```python
# Track processed message IDs to deduplicate
if message_id in processed_ids:
    return  # already handled — skip
processed_ids.add(message_id)
process(message)
```

### 2. Message Ordering
Global ordering is rarely guaranteed. Use **partition keys** (Kafka) or **ordering keys** (GCP Pub/Sub) for per-key ordering. Always include `eventTime` + `sequenceNumber` in messages to handle out-of-order delivery.

### 3. Subscriber Failure + DLQ
Use bounded retries with exponential backoff. After max retries, move to **Dead Letter Queue**:
```
Process message
  │ fails
  ▼
Retry (1s, 2s, 4s, 8s...)
  │ still failing after N retries
  ▼
Dead Letter Queue → alert + manual inspection
```

### 4. Schema Evolution
- ✅ **Safe:** Add new optional fields
- ❌ **Breaking:** Remove fields, rename fields, change types
- Use a **Schema Registry** (Confluent, AWS Glue) to enforce compatibility
- Version your events; prefer additive changes

---

## Common Pub/Sub Patterns

| Pattern | Description |
|---|---|
| **Event Notification** | Publish event → subscribers decide what to do independently |
| **Fan-out to Queues** | Topic fans out → each service has own queue for reliable processing |
| **Event Sourcing** | Append-only event log; rebuild state by replaying events |
| **CQRS** | Separate write model from read model; events are the bridge between them |

---

## Quick Decision Guide

```
Need immediate response / user is waiting?
└── Synchronous (REST, gRPC)

Operation can be deferred / background task?
├── One receiver processes each message?   → Message Queue (SQS, RabbitMQ)
└── Multiple receivers need each message?  → Pub/Sub (Kafka, SNS, GCP Pub/Sub)

Real-time, ephemeral, in-memory only?
└── Redis Pub/Sub

High-throughput streaming + replay needed?
└── Apache Kafka
```

---

*Sources:*
- *[algomaster.io – Sync vs Async Communication](https://algomaster.io/learn/system-design/sync-vs-async-communication)*
- *[algomaster.io – Message Queues](https://algomaster.io/learn/system-design/message-queues)*
- *[algomaster.io – Pub/Sub](https://algomaster.io/learn/system-design/pub-sub)*