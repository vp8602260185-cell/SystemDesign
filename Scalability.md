# 📐 Scalability — System Design Revision Notes

---

## 1. What is Scalability?

**Scalability** is a system's ability to handle **growing amounts of load** (users, data, requests) gracefully without degrading performance.

> A system is scalable if adding more resources results in a proportional increase in performance/capacity.

---

## 2. Measuring Scalability

| Metric | Description |
|---|---|
| **Throughput** | Requests handled per second (RPS / QPS) |
| **Latency** | Time to respond to a single request (p50, p95, p99) |
| **Response Time** | End-to-end time experienced by user |
| **Error Rate** | % of requests that fail under load |

> **Key principle:** Scalability = maintaining acceptable latency and throughput as load increases.

---

## 3. Vertical Scaling (Scale Up) ⬆️

**Add more power to existing machines** — bigger CPU, more RAM, faster disk.

### ✅ Pros
- Simple — no code changes needed
- No distributed system complexity
- Low latency (single machine, no network hops)

### ❌ Cons
- **Hard limit** — there's a maximum hardware cap
- **Single point of failure (SPOF)** — if the machine dies, everything goes down
- Expensive at high-end specs
- Downtime needed to upgrade

### When to use
Early stage systems, databases (easier than sharding), quick wins.

---

## 4. Horizontal Scaling (Scale Out) ➡️

**Add more machines** and distribute the load across them.

### ✅ Pros
- **Theoretically unlimited** scalability
- High availability — no single point of failure
- Commodity hardware (cheap)
- Can scale independently per component

### ❌ Cons
- Increased complexity (distributed systems)
- Need **load balancers**
- Stateful services are harder to scale (sessions, caches)
- Network latency between nodes

### When to use
Large-scale systems, stateless services, web servers, API layers.

---

## 5. Vertical vs Horizontal — Quick Comparison

| | Vertical | Horizontal |
|---|---|---|
| Cost | High at scale | Lower (commodity) |
| Complexity | Low | High |
| Limit | Hardware ceiling | Virtually none |
| Failure risk | High (SPOF) | Low (redundant) |
| Downtime | Yes (for upgrade) | No |
| Example | Bigger RDS instance | Adding more EC2s |

---

## 6. Scaling Different Components

### 🌐 Web / Application Servers
- **Stateless** → easy to scale horizontally
- Use a **Load Balancer** (Round Robin, Least Connections, IP Hash)
- Auto-scaling groups (AWS ASG, GCP MIG)

### 🗄️ Databases
Databases are the hardest to scale. Strategies:

| Strategy | Description |
|---|---|
| **Read Replicas** | Replicate DB; reads go to replicas, writes to primary |
| **Connection Pooling** | Reuse DB connections (PgBouncer, HikariCP) |
| **Caching** | Cache frequent reads (Redis, Memcached) |
| **Sharding** | Split data across multiple DBs by a shard key |
| **Vertical Scaling** | Upgrade DB instance size first (simple) |

### ⚡ Caching
- **Client-side cache** — browser cache
- **CDN** — cache static assets at edge (Cloudflare, CloudFront)
- **Application cache** — Redis / Memcached for DB query results
- **DB cache** — Query cache, buffer pool

> Cache invalidation is hard — use **TTL**, **write-through**, or **cache-aside** patterns.

### 📨 Message Queues (Async Scaling)
- Decouple producers from consumers
- Handle traffic spikes by buffering work
- Examples: **Kafka**, **RabbitMQ**, **SQS**
- Use for: sending emails, processing images, analytics pipelines

### 🔍 Search
- Offload search to dedicated engine: **Elasticsearch**, **Solr**
- Don't use SQL `LIKE '%query%'` queries at scale

### 💾 Storage / File Serving
- Use **Object Storage** (S3, GCS) instead of local disk
- Serve files via **CDN** to reduce origin load

---

## 7. Load Balancing

Distributes incoming traffic across multiple servers.

