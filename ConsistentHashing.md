# 🔄 Consistent Hashing — Revision Notes

---

## 1. The Problem with Traditional Hashing

In a distributed system, a simple approach to route requests is:

```
server = Hash(key) mod N        (N = number of servers)
```

### Example — 5 Servers (S0–S4)
- User IP is hashed → `Hash(IP) mod 5` → routed to a server
- Works fine **as long as N stays constant**

### What Breaks When N Changes?

#### Adding a server (N: 5 → 6)
```
Old: Hash(key) mod 5  →  server = 2
New: Hash(key) mod 6  →  server = 4   ← DIFFERENT!
```
- Most keys remap to **completely different servers**
- **Massive cache misses**, session loss, DB overload

#### Removing a server (N: 5 → 4)
```
Old: Hash(key) mod 5  →  server = 3
New: Hash(key) mod 4  →  server = 1   ← DIFFERENT!
```
- Even though only **1 server** was removed, **~80% of keys** are reassigned
- Causes: session loss, cache invalidation, performance degradation

> **Root cause:** `mod N` ties every key to the total count of servers. Any change in N breaks almost all mappings.

---

## 2. What is Consistent Hashing?

Consistent hashing maps both **servers** and **keys** onto a **circular hash ring** (hash space: `0` to `2³² - 1`).

> **Key guarantee:** When N changes, only `k/n` keys need to be reassigned, where `k` = total keys and `n` = total nodes.

### Traditional vs Consistent Hashing

| | Traditional (`mod N`) | Consistent Hashing |
|---|---|---|
| Add/remove server | ~All keys reassigned | Only `k/n` keys reassigned |
| Cache hit rate after scaling | Drops dramatically | Stays high |
| Hotspot risk | Moderate | Low (with VNodes) |
| Complexity | Simple | Moderate |
| Used in | Simple LBs | DynamoDB, Cassandra, Redis Cluster |

---

## 3. How the Hash Ring Works

### Step 1 — Build the Ring
- Hash space: `0` to `2³² − 1`, wrapped into a circle
- Each **server** is placed on the ring: `position = Hash(server_id)`

```
            0
           /
    S3 ---*--- S0
   /               \
 S2                 S1
   \               /
    S4 -----------
```

### Step 2 — Map a Key to a Server
1. Compute `Hash(key)` → get a position on the ring
2. Move **clockwise** from that position
3. The **first server** encountered = the owner of that key

```
Hash(User_A_IP) → position 42
Move clockwise → hits S1 first
→ Route to S1
```

### Step 3 — Adding a Server (S5 between S1 and S2)
- Only keys that fall between `S1` and `S5` are moved from `S2` → `S5`
- All other keys **unaffected**

```
Before: [S1] ----keys---- [S2]
After:  [S1] --keys-- [S5] --keys-- [S2]
                 ↑
           Only these move
```

### Step 4 — Removing a Server (S4 fails)
- Keys that were on `S4` are reassigned to the **next clockwise server** (S3)
- All other keys **unaffected**

```
Before: [S3] ---S4's keys--- [S4] --- [S0]
After:  [S3] ---S4's keys-----------  [S0]
               ↑
          S4's keys move to S3
```

---

## 4. Virtual Nodes (VNodes)

### Problem with Basic Consistent Hashing
- With few servers, positions may **cluster together**
- One server may own a large arc → **uneven load (hotspot)**
- When a server fails, **all** its load goes to a single neighbor → overload

### Solution — Virtual Nodes
Each physical server gets **multiple positions** on the ring by hashing it multiple times:

```
S1 → Hash("S1-0"), Hash("S1-1"), Hash("S1-2") ... (e.g. 150 vnodes)
S2 → Hash("S2-0"), Hash("S2-1"), Hash("S2-2") ...
S3 → Hash("S3-0"), Hash("S3-1"), Hash("S3-2") ...
```

### Without VNodes (3 servers, 1 position each)
```
Ring:  [S1 @ 10] -------- [S2 @ 50] -------- [S3 @ 90]
       S1 owns 10–50       S2 owns 50–90       S3 owns 90–10
```
If S1 fails → **all S1 load dumps onto S2**

### With VNodes (3 servers, multiple positions)
```
Ring:  [S1][S3][S2][S1][S2][S3][S1][S2][S3]...
```
If S1 fails → its load spreads **evenly across S2 and S3**

### Benefits of VNodes

