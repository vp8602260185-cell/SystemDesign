# 🟢 Availability — System Design Revision Notes

---

## 1. What is Availability?

**Availability** is the percentage of time a system is **operational and accessible** when needed.

> Availability = (Uptime) / (Uptime + Downtime) × 100%

A highly available system continues to function even when parts of it fail — through **redundancy**, **failover**, and **fault tolerance**.

---

## 2. Measuring Availability — The "Nines"

| Availability | Downtime / Year | Downtime / Month | Downtime / Week |
|---|---|---|---|
| 90% (1 nine) | 36.5 days | 73 hours | 16.8 hours |
| 99% (2 nines) | 3.65 days | 7.3 hours | 1.68 hours |
| 99.9% (3 nines) | 8.76 hours | 43.8 min | 10.1 min |
| 99.99% (4 nines) | 52.6 min | 4.38 min | 1.01 min |
| 99.999% (5 nines) | 5.26 min | 26.3 sec | 6.05 sec |

> 💡 Most production systems target **99.9%** (3 nines). Mission-critical systems (banking, healthcare) target **99.99%** or higher.

### SLA vs SLO vs SLI

| Term | Meaning | Example |
|---|---|---|
| **SLI** (Service Level Indicator) | The actual metric measured | Request success rate = 99.95% |
| **SLO** (Service Level Objective) | Internal target for the SLI | Success rate must be ≥ 99.9% |
| **SLA** (Service Level Agreement) | External contract with users | "We guarantee 99.9% uptime or credits" |

---

## 3. Common Failure Modes

Understanding *why* systems fail is the first step to preventing it.

| Failure Mode | Description | Example |
|---|---|---|
| **Hardware failure** | Disk crash, NIC failure, power loss | Disk dies on DB server |
| **Software bugs** | Memory leaks, unhandled exceptions | OOM crash under load |
| **Network partitions** | Nodes can't communicate | Switch failure between datacenters |
| **Overload / traffic spikes** | System overwhelmed beyond capacity | Flash sale or viral traffic |
| **Dependency failures** | Third-party service goes down | Payment gateway timeout |
| **Human error** | Misconfiguration, bad deployment | Wrong config pushed to prod |
| **Natural disasters** | Fire, flood, earthquake | Entire datacenter goes offline |
| **Cascading failures** | One failure triggers another | Slow DB → timeouts → thread pool exhaustion → full outage |

---

## 4. Redundancy — The Foundation of Availability

**Redundancy** = having backup components so that if one fails, another takes over.

### Types of Redundancy

#### 4.1 Active-Active (Hot-Hot)
- **All nodes are live** and serving traffic simultaneously
- Load is distributed across all nodes
- If one fails, remaining nodes absorb its traffic
- **Higher resource utilization**, **no failover delay**

```
        ┌──────────────┐
        │  Load Balancer│
        └──────┬───────┘
       ┌───────┴───────┐
       ▼               ▼
  [Server A]       [Server B]
  (Active)         (Active)
  serving traffic  serving traffic
```

> Best for: **stateless services** — web servers, API layers.

#### 4.2 Active-Passive (Hot-Standby)
- **One node is active**, others are on standby
- Standby monitors primary and takes over on failure (failover)
- **Wasted standby capacity** but simpler consistency
- Small **failover delay** (seconds to minutes)

```
        ┌──────────────┐
        │  Load Balancer│
        └──────┬───────┘
               ▼
          [Server A]  ←──heartbeat──→  [Server B]
          (Active)                      (Passive/Standby)
```

> Best for: **stateful services** — primary databases, leader election.

#### 4.3 Cold Standby
- Backup is **off** until needed
- Must be manually or automatically brought online
- **Cheapest** but **slowest** recovery (RTO is high)

---

## 5. Key Availability Metrics

| Metric | Definition |
|---|---|
| **MTBF** (Mean Time Between Failures) | Average time between two consecutive failures |
| **MTTR** (Mean Time To Recovery) | Average time to restore the system after failure |
| **MTTF** (Mean Time To Failure) | Average time until a (non-repairable) component fails |
| **RTO** (Recovery Time Objective) | Max acceptable downtime after a failure |
| **RPO** (Recovery Point Objective) | Max acceptable data loss (in time) |