### Algorithms
| Algorithm | How it works | Best for |
|---|---|---|
| Round Robin | Requests cycle through servers | Stateless, equal servers |
| Weighted RR | Servers get traffic proportional to weight | Servers of different capacities |
| Least Connections | Send to server with fewest active connections | Long-lived connections |
| IP Hash | Same client always hits same server | Sticky sessions |
| Random | Pick a random server | Simple use cases |

### Layer 4 vs Layer 7
- **L4 (Transport)** — routes based on IP/TCP without reading content. Fast.
- **L7 (Application)** — routes based on URL, headers, cookies. More flexible (e.g., route `/api` to API servers, `/images` to image servers).

---

## 8. Stateless vs Stateful Services

| | Stateless | Stateful |
|---|---|---|
| Definition | No session stored on server | Server holds client state |
| Scalability | Easy (any server handles any request) | Hard (client must hit same server) |
| Solution for stateful | Move state to external store (Redis, DB) | Sticky sessions or distributed sessions |

> **Golden rule:** Make your app tier stateless. Store state in a shared external layer (Redis, DB).

---

## 9. Database Sharding

Splitting a large database into smaller **shards**, each shard holds a subset of data.

### Shard Key Strategies
| Strategy | Example | Pros | Cons |
|---|---|---|---|
| **Range-based** | User IDs 1–1M on Shard A | Simple | Hot spots |
| **Hash-based** | `hash(user_id) % N` | Even distribution | Hard to range query |
| **Directory-based** | Lookup table maps key → shard | Flexible | Lookup overhead |

### Challenges
- **Joins across shards** are expensive
- **Rebalancing** shards is complex
- Transactions across shards (distributed transactions)

---

## 10. CAP Theorem (Related Concept)

In a distributed system, you can only guarantee **2 of 3**:

| Property | Meaning |
|---|---|
| **C**onsistency | Every read sees the latest write |
| **A**vailability | Every request gets a response |
| **P**artition Tolerance | System works despite network partitions |

> Network partitions are unavoidable → choose **CP** (banks, strong consistency) or **AP** (social media, high availability).

---

## 11. Example: Scaling from 0 → Millions of Users

```
Stage 1: Single Server
├── Web server + DB on same machine
└── Works for early prototypes

Stage 2: Separate DB Server
├── Web server and DB on separate machines
└── Can scale each independently

Stage 3: Add Load Balancer + Multiple Web Servers
├── LB distributes traffic
└── Web servers are stateless

Stage 4: Add Caching Layer (Redis)
├── Cache DB query results
└── Reduces DB load dramatically

Stage 5: Add CDN
├── Static assets served from edge
└── Reduces origin server load

Stage 6: Read Replicas for DB
├── Read traffic → replicas
└── Write traffic → primary

Stage 7: Database Sharding
├── Data split across multiple DBs
└── Handles massive data volumes

Stage 8: Microservices + Message Queues
├── Services scale independently
└── Async processing with Kafka/SQS
```

---

## 12. Key Numbers to Remember

| Item | Approximate Value |
|---|---|
| L1 cache reference | 0.5 ns |
| RAM reference | 100 ns |
| SSD read | 100 µs |
| HDD seek | 10 ms |
| Round trip in same datacenter | 0.5 ms |
| Redis GET | ~0.1 ms |
| DB query (indexed) | ~1 ms |
| DB query (full table scan) | 100 ms – seconds |

---

## 13. Quick Summary

| Concept | Key Takeaway |
|---|---|
| Vertical Scaling | Easy but has limits + SPOF |
| Horizontal Scaling | Complex but unlimited + resilient |
| Load Balancer | Distributes traffic; enables horizontal scaling |
| Stateless Servers | Critical for horizontal scaling |
| Caching | Biggest bang for buck — reduces DB load |
| Read Replicas | Easy DB read scaling |
| Sharding | Last resort for DB write scaling |
| CDN | Offload static content delivery |
| Message Queues | Decouple and handle spikes asynchronously |

---

> 💡 **Interview tip:** Always start with the simplest solution (vertical scale, single server), then evolve to horizontal scaling, adding caching, load balancing, and sharding only as needed. Justify each step with load estimations.