| Benefit | Description |
|---|---|
| **Even distribution** | Load spread uniformly across all nodes |
| **Graceful failure** | Failed node's load shared by many, not one |
| **Heterogeneous nodes** | Powerful servers get more VNodes → handle more load |
| **Easier scaling** | New node takes small slices from many existing nodes |

> **Typical VNode count:** 100–200 per physical node (used by Cassandra, DynamoDB)

---

## 5. Code Implementation (Python)

```python
import hashlib
import bisect

class ConsistentHashing:
    def __init__(self, servers=None, num_replicas=150):
        self.num_replicas = num_replicas   # virtual nodes per server
        self.ring = {}                     # hash_position → server
        self.sorted_keys = []              # sorted list of positions
        self.servers = set()
        for server in (servers or []):
            self.add_server(server)

    def _hash(self, key):
        """MD5 hash → large integer position on ring"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_server(self, server):
        self.servers.add(server)
        for i in range(self.num_replicas):
            h = self._hash(f"{server}-{i}")
            self.ring[h] = server
            bisect.insort(self.sorted_keys, h)

    def remove_server(self, server):
        self.servers.discard(server)
        for i in range(self.num_replicas):
            h = self._hash(f"{server}-{i}")
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_server(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
```

### Key Points
- `bisect.bisect()` finds the next clockwise position in O(log N)
- Wraps around (`% len`) to handle keys that fall past the last node
- MD5 used for uniform distribution across the ring

---

## 6. Real-World Usage

| System | How it uses Consistent Hashing |
|---|---|
| **Amazon DynamoDB** | Distributes data partitions across nodes |
| **Apache Cassandra** | Routes writes/reads to correct replica nodes |
| **Redis Cluster** | Maps keys to hash slots (16384 slots) |
| **Memcached** | Client-side consistent hashing for cache sharding |
| **Nginx / HAProxy** | Consistent hashing for upstream load balancing |
| **CDNs (Akamai)** | Routes requests to edge servers |

---

## 7. Consistent Hashing with Replication

For fault tolerance, a key is often stored on **multiple consecutive nodes** (clockwise):

```
Replication factor = 3
Key K maps to S2 → also stored on S3 and S4 (next 2 clockwise)
```

- Used by **Cassandra** and **DynamoDB**
- Read/write quorum rules apply: `W + R > N` for strong consistency

---

## 8. Quick Summary

| Concept | Key Takeaway |
|---|---|
| **`mod N` problem** | Any change in N remaps ~all keys — catastrophic |
| **Hash ring** | Both servers and keys placed on a circular space |
| **Clockwise lookup** | Key goes to the first server clockwise from its hash |
| **Add server** | Only keys in the new server's arc are remapped |
| **Remove server** | Only that server's keys go to next clockwise node |
| **k/n guarantee** | Only 1/N of keys move when 1 node is added/removed |
| **Virtual nodes** | Each server has multiple ring positions → even load |
| **VNode count** | More VNodes = better balance, more memory overhead |

---

---

# 🎯 Most Asked Interview Questions (Scenario-Based)

---

**Q1. You have a cache cluster of 5 nodes using `mod N` hashing. One node goes down. What happens and how would you fix it?**

> With `mod 5`, all keys remapped to `mod 4` — roughly **80% of cache keys** suddenly point to different nodes, causing a **cache stampede**: all those requests hit the database simultaneously. This can overwhelm the DB and cause cascading failures.
>
> **Fix:** Replace `mod N` with **consistent hashing**. When the node goes down, only the keys that were on that node (~20%) move to the next clockwise node. The other 80% are unaffected. Adding virtual nodes ensures the failed node's load spreads across all remaining nodes evenly, not just one neighbor.

---

**Q2. You're designing a distributed cache (like Redis Cluster). How do you ensure even data distribution and handle node failures gracefully?**

> Use consistent hashing with **virtual nodes**.
> - Assign each physical node 100–200 virtual node positions on the ring
> - Keys always route to the next clockwise virtual node
> - On failure: the dead node's virtual nodes are spread across many other nodes, so the load is absorbed evenly — not dumped on one neighbor
> - On scaling: new node takes small slices from every existing node, keeping distribution uniform

---

**Q3. Your system uses consistent hashing. You add a powerful new server that should handle 3× more traffic than others. How do you support this?**