> **Availability ≈ MTBF / (MTBF + MTTR)**
> To improve availability: **increase MTBF** (more resilient system) or **decrease MTTR** (faster recovery).

### RTO vs RPO Visual

```
                         Failure
                            │
  ─────────────────────────►│◄──── RTO ────►│ System restored
                            │               │
  Last backup ◄──── RPO ───►│               │
```

- **Low RPO** → more frequent backups / synchronous replication
- **Low RTO** → automated failover, hot standby, runbooks

---

## 6. High Availability Patterns

### 6.1 Load Balancing
Distribute traffic across multiple servers so no single server is a bottleneck or SPOF.

- **Health checks** — LB removes unhealthy nodes automatically
- **Multiple LBs** — avoid the LB itself becoming a SPOF (use DNS round-robin or anycast)
- Tools: **Nginx**, **HAProxy**, **AWS ALB/NLB**, **Cloudflare**

### 6.2 Database High Availability

#### Primary-Replica (Master-Slave)
```
Writes ──► [Primary DB] ──replication──► [Replica 1]
                                    └──► [Replica 2]
Reads ──────────────────────────────────► Replicas
```
- If primary fails → promote a replica to primary
- **Semi-synchronous** or **async** replication

#### Multi-Primary (Multi-Master)
- Multiple nodes accept writes
- Conflict resolution needed
- Example: **CockroachDB**, **Cassandra**, **Galera Cluster**

#### Automatic Failover Tools
- **AWS RDS Multi-AZ** — automatic standby promotion
- **Patroni** (PostgreSQL), **MHA** (MySQL)
- **Redis Sentinel** / **Redis Cluster**

### 6.3 Geographic Redundancy (Multi-Region)

Deploy across multiple **Availability Zones (AZs)** and **Regions** to survive datacenter-level failures.

| Level | Protects Against | Latency Impact |
|---|---|---|
| Multiple servers (same rack) | Single server failure | None |
| Multiple AZs (same region) | Datacenter failure | Low (~1–2 ms) |
| Multiple Regions | Regional disaster | Higher (~50–150 ms) |

> **Active-Active multi-region** = highest availability, but requires **data synchronization** across regions.

### 6.4 Health Checks & Auto-Healing
- **Liveness probe** — Is the process alive?
- **Readiness probe** — Is the service ready to receive traffic?
- Orchestrators (Kubernetes) automatically restart failed containers
- Auto Scaling Groups replace failed EC2 instances

### 6.5 Circuit Breaker Pattern
Prevents cascading failures when a dependency is slow or down.

```
States:
  CLOSED ──(failures exceed threshold)──► OPEN ──(timeout)──► HALF-OPEN
    ▲                                                               │
    └──────────────(success)────────────────────────────────────────┘

CLOSED:    Normal — requests pass through
OPEN:      Dependency is failing — fail fast, don't call it
HALF-OPEN: Test with a few requests — if OK, move back to CLOSED
```

> Libraries: **Resilience4j** (Java), **Hystrix** (Netflix), **Polly** (.NET)

### 6.6 Retry with Exponential Backoff + Jitter
When a request fails, retry — but don't hammer the failing service.

```
Retry 1: wait 1s
Retry 2: wait 2s
Retry 3: wait 4s
Retry 4: wait 8s  (+/- jitter to avoid thundering herd)
```

> Always set a **max retry limit** and combine with circuit breakers.

### 6.7 Timeouts
Every external call (DB, API, cache) must have a timeout. Without timeouts:
- Threads block indefinitely
- Thread pool exhaustion → entire service unresponsive

> Rule of thumb: set timeouts at **p99 latency + buffer** of the dependency.

### 6.8 Bulkhead Pattern
Isolate failures so one component's failure doesn't drain resources from others.

- Separate **thread pools** per downstream service
- Separate **connection pools** per DB
- Like watertight compartments on a ship 🚢

### 6.9 Graceful Degradation
When a non-critical component fails, return a **degraded but functional response** instead of a full error.

| Scenario | Graceful Degradation |
|---|---|
| Recommendation service is down | Show generic popular items |
| Search is slow | Return cached results |
| Payment gateway times out | Show "try again" instead of crashing |

### 6.10 Rate Limiting
Protect your system from overload that causes unavailability.

