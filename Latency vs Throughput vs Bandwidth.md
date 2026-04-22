# ⚡ Latency vs Throughput vs Bandwidth — Revision Notes

---

## 1. The Highway Analogy (Quick Mental Model)

| Concept | Highway Analogy |
|---|---|
| **Bandwidth** | Number of lanes on the highway (capacity) |
| **Throughput** | Actual cars passing per hour (real usage) |
| **Latency** | Time for one car to travel city A → city B |

> A 4-lane highway (high bandwidth) with an accident might only pass 100 cars/hour (low throughput), while each car takes 2 hours (high latency). These three metrics are **independent**.

---

## 2. Latency

**Latency** = time for a single request to travel from source to destination and back (delay).

> Also called **Round-Trip Time (RTT)** in networking.

### Components of Latency

| Component | Description |
|---|---|
| **Propagation delay** | Time for signal to travel the medium. Light in fiber ≈ 200,000 km/s. Cross-Atlantic (~6000 km) = ~30ms |
| **Transmission delay** | Time to push bits onto the wire (depends on packet size + link speed) |
| **Processing delay** | Time for routers, LBs, servers to process packets |
| **Queuing delay** | Wait time in buffers when components are busy |

### Measuring Latency — Use Percentiles, Not Averages!

| Metric | Meaning |
|---|---|
| **p50** | 50% of requests are faster than this (median) |
| **p95** | 95% of requests are faster than this |
| **p99** | 99% of requests are faster than this |
| **p99.9** | 99.9% of requests are faster than this |

> ⚠️ **Average hides outliers!** A system with 10ms average can have p99 = 500ms — meaning 1% of users (= 10,000 users/day at 1M req/day) have a terrible experience.

### What Affects Latency?

| Factor | Impact |
|---|---|
| Geographic distance | More distance = more propagation delay |
| Network congestion | Causes queuing delays |
| Server load | Increases processing time |
| Slow DB queries | Directly adds to response time |
| DNS resolution | Cold requests need a lookup |
| TLS handshake | Adds 1–2 extra round trips |

### How to Reduce Latency
- **CDN** — serve content from edge nodes closer to users
- **Caching** — eliminate round trips at multiple layers
- **Connection pooling** — avoid repeated connection setup overhead
- **DB optimization** — indexes, query tuning
- **Geographic distribution** — deploy closer to users
- **HTTP/2 or HTTP/3 (QUIC)** — protocol-level optimizations

---

## 3. Throughput

**Throughput** = amount of work completed per unit of time (volume).

> Expressed as **RPS** (requests/sec), **TPS** (transactions/sec), or **Mbps** for data.

### Throughput vs Bandwidth

| Metric | Definition | Example |
|---|---|---|
| **Bandwidth** | Maximum *possible* transfer rate (theoretical) | 1 Gbps network link |
| **Throughput** | Actual *achieved* transfer rate | 600 Mbps real transfer |

> Throughput ≤ Bandwidth always. The gap exists due to: protocol overhead, congestion, packet loss, processing limits.

### What Limits Throughput?

The **bottleneck** determines max throughput — system is only as fast as its slowest component.

| Bottleneck | Cause |
|---|---|
| **CPU** | Compute-heavy tasks |
| **Memory** | Large working sets |
| **I/O** | Disk reads/writes or network |
| **Connections** | DB/network connection limits |
| **Contention** | Locks, shared resources |

### How to Improve Throughput
- **Horizontal scaling** — add more servers
- **Vertical scaling** — more CPU/RAM
- **Async processing** — don't block threads on slow ops
- **Batching** — process multiple items together
- **Caching** — reduce repeated work
- **Connection pooling** — reuse expensive connections
- **Load balancing** — distribute work evenly

---

## 4. Bandwidth

**Bandwidth** = maximum rate at which data *can* be transferred (capacity).

> Expressed in bits/sec — Kbps, Mbps, Gbps.

### Types of Bandwidth

| Type | Description | Example |
|---|---|---|
| **Network bandwidth** | Capacity of network links | 1 Gbps Ethernet |
| **Memory bandwidth** | Data transfer rate to/from RAM | DDR4 ≈ 25 GB/s |
| **Disk bandwidth** | Read/write speed of storage | SSD ≈ 500 MB/s |
| **Bus bandwidth** | Internal data transfer (PCIe) | PCIe 4.0 ≈ 64 GB/s |

### Bandwidth-Delay Product (BDP)

```
BDP = Bandwidth × Latency (RTT)
```

Represents how much data can be **"in flight"** at any moment.

**Example:**
- Bandwidth = 1 Gbps = 125 MB/s
- Latency = 100ms (coast-to-coast US)
- BDP = 125 MB/s × 0.1s = **12.5 MB**

> If TCP window size < BDP, the sender must wait for ACKs before sending more — leaving bandwidth underutilized. Fix: TCP window scaling, QUIC, or application pipelining.

---

## 5. Quick Comparison Table