> Use **weighted virtual nodes**. Assign the powerful server **3× more virtual nodes** than regular servers (e.g., 450 VNodes vs 150). Since VNodes represent ring positions, more positions = larger total arc = more keys assigned to that server. This naturally distributes proportionally more load to stronger hardware without changing the hashing logic.

---

**Q4. In Cassandra, how does consistent hashing enable replication?**

> Cassandra places each node on a hash ring. When a write comes in, it's hashed to a position and assigned to the first node clockwise (the **coordinator**). That node then replicates the data to the **next N-1 nodes clockwise** (where N = replication factor, typically 3). Reads use a **quorum** (`R + W > N`) to ensure consistency. This means data is always available even if 1 or 2 nodes fail, without any centralized coordination.

---

**Q5. You're building a URL shortener at massive scale. Short URLs must always resolve to the same backend server for caching. How do you route traffic?**

> Hash the short URL key using consistent hashing to always route to the same server. This gives **cache locality** — the same server caches the same URLs, maximizing cache hit rate.
>
> When a server is added/removed, only a small fraction of URL mappings change. The rest continue hitting the same server with warm caches. Compare this to `mod N` hashing: adding a server would invalidate ~83% of the cache in a 6-node cluster.

---

**Q6. A single node in your consistent hashing ring is getting disproportionately high traffic. What's the likely cause and fix?**

> **Likely cause:** Too few virtual nodes, causing that server to own a large arc of the ring. A large arc means more keys hash into that range → more traffic.
>
> **Fix:**
> 1. Increase the number of virtual nodes per server (e.g., 150–200) for better spread
> 2. Check if the hash function is producing skewed output — switch to a better one (MD5, SHA-1, xxHash)
> 3. Monitor ring arc sizes and rebalance if a server owns >2× its fair share

---

**Q7. What happens during a "thundering herd" in a cache cluster and how does consistent hashing help?**

> A **thundering herd** happens when a cache node goes down and all its keys suddenly miss, sending a flood of requests to the database simultaneously.
>
> Consistent hashing limits this because only `k/n` keys are remapped (e.g., 20% for a 5-node cluster). Traditional hashing remaps ~80–100% of keys, causing a far larger herd.
>
> Additional mitigations: **request coalescing** (only one request fetches a missed key, others wait), **circuit breakers** on the DB, and **staggered TTLs** to avoid mass expiry.

---

**Q8. How would you implement a distributed rate limiter using consistent hashing?**

> Each user's rate limit counter needs to live on a **single node** for consistency (otherwise you'd need distributed coordination for every request).
>
> Use consistent hashing to map `user_id` → a specific rate-limiter node. All requests for that user are routed to the same node, which tracks and enforces the limit locally. When a rate-limiter node goes down, only that slice of users temporarily loses rate limiting — acceptable as a tradeoff vs. distributed coordination overhead. Tools: Redis with consistent hashing client (e.g., `redis-py` with `ketama` hashing).

---

**Q9. Compare consistent hashing with rendezvous hashing. When would you use each?**

> Both solve the same problem (minimal remapping on topology change), but differ in approach:
>
> | | Consistent Hashing | Rendezvous Hashing |
> |---|---|---|
> | Lookup | O(log N) binary search | O(N) score computation |
> | Memory | Ring + sorted array | Just server list |
> | Load balance | VNodes needed for balance | Naturally uniform |
> | Complexity | Higher | Simpler |
>
> Use **consistent hashing** for large clusters where O(log N) lookup matters (thousands of nodes). Use **rendezvous hashing** for small clusters where simplicity is preferred and O(N) lookup cost is acceptable.

---

**Q10. You're designing DynamoDB's partition layer. Walk through how consistent hashing is used end-to-end.**

> 1. **Ring setup:** All DynamoDB storage nodes are placed on a hash ring using their node IDs
> 2. **Partition key hashing:** When a write arrives, the partition key is hashed → position on ring
> 3. **Node assignment:** The first clockwise node becomes the **primary** for that key range
> 4. **Replication:** The next 2 clockwise nodes also store a copy (replication factor = 3)
> 5. **Reads/Writes:** Use quorum (`W=2, R=2` out of `N=3`) for strong consistency
> 6. **Adding a node:** New node takes over a contiguous arc from an existing node, which ships only that slice of data
> 7. **Node failure:** Its key range is served by replicas; the system auto-heals by re-replicating to maintain RF=3
> 8. **VNodes:** DynamoDB uses virtual nodes internally so each physical server owns many small arcs for even balance