- **Token Bucket** — smooth bursts, allows short spikes
- **Leaky Bucket** — strict rate, queues excess requests
- **Fixed Window** — simple but allows boundary spikes
- **Sliding Window** — more accurate, no boundary issues

---

## 7. Availability in Series vs Parallel

### Services in Series (Sequential)
If components are chained, **overall availability DROPS**:

```
Availability = A₁ × A₂ × A₃ ...

Example: 3 services each at 99.9%
= 0.999 × 0.999 × 0.999 = 99.7%
```

### Services in Parallel (Redundant)
Redundancy **increases** overall availability:

```
Availability = 1 − (1 − A)ⁿ   (n = number of redundant nodes)

Example: 2 servers each at 99%
= 1 − (1 − 0.99)² = 1 − 0.0001 = 99.99%
```

> 💡 This is why **redundancy** is so powerful — two 99% servers give you 99.99% combined.

---

## 8. Failover Strategies

| Strategy | Description | RTO |
|---|---|---|
| **DNS Failover** | DNS record points to backup IP | Minutes (TTL-based) |
| **Load Balancer Failover** | LB health checks reroute traffic | Seconds |
| **Database Promotion** | Replica promoted to primary | Seconds to minutes |
| **Cross-region failover** | Traffic routed to backup region | Minutes |

---

## 9. Data Redundancy & Backups

| Approach | Description |
|---|---|
| **Synchronous Replication** | Write confirmed only after replica also writes. Zero data loss, higher latency. |
| **Asynchronous Replication** | Write confirmed at primary, replica catches up. Lower latency, small data loss risk. |
| **Snapshots** | Periodic full/incremental backups. Good RPO if frequent. |
| **WAL Shipping** | PostgreSQL/MySQL ship transaction logs to replica continuously. |
| **Multi-region replication** | Data replicated across geographic regions (e.g., DynamoDB Global Tables, S3 CRR). |

---

## 10. Avoiding Single Points of Failure (SPOF)

Ask for every component: **"What happens if this fails?"**

| Component | SPOF Risk | Solution |
|---|---|---|
| Single web server | Yes | Add more + LB |
| Single database | Yes | Primary-replica + auto failover |
| Single load balancer | Yes | Multiple LBs / anycast DNS |
| Single datacenter | Yes | Multi-AZ / Multi-region |
| Single DNS provider | Yes | Secondary DNS |
| Single CDN | Yes | Multi-CDN strategy |

---

## 11. Chaos Engineering
Intentionally inject failures to test availability assumptions.

- **Netflix Chaos Monkey** — randomly terminates instances in production
- **Gremlin** — controlled chaos experiments
- Validates that redundancy, failover, and circuit breakers actually work

> "Hope is not a strategy. Test your failover."

---

## 12. Quick Summary

| Concept | Key Takeaway |
|---|---|
| **Availability %** | Measured in "nines" — 99.9% = ~8.7h downtime/year |
| **MTBF / MTTR** | Improve availability by increasing MTBF or reducing MTTR |
| **RTO / RPO** | Define acceptable downtime and data loss for DR planning |
| **Active-Active** | All nodes serve traffic — highest availability |
| **Active-Passive** | Standby takes over on failure — simpler but has delay |
| **Redundancy** | Parallel components dramatically increase availability |
| **Circuit Breaker** | Fail fast — prevent cascading failures |
| **Bulkhead** | Isolate failure blast radius |
| **Graceful Degradation** | Partial service > no service |
| **Multi-AZ / Multi-Region** | Survive datacenter and regional failures |
| **Health Checks** | Automatically route away from failed nodes |

---

## 13. Availability vs Related Concepts

| Concept | Focus |
|---|---|
| **Availability** | System is up and reachable |
| **Reliability** | System performs correctly when up |
| **Fault Tolerance** | System continues despite failures (overlaps with availability) |
| **Durability** | Data is not lost (storage-specific) |
| **Resilience** | System recovers quickly from failures |

---

> 💡 **Interview tip:** When asked about availability, mention redundancy, failover, health checks, circuit breakers, and multi-region. Always tie each pattern back to reducing MTTR or eliminating SPOFs. Use the "nines" table to anchor your SLA discussions.