| | **Latency** | **Throughput** | **Bandwidth** |
|---|---|---|---|
| **Measures** | Delay per request | Volume of work | Maximum capacity |
| **Unit** | ms, µs, ns | RPS, TPS, Mbps | Mbps, Gbps |
| **Analogy** | Travel time per car | Cars passing per hour | Number of lanes |
| **Improved by** | CDN, caching, proximity | Scaling, async, batching | Better hardware/links |
| **Key metric** | p99 latency | RPS / TPS | Gbps |
| **Limited by** | Distance, queuing, processing | Bottleneck component | Physical medium |

---

## 6. Key Relationships

```
Throughput ≤ Bandwidth          (can never exceed capacity)

Latency ↑  →  Throughput ↓     (higher latency = fewer completions/sec under fixed concurrency)

Little's Law:  L = λ × W
  L = avg requests in system
  λ = arrival rate (throughput)
  W = avg time in system (latency)
```

> If latency doubles, throughput halves for the same concurrency level.

---

## 7. Real-World Reference Numbers

| Operation | Approximate Latency |
|---|---|
| L1 cache hit | ~0.5 ns |
| L2 cache hit | ~7 ns |
| RAM access | ~100 ns |
| SSD read | ~100 µs |
| HDD seek | ~10 ms |
| Same datacenter round trip | ~0.5 ms |
| Cross-region (US East ↔ West) | ~60 ms |
| Cross-Atlantic | ~150 ms |

---

## 8. Summary

| Concept | One-Line Takeaway |
|---|---|
| **Latency** | How *fast* — time for one request; always measure with percentiles |
| **Throughput** | How *much* — requests per second; limited by the bottleneck |
| **Bandwidth** | How *wide* — theoretical max capacity; throughput always falls below it |
| **BDP** | Links bandwidth + latency; how much data can be in-flight simultaneously |
| **p99 > average** | Always use percentiles to understand real user experience |

---

---

# 🎯 Most Asked Interview Questions

### Conceptual

**Q1. What is the difference between latency and throughput?**
> Latency measures the *delay* for a single request (how fast). Throughput measures the *volume* of requests completed per second (how much). A system can have low latency but low throughput (handles requests quickly, but one at a time) or high throughput but high latency (processes many requests, but each takes longer).

---

**Q2. What is the difference between bandwidth and throughput?**
> Bandwidth is the *theoretical maximum* capacity of a link. Throughput is the *actual achieved* rate. Throughput is always ≤ bandwidth due to overhead, congestion, packet loss, and processing limits.

---

**Q3. Why use p99 latency instead of average latency?**
> Average hides outliers. If 99 requests take 10ms and 1 takes 500ms, the average looks fine (~15ms) but 1% of users experience 500ms. At scale (1M req/day), that is 10,000 bad experiences. p99 exposes the tail latency that actually hurts real users.

---

**Q4. What is Little's Law and how does it relate to throughput?**
> `L = λW` — requests in system = arrival rate × avg time in system. If latency (W) doubles under load, throughput (λ) halves for the same concurrency. Reducing latency directly increases throughput capacity.

---

**Q5. How do latency and throughput trade off against each other?**
> They often conflict. Batching improves throughput but increases latency (items wait to form a batch). Replication improves availability but adds write latency. Caching reduces latency but adds complexity. Good design explicitly decides which to prioritize.

---

### Scenario-Based

**Q6. Your API has good average latency (10ms) but users are complaining. What do you investigate?**
> Check **tail latency** (p95, p99, p99.9). Look for: slow DB queries (missing index, lock contention), GC pauses, thread pool saturation, or a slow downstream dependency. Tools: distributed tracing (Jaeger), APM (Datadog, New Relic).

---

**Q7. How would you improve throughput of a CPU-bound system?**
> Horizontal scaling (more servers), optimize algorithms, use async/non-blocking I/O to free threads, introduce caching to reduce compute, or move heavy tasks to a background queue (Kafka, SQS).

---

**Q8. You have a 1 Gbps link between two DCs 5000 km apart. Why is actual throughput much lower?**
> Due to **Bandwidth-Delay Product**. With ~25ms RTT, BDP = 125 MB/s × 0.025s = ~3 MB. If TCP window size < BDP, the sender waits for ACKs before sending more, leaving the pipe underutilized. Fix: TCP window scaling, QUIC protocol, or application-level pipelining.

---

**Q9. How does caching improve both latency and throughput simultaneously?**
> Cache hits are served from memory (microseconds) instead of DB (milliseconds) — directly reducing latency. Since each request completes faster, the server handles more requests per second — improving throughput. It also reduces load on the bottleneck (DB), raising the system's overall capacity.

---

**Q10. Design a system needing <10ms p99 latency AND 100K RPS throughput. What are your strategies?**
> - **Stateless app servers** behind a load balancer → horizontal scale for throughput
> - **In-memory cache (Redis)** → serve hot data fast, reduce DB pressure
> - **Async / non-blocking I/O** (Netty, FastAPI, Node.js) → avoid thread starvation
> - **Connection pooling** to DB and downstream services
> - **CDN** for static assets → push latency to the edge
> - **Read replicas** → scale DB reads independently
> - **Monitor p99** with Prometheus + Grafana, set SLOs and alert on breaches