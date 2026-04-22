# 📐 CAP Theorem — Revision Notes

---

## 1. What is the CAP Theorem?

Introduced by **Eric Brewer in 2000**, the CAP theorem states:

> **A distributed data store can only guarantee 2 out of 3 properties: Consistency, Availability, and Partition Tolerance — never all three simultaneously.**

```
              Consistency (C)
                   /\
                  /  \
                 / CP \
                /------\
               /  (CA)  \
    Availability(A)----Partition Tolerance(P)
                   AP
```

---

## 2. The Three Pillars

### C — Consistency
> Every read receives the **most recent write** or an error. All nodes return the same data at the same time.

- Write to Node A → read from Node B → must see that write immediately
- If the system can't guarantee this, it returns an **error** rather than stale data
- Example: Bank balance — must always reflect latest transaction

### A — Availability
> Every request receives a **non-error response** — but it may **not** contain the most recent write.

- System is always operational and responsive
- Nodes may return **stale data** but they never refuse a request
- Example: Amazon shopping cart — always accepts items, never fails

### P — Partition Tolerance
> System continues to operate despite **network partitions** (nodes unable to communicate).

- A **network partition** = network failure splits nodes into isolated groups that can't talk to each other
- Partitions **always happen** in real distributed systems (network cables fail, switches crash, etc.)
- Therefore: **P is not optional** — every real distributed system must tolerate partitions

> ⚠️ Since P is unavoidable, the real choice is always between **C and A** during a partition.

---

## 3. The CAP Trade-Off

When a network partition occurs, a system must choose:

```
Network Partition Occurs
         │
         ▼
┌────────────────────────┐
│  Return consistent     │  ← Choose C (reject/error some requests)
│  data or error?        │
├────────────────────────┤
│  Return available      │  ← Choose A (serve possibly stale data)
│  (possibly stale) data?│
└────────────────────────┘
```

---

## 4. CP vs AP vs CA

### CP — Consistency + Partition Tolerance
- **Sacrifice:** Availability
- During a partition → system **rejects requests** rather than serve stale data
- Returns error or timeout until partition heals

| | |
|---|---|
| **Behaviour** | Refuses requests to stay consistent |
| **Examples** | MySQL/PostgreSQL (strong consistency config), HBase, Zookeeper, etcd |
| **Use case** | Banking, financial transactions, inventory management |

> **ATM example:** When you withdraw cash, all nodes must agree on your balance. If they can't sync, the ATM rejects the transaction rather than risk overdraft.

---

### AP — Availability + Partition Tolerance
- **Sacrifice:** Consistency (temporarily)
- During a partition → system **serves stale data** rather than reject requests
- Nodes may return different values for the same key
- Eventually converges to consistency when partition heals (**Eventual Consistency**)

| | |
|---|---|
| **Behaviour** | Always responds, may return outdated data |
| **Examples** | Cassandra, DynamoDB, CouchDB, Riak, DNS, CDNs |
| **Use case** | Social media feeds, shopping carts, product recommendations |

> **Amazon cart example:** When you add an item during Black Friday traffic, it always succeeds — even if some nodes haven't synced yet. You might briefly see an outdated cart count, but it self-corrects.

---

### CA — Consistency + Availability
- **Sacrifice:** Partition Tolerance
- Only works on a **single node** or in environments where partitions never happen
- **Not practical** in real distributed systems (partitions are inevitable)

| | |
|---|---|
| **Behaviour** | Consistent + available, but breaks during any network failure |
| **Examples** | Single-node PostgreSQL, SQLite |
| **Use case** | Local/single-machine systems only |

> **Reality check:** You can't build a reliable distributed system without P. CA is mostly theoretical.

---

## 5. Quick Comparison Table

| Type | Consistency | Availability | Partition Tolerance | Real-World Systems |
|---|---|---|---|---|
| **CP** | ✅ | ❌ (during partition) | ✅ | HBase, Zookeeper, etcd, MongoDB (strong) |
| **AP** | ❌ (eventual) | ✅ | ✅ | Cassandra, DynamoDB, CouchDB, DNS |
| **CA** | ✅ | ✅ | ❌ | Single-node DB (not distributed) |

---

## 6. Practical Design Strategies

### 6.1 Eventual Consistency
Updates propagate to all nodes **eventually**, not immediately.

- Acceptable when small windows of stale data are OK
- Systems self-heal after partition resolves
- Examples: **DNS**, **CDNs**, **social media timelines**, **product recommendations**

```
Write to Node A
Node A: balance = $500  ✅
Node B: balance = $450  ⏳ (stale, will sync soon)
```

---

### 6.2 Strong Consistency
Once a write is confirmed, **any subsequent read returns that value** — no exceptions.

- Higher latency (must wait for all replicas to acknowledge)
- Examples: **financial ledgers**, **inventory systems**, **order processing**

```
Write to Node A: balance = $500
Node A confirms → broadcasts to all replicas
Only then → write acknowledged to client
All reads now return $500
```

---

### 6.3 Tunable Consistency
Adjust consistency level **per operation** based on needs.

- **Cassandra** allows per-query consistency: `ONE`, `QUORUM`, `ALL`
- High consistency for critical ops, eventual for non-critical

| Cassandra Level | Meaning |
|---|---|
| `ONE` | Read/write from 1 replica (fastest, lowest consistency) |
| `QUORUM` | Majority of replicas must respond (balanced) |
| `ALL` | All replicas must respond (strongest, slowest) |

> **E-commerce example:** Order checkout → `QUORUM` (strong). Product recommendations → `ONE` (eventual, fast).

---

### 6.4 Quorum-Based Approaches
Nodes **vote** to agree on a value. Requires `W + R > N` for consistency.

```
N = total replicas
W = nodes that must confirm a write
R = nodes that must respond to a read

For N=3: W=2, R=2 → W+R=4 > 3 ✅ (guaranteed to overlap)
```

Used in **Paxos**, **Raft**, **Cassandra**, **DynamoDB**

---

## 7. Beyond CAP — PACELC Theorem

CAP only describes behaviour **during a partition**. **PACELC** (Daniel Abadi) extends this:

```
If Partition (P):    choose between Availability (A) vs Consistency (C)
Else (E — normal):   choose between Latency (L) vs Consistency (C)
```

> Even without failures, there's a tradeoff: **replicating to all nodes synchronously = more consistent but higher latency.**

| System | Partition behaviour | Normal behaviour |
|---|---|---|
| DynamoDB | AP | EL (low latency) |
| Cassandra | AP | EL (low latency) |
| MongoDB | CP | EC (higher consistency) |
| MySQL (sync replication) | CP | EC |

---

## 8. Real-World System Mapping

| System | CAP Type | Why |
|---|---|---|
| **MySQL / PostgreSQL** | CP | Strong consistency, rejects during partition |
| **MongoDB** | CP (default) | Primary election during partition |
| **HBase** | CP | Built on Zookeeper (CP) |
| **Zookeeper / etcd** | CP | Consensus-based, consistent leader |
| **Cassandra** | AP | Always available, eventual consistency |
| **DynamoDB** | AP (default) | Highly available, tunable |
| **CouchDB** | AP | Optimistic replication, eventual |
| **DNS** | AP | Propagation delay accepted |
| **CDN** | AP | Edge caches may serve stale content |

---

## 9. Summary

| Concept | Key Takeaway |
|---|---|
| **C — Consistency** | Every read gets the latest write or an error |
| **A — Availability** | Every request gets a response (may be stale) |
| **P — Partition Tolerance** | System works despite network failures |
| **P is non-negotiable** | Real distributed systems always need P |
| **CP** | Prioritise correctness — banking, finance |
| **AP** | Prioritise uptime — social media, carts, DNS |
| **CA** | Not feasible in distributed systems |
| **Eventual consistency** | AP systems self-heal after partition |
| **Tunable consistency** | Cassandra/Dynamo allow per-op tradeoff |
| **PACELC** | Extends CAP: even without partition, latency vs consistency tradeoff exists |

---

---

# 🎯 Most Asked Interview Questions (Scenario-Based)

---

**Q1. You're designing a banking transaction system. Which CAP properties do you prioritise and why?**

> Choose **CP**. In a banking system, serving stale or inconsistent data (e.g., an outdated balance) can lead to double-spending or overdrafts — which is far worse than a brief unavailability. During a network partition, it is better to reject the transaction (return an error) than to allow both sides of the network to process it with different states.
>
> Implementation: Use **synchronous replication** with a consensus protocol (Raft/Paxos). Accept writes only after a quorum confirms. If quorum can't be reached, reject the write. Use systems like **PostgreSQL with sync replication**, **etcd**, or **Spanner**.

---

**Q2. You're building a social media platform (like Twitter's timeline). Which CAP properties do you choose?**

> Choose **AP**. It's acceptable for a user to briefly see a slightly stale timeline (a tweet that's 2 seconds delayed). Refusing to load the feed during a network hiccup would be a far worse user experience. The system should always respond with the best data it has.
>
> Use **Cassandra** or **DynamoDB** in AP mode with eventual consistency. Timelines self-correct within seconds when partitions heal. Trade-off: users in different regions may momentarily see different tweet counts — acceptable for this use case.

---

**Q3. Amazon's shopping cart must never fail to add items, even during peak load (Black Friday). How do you design this?**

> This is a classic **AP** problem (as described in Amazon's Dynamo paper). The cart service always accepts writes to a local node immediately — never blocking on cross-node consensus. If two nodes accept conflicting writes during a partition, the conflict is resolved later using **version vectors** (last-write-wins or merge strategies). The user might briefly see a slightly inconsistent cart, but it never refuses an "Add to Cart" action. DynamoDB was built specifically for this pattern.

---

**Q4. You are using Cassandra for an e-commerce app. Order placement needs strong consistency but product search can be eventual. How do you configure this?**

> Cassandra's **tunable consistency** handles this perfectly. For order placement (critical), use `QUORUM` or `ALL` consistency level — reads and writes require majority of replicas to respond, ensuring no stale data. For product search (non-critical), use `ONE` — reads from any single replica, fastest response, stale data acceptable. This avoids the overhead of strong consistency everywhere while maintaining correctness where it matters.

---

**Q5. A network partition splits your database cluster into two halves. Both halves keep accepting writes. What problem does this cause and how do you prevent it?**

> This is a **split-brain** scenario — both halves believe they are the primary and accept conflicting writes. When the partition heals, you have two divergent versions of the data that must be reconciled, potentially with data loss or corruption.
>
> Prevention (CP approach): Use a **leader election** protocol (Raft/Paxos). Only the partition with a **quorum (majority)** of nodes continues accepting writes. The minority partition rejects writes and becomes read-only until it reconnects. This ensures only one "truth" exists at all times. Tools: **Zookeeper**, **etcd**, **MongoDB replica sets** (primary election).

---

**Q6. Your system uses DynamoDB (AP). During a partition, a user updates their profile picture on one node. Another node still shows the old picture. How do you handle this in your application?**

> This is expected behaviour for an AP system with eventual consistency. Strategies:
> 1. **Accept it for non-critical data** — profile pictures syncing 1–2 seconds late is fine for UX
> 2. **Read-your-own-writes consistency** — route the same user's reads to the same node (using consistent hashing / sticky routing) right after a write so they always see their own updates
> 3. **Conditional writes** — use DynamoDB's `ConditionExpression` to detect conflicts before overwriting
> 4. **Version tracking** — store a `version` field; if a conflict is detected on sync, apply a merge strategy or keep the latest timestamp

---

**Q7. Explain the PACELC theorem with a real-world example. How does it improve on CAP?**

> CAP only describes what happens **during a partition**. But in practice, partitions are rare — most of the time, systems run normally. PACELC fills this gap: even under normal operation, **replicating data to multiple nodes synchronously adds latency** (you wait for all replicas to confirm). If you replicate asynchronously (low latency), you risk serving stale data (low consistency).
>
> **Real example — DynamoDB:** During a partition (P), it chooses Availability (A) over Consistency (C). During normal operation (E), it chooses low Latency (L) over strong Consistency (C) — writes are acknowledged before all replicas sync. If you want stronger consistency, you enable strong consistency reads, but pay with higher latency. PACELC makes this everyday tradeoff explicit, which CAP ignores.

---

**Q8. You're designing a distributed inventory system for flash sales. Items can't go below 0 stock. Which CAP type do you need?**

> **CP** is mandatory here. Overselling (stock going below 0) is a critical correctness violation — worse than a brief checkout failure. During a partition, the system must reject purchases rather than risk two nodes both selling the last item.
>
> Implementation: Use **pessimistic locking** or **compare-and-swap (CAS)** operations. With DynamoDB: use `ConditionExpression: stock > 0` on every write — if the condition fails, reject the purchase. Alternatively, use a CP system like **Redis with WATCH/MULTI/EXEC** or **PostgreSQL with serializable isolation** for the inventory counter.

---

**Q9. DNS is often cited as an AP system. Can you explain why, with an example?**

> DNS prioritises **Availability** and **Partition Tolerance** over strong Consistency. When you update a domain's IP address (e.g., deploying a new server), the change propagates gradually across DNS resolvers worldwide — this can take minutes to 48 hours (TTL-controlled). During this window, different users in different regions may resolve the same domain to different IPs (stale vs. updated).
>
> This is intentional: DNS never refuses to resolve a domain just because it might be slightly stale. Availability is more important than having every resolver instantly agree. The eventual consistency is acceptable because DNS changes are relatively infrequent and the staleness window is predictable via TTL values.

---

**Q10. An interviewer asks: "Can you build a CA system in production?" How do you answer?**

> Technically yes — but only as a **single-node system**. A single PostgreSQL instance with no replication is both consistent and available, and since there's only one node, there are no network partitions to worry about. However, this is not a distributed system.
>
> In any **multi-node distributed system**, network partitions are inevitable (the network will fail at some point). Once a partition occurs, a CA system has no strategy — it either violates consistency or availability anyway. This is why the CAP theorem says CA is impractical for distributed systems: you're forced to choose CP or AP the moment a real partition happens. Production distributed systems always need P, making the real choice C vs A.