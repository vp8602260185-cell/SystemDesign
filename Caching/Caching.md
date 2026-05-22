# 🗂️ Caching — Complete Revision Notes

> Covers: What is Caching · Cache-Aside · Read-Through & Write-Through · Write-Behind · Caching Strategies · Eviction Policies

---

## 1. What is Caching?

**Caching** = storing a copy of frequently accessed data in a fast-access layer (memory) so future requests are served faster without hitting the origin (database/API).

### Why It Matters
- Reduces **latency** (memory access ~ns vs DB ~ms)
- Reduces **load** on databases and downstream services
- Improves **throughput** and **scalability**

### Key Terms

| Term | Meaning |
|------|---------|
| **Cache Hit** | Data found in cache → served immediately |
| **Cache Miss** | Data NOT in cache → fetched from source, then cached |
| **Hit Rate** | `hits / (hits + misses)` — higher is better |
| **Cache Warm-up** | Pre-loading cache before traffic hits |
| **Cold Cache** | Empty/freshly started cache; high miss rate |
| **Thundering Herd** | Many requests hit DB simultaneously on cache miss |

### Cache Layers (from fastest to slowest)
```
CPU Registers → L1/L2/L3 Cache → RAM → Disk → Network (DB/API)
```
In system design context:
```
Client (browser cache) → CDN → App-level cache (Redis/Memcached) → Database
```

### What to Cache ✅
- Expensive query results
- Session data
- API responses from external services
- Computed/aggregated results (e.g., leaderboards)
- Static assets (CDN)

### What NOT to Cache ❌
- Highly dynamic data (changes every request)
- Sensitive/private user-specific data (security risk)
- Data that must always be consistent in real-time

### Cache Consistency
- Cache and DB can go **out of sync** → stale data problem
- Strategies to handle: TTL (time-to-live), invalidation on write, versioning

### Caching Anti-Patterns
| Anti-Pattern | Problem |
|---|---|
| Caching everything | Wastes memory, complex invalidation |
| No TTL | Stale data forever |
| Cache stampede | All entries expire at once → DB overload |
| Over-caching sensitive data | Privacy/security risk |

### Measuring Cache Performance
- **Hit rate** (target > 90% for production)
- **Latency** (p50, p95, p99)
- **Memory usage**
- **Eviction rate** (high rate = cache is too small)

---

## 2. Cache-Aside Pattern (Lazy Loading)

> App code manages the cache manually. Cache sits **beside** the database.

### How It Works
```
1. App checks cache for data
2. Cache HIT  → return data
3. Cache MISS → fetch from DB → store in cache → return data
```

### Diagram
```
Client → App → Cache (miss?) → DB → App stores in Cache → Client
```

### Pros ✅
- Only **used** data is cached (memory efficient)
- Cache failure doesn't break the app (can still hit DB)
- Works well with **read-heavy** workloads

### Cons ❌
- **Cache miss penalty**: first request always slower
- **Stale data**: DB updates don't auto-update cache
- **Thundering herd**: many misses at once → DB flood

### Best For
- Read-heavy apps where stale data is tolerable
- When you want fine-grained control over what gets cached

### Example
```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if not user:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(f"user:{user_id}", user, ttl=300)
    return user
```

---

## 3. Read-Through & Write-Through Cache

### Read-Through Cache

> Cache sits **in front** of the DB. App talks only to the cache; cache fetches from DB on miss.

```
Client → Cache → (miss) → DB → Cache stores → Client
```

**vs Cache-Aside:**
- In Cache-Aside, **app** loads data into cache on miss
- In Read-Through, **cache library/provider** loads data automatically

**Pros ✅**
- App code is simpler (no manual cache population)
- Data always flows through cache (good for read-heavy)

**Cons ❌**
- Cache miss still causes latency
- Requires cache provider to support read-through logic
- Initial cold-start problem

---

### Write-Through Cache

> Every **write** goes to cache AND DB **synchronously** at the same time.

```
Client → App → Cache (write) + DB (write) → Confirm
```

**Pros ✅**
- Cache always has **up-to-date data**
- No stale reads after writes
- Great paired with **Read-Through**

**Cons ❌**
- **Write latency** is higher (must write to both)
- Caches data that may never be read (write amplification)
- Not ideal for write-heavy workloads

**Best For:** Read-heavy apps where data freshness is critical (user profiles, product prices)

---

### Read-Through + Write-Through Together
This combo ensures:
- Reads always come from cache (fast)
- Writes always keep cache fresh (consistent)
- App never talks to DB directly

---

## 4. Write-Behind Cache (Write-Back)

> Writes go to cache **immediately**, DB is updated **asynchronously** later.

```
Client → App → Cache (write, returns OK) → [async] → DB
```

### How It Works
1. Write is acknowledged immediately after updating cache
2. A background process (or event queue) flushes cache changes to DB in batches

### Pros ✅
- **Lowest write latency** (fire-and-forget to DB)
- **Batching** reduces DB load (multiple writes → one DB call)
- Great for **write-heavy** workloads (analytics, logging, counters)

### Cons ❌
- **Risk of data loss**: if cache crashes before flush → writes lost
- **Consistency lag**: DB lags behind cache temporarily
- More complex to implement and debug
- Not suitable where DB must always be authoritative

### Write-Behind vs Write-Through

| | Write-Through | Write-Behind |
|---|---|---|
| DB write timing | Synchronous (immediate) | Asynchronous (delayed) |
| Write latency | Higher | Lower |
| Data safety | Safe | Risk of loss |
| Best for | Read-heavy, strong consistency | Write-heavy, throughput |

---

## 5. Caching Strategies — When to Use What

### Summary Table

| Strategy | Who populates cache? | Write behavior | Best for |
|---|---|---|---|
| **Cache-Aside** | Application (on miss) | App updates DB; cache invalidated/expired | Read-heavy, flexible |
| **Read-Through** | Cache provider (on miss) | — | Read-heavy, simpler app code |
| **Write-Through** | On every write | Cache + DB simultaneously | Strong consistency |
| **Write-Behind** | On every write | Cache now, DB later (async) | Write-heavy, high throughput |
| **Refresh-Ahead** | Cache pre-fetches | — | Predictable access patterns |

### Refresh-Ahead (Bonus)
- Cache **proactively refreshes** data before it expires
- Prevents cache miss latency for frequently accessed hot data
- Risk: may fetch data that won't be requested again (wasted work)

### Choosing a Strategy — Decision Guide
```
Is your workload read-heavy?
├── Yes → Cache-Aside or Read-Through
│         Need strong consistency? → Add Write-Through
│         Can tolerate stale data? → Cache-Aside with TTL
└── No (write-heavy)?
    ├── Can tolerate data loss risk? → Write-Behind
    └── Need durability? → Write-Through (accept higher latency)
```

---

## 6. Cache Eviction Policies

> When cache is full, which data gets removed to make space?

### LRU — Least Recently Used ⭐ (most common)
- Evicts the item that was **accessed longest ago**
- Works well for most general workloads
- Assumption: recently used = likely to be used again

```
Access order: A → B → C → A → (cache full, add D)
Evict: B (least recently used)
```

### LFU — Least Frequently Used
- Evicts the item with the **lowest access count**
- Better for workloads with stable "hot" data
- Con: New items can get evicted before getting a chance (recency bias problem)

```
Counts: A=10, B=2, C=7 → Evict B
```

### FIFO — First In, First Out
- Evicts the **oldest inserted** item regardless of usage
- Simple but ignores access patterns
- Rarely used in production caches

### MRU — Most Recently Used
- Evicts the **most recently used** item
- Niche use case: access pattern where latest data won't be reused (e.g., video streaming)

### Random Replacement
- Evicts a **random** item
- Surprisingly effective in some workloads, very simple
- Used in CPU caches

### TTL — Time To Live (not strictly eviction, but related)
- Each item has an expiry time
- Expired items are evicted on access or by background cleanup
- Always use TTL alongside eviction policies

### Comparison Table

| Policy | Evicts | Best For | Weakness |
|---|---|---|---|
| **LRU** | Least recently used | General purpose | Cache pollution from one-time scans |
| **LFU** | Least frequently used | Stable hot-data workloads | Slow to adapt to new hot data |
| **FIFO** | Oldest inserted | Simple, ordered data | Ignores usage patterns |
| **MRU** | Most recently used | Sequential/streaming access | Counter-intuitive for most apps |
| **Random** | Random item | Low overhead needs | Unpredictable |
| **TTL** | Expired items | Time-sensitive data | Thundering herd on mass expiry |

### Tips for Choosing Eviction Policy
- **Default choice:** LRU (Redis default, Memcached default)
- **Skewed hot data (80/20 rule):** LFU works better
- **Use TTL always** to prevent permanent stale data
- **Add jitter to TTL** to prevent thundering herd (e.g., TTL = 300 ± random(0–30)s)

---

## Quick Reference — One-Liner Summaries

| Topic | One-Liner |
|---|---|
| **Caching** | Store expensive data in fast memory to avoid repeated slow lookups |
| **Cache-Aside** | App checks cache → miss → loads from DB → stores in cache |
| **Read-Through** | Cache auto-fetches from DB on miss; app only talks to cache |
| **Write-Through** | Every write goes to cache AND DB at the same time (strong consistency) |
| **Write-Behind** | Write to cache now, flush to DB later async (high write throughput) |
| **LRU** | Drop the item you haven't used in the longest time |
| **LFU** | Drop the item you've used the fewest times |
| **TTL** | Every cached item has an expiry — stale data auto-clears |

---

## Common Interview Questions

1. **What's the difference between Cache-Aside and Read-Through?**
   → In Cache-Aside, the *app* populates the cache on miss. In Read-Through, the *cache layer* does it automatically.

2. **When would you use Write-Behind over Write-Through?**
   → Write-Behind for write-heavy systems (lower latency, batching). Write-Through when consistency between cache and DB is critical.

3. **How do you prevent cache stampede (thundering herd)?**
   → Add TTL jitter, use mutex/locks for first miss, or use Refresh-Ahead strategy.

4. **What eviction policy does Redis use by default?**
   → `noeviction` (returns error when full), but `allkeys-lru` is common in production configs.

5. **How do you handle cache invalidation?**
   → TTL (passive expiry), event-driven invalidation on write, or versioned cache keys.

---

*Source: AlgoMaster System Design Course — algomaster.io*

---
---

# 🌐 Caching — Advanced Topics (Part 2)

> Covers: Distributed Caching · Cache Invalidation · Cache Stampede · Cache Warming

---

## 7. Distributed Caching

**Distributed caching** = spreading cache data across **multiple nodes (servers)** instead of a single machine, so the cache can scale horizontally with the system.

### Why Not a Single Cache Node?
A single-node cache has limits:
- **Memory cap** — one server's RAM runs out
- **Single point of failure** — goes down → cold cache → DB flood
- **Bottleneck** — all traffic hits one machine

### Why Use Distributed Caching?

| Benefit | Explanation |
|---|---|
| **Scalability** | Add more cache nodes as traffic grows |
| **Fault Tolerance** | One node fails → others still serve data |
| **Load Balancing** | Traffic and data spread evenly across nodes |

### Core Components

| Component | Role |
|---|---|
| **Cache Nodes** | Individual servers storing cache data |
| **Client Library** | App-side code that routes keys to the right node |
| **Consistent Hashing** | Distributes keys across nodes; minimal reshuffling when nodes are added/removed |
| **Replication** | Copies data to backup nodes for fault tolerance |
| **Sharding** | Splits data into partitions — each node owns a shard |
| **Eviction Policies** | LRU / LFU / TTL to free space (see Part 1) |
| **Distributed Locks** | Prevents race conditions when multiple nodes write the same key |

### Consistent Hashing (Key Concept)
```
Normal hashing: key % N nodes → adding/removing a node reshuffles almost everything

Consistent hashing: nodes placed on a ring → only keys near the removed node are remapped
→ Minimizes cache invalidation on scale up/down
```

### Dedicated Cache Servers vs. Co-located Cache

| | Dedicated (e.g., separate Redis cluster) | Co-located (cache on app server) |
|---|---|---|
| **Latency** | Slightly higher (network hop) | Lowest (same machine) |
| **Scalability** | Scale cache independently ✅ | Tied to app server scaling |
| **Cost** | Higher (extra servers) | Lower |
| **Resource contention** | None | Cache competes with app for CPU/RAM |
| **Best for** | Large-scale systems | Small apps, real-time / HFT |

### How It Works End-to-End
```
1. App calls cache.get("user:42")
2. Client library hashes the key → finds the right node
3. Cache HIT → return data
4. Cache MISS → fetch from DB → store on that node → return data
5. Replication copies the value to backup node(s)
6. TTL/eviction keeps memory in check
```

### Challenges
- **Data consistency** — nodes can briefly disagree on a value
- **Cache invalidation** — invalidating across multiple nodes is complex
- **Network partitions** — nodes may be unable to communicate (CAP theorem applies)
- **Hot keys** — one popular key overwhelms a single shard

### Popular Solutions

| Tool | Key Traits |
|---|---|
| **Redis** | Rich data structures, persistence, replication, pub/sub, Lua scripting |
| **Memcached** | Pure in-memory, no persistence, extremely fast, simple key-value only |
| **Amazon ElastiCache** | Managed Redis/Memcached on AWS, auto-failover, multi-AZ |

**Redis vs Memcached:**
- Use **Redis** when: you need persistence, data structures (sorted sets, lists), pub/sub, or Lua scripts
- Use **Memcached** when: you need pure raw speed, simple key-value, and horizontal scaling with no frills

### Best Practices
1. Cache only frequently accessed, relatively stable data
2. Always set TTLs — never cache indefinitely
3. Use Cache-Aside pattern to load lazily
4. Monitor hit rate, memory usage, eviction rate
5. Plan for node failure — app must fall back to DB gracefully
6. Pre-warm cache on deploy to avoid cold starts

---

## 8. Cache Invalidation

> **"There are only two hard things in Computer Science: cache invalidation and naming things."** — Phil Karlton

**Cache invalidation** = removing or updating cached data when the source (DB) changes, so users never see stale data beyond an acceptable threshold.

The core problem: once you add a cache, you now have **two sources of truth** — the cache and the DB. Keeping them in sync is the challenge.

### Why It's Hard
- Writes can come from multiple services simultaneously
- Cache and DB updates are two separate operations — not atomic
- Distributed caches have multiple nodes to invalidate
- Race conditions between reads and writes can re-insert stale data

### Invalidation Strategies

#### 1. TTL-Based (Time-To-Live) — Passive Expiry
- Every cached item gets an expiry time
- After TTL expires, next read triggers a cache miss → fresh fetch
- **Simple** but data can be stale for up to TTL duration

```
cache.set("product:99", data, ttl=300)  # stale for up to 5 min
```

**Pros:** Simple, no write-side logic  
**Cons:** Staleness window = TTL; thundering herd on mass expiry

#### 2. Event-Driven Invalidation — Active Expiry
- When data changes in DB, explicitly **delete or update** the cache entry
- Immediate consistency

```
# On DB update:
db.update("UPDATE products SET price=50 WHERE id=99")
cache.delete("product:99")   # or cache.set("product:99", new_data)
```

**Pros:** Near real-time consistency  
**Cons:** Every write path must also touch cache; bugs → inconsistency

#### 3. Write-Through (covered in Part 1)
- Cache is updated on every DB write automatically
- Cache is always fresh

#### 4. CDC — Change Data Capture
- A background process (e.g., Debezium) watches DB transaction logs
- On any DB change → event published → cache updated automatically
- Decouples cache invalidation from application code

```
DB binlog → Debezium → Kafka → Cache invalidation service → Redis.delete(key)
```

**Pros:** App code stays clean; works across microservices  
**Cons:** Added infrastructure complexity; slight delay (eventual consistency)

### Race Conditions in Invalidation

**Classic race (read-then-write):**
```
T1: cache miss → reads from DB (old value = $10)
T2: DB updated to $20 → cache.delete("price")
T1: cache.set("price", $10)   ← stale value re-inserted!
```

**Solutions:**
- **Versioned keys**: `cache.set("product:99:v5", data)` — old versions are simply ignored
- **Distributed locks**: Only one process writes the cache key at a time
- **Short TTL as safety net**: Even if stale data gets inserted, it expires soon

### Invalidation in Distributed Systems
- Must invalidate the key on **all nodes** — use pub/sub or a central invalidation bus
- **Broadcast invalidation**: publish a delete event to all cache nodes
- **Tag-based invalidation**: group related keys under a tag, invalidate the whole tag at once
  ```
  tag: "user:42" → keys: [user:42:profile, user:42:orders, user:42:prefs]
  invalidate_tag("user:42") → all three deleted at once
  ```

### Best Practices
- Combine TTL (safety net) + event-driven invalidation (immediate) for production
- Keep invalidation logic close to the write path
- Use versioned/namespaced keys to make stale-data bugs visible
- Add TTL jitter to prevent mass simultaneous expiry
- Monitor stale hit rate alongside overall hit rate

---

## 9. Cache Stampede (Thundering Herd)

A **cache stampede** happens when a popular cache entry expires and **many concurrent requests** all miss at once, all querying the DB simultaneously to rebuild the same entry — overwhelming it with identical redundant queries.

```
Popular key "homepage" expires at T=0
→ 1000 requests/sec all get cache miss
→ 1000 DB queries fired at once for the same data
→ DB gets overwhelmed → timeouts → cascading failures
```

### Why It Happens
- High-traffic key with a hard expiry
- Sudden spike + empty cache (after deploy or cache flush)
- Multiple app servers all independently detect the miss

### Impact
- DB CPU spikes to 100%
- Response times shoot up (p99 goes from ms → seconds)
- Timeouts cascade to other queries
- Can cause full outage even though the app code is fine

### Prevention Strategies

#### 1. Mutex Lock (Cache Lock / Single-Flight)
- Only **one** request fetches from DB; all others **wait** for it
- The winner populates the cache; waiters read the freshly cached value

```python
if not cache.get(key):
    lock = acquire_lock(key, timeout=5s)
    if lock:
        data = db.fetch()
        cache.set(key, data, ttl=300)
        release_lock(key)
    else:
        wait_and_retry()  # another thread is fetching
```

**Pros:** Exactly one DB query  
**Cons:** Waiters are blocked; lock contention at very high scale

#### 2. Probabilistic Early Expiration (PER)
- Before the TTL actually expires, **probabilistically** refresh the cache early
- Higher traffic → higher chance of early refresh → smooth handoff

```
remaining_ttl = cache.ttl(key)
if random() < (1 / remaining_ttl):   # more likely as TTL gets low
    refresh_cache(key)
```

**Pros:** No locks, works well at scale  
**Cons:** Slightly complex; may cause occasional premature refreshes

#### 3. TTL Jitter (Staggered Expiry)
- Add random offset to TTL so not all keys expire simultaneously

```python
ttl = 300 + random.randint(0, 60)   # 5–6 min instead of exactly 5 min
cache.set(key, data, ttl=ttl)
```

**Pros:** Simple, widely used  
**Cons:** Doesn't help when a single hot key expires

#### 4. Background Refresh (Refresh-Ahead)
- A background job refreshes hot keys **before** they expire
- Cache always has a valid value; clients never see a miss

```
Cron / background thread:
every 4 min → refresh all keys with TTL of 5 min
```

**Pros:** Zero miss latency for hot keys  
**Cons:** Need to know which keys are hot; wastes work if data isn't requested

#### 5. Serve Stale While Revalidating
- When TTL expires, **serve the stale value** immediately
- Kick off a background fetch to update the cache
- Next request gets the fresh value

**Pros:** Zero added latency for user  
**Cons:** One request always sees stale data

### Strategies Comparison

| Strategy | Latency Impact | Complexity | Best For |
|---|---|---|---|
| Mutex Lock | Waiters blocked briefly | Medium | Moderate traffic |
| Probabilistic Early Expiry | None | Medium | High traffic, hot keys |
| TTL Jitter | None | Low | Preventing mass expiry |
| Background Refresh | None | Medium | Known hot keys |
| Stale-While-Revalidate | None (serves stale) | Low | Eventual consistency ok |

### Monitoring & Detection
- Watch for sudden **DB query rate spikes** correlated with cache miss spikes
- Alert on `cache_miss_rate > threshold` for specific keys
- Track `db_query_rate / cache_hit_rate` ratio — sudden rise = stampede

---

## 10. Cache Warming

**Cache warming** = pre-populating the cache with data **before** real traffic hits, to avoid the cold cache problem.

### The Cold Cache Problem
```
Deploy → cache is empty → every request = cache miss
→ all traffic hits DB → 10–20× normal DB load
→ latency spikes, timeouts, potential outage
```
This happens after: fresh deploys, cache server restarts, cache flushes, or scaling out new cache nodes.

### When You Need Cache Warming
- High-traffic applications where DB can't handle full raw load
- Apps with slow queries that are cheap to pre-compute
- After any event that drains the cache (restart, flush, new node)
- Before planned traffic spikes (product launches, flash sales)

### Warming Strategies

#### 1. Lazy Warming (Organic)
- Don't pre-populate; let the cache fill naturally as users make requests
- **Simple** but the first wave of users sees high latency
- Only acceptable for low-traffic systems or non-critical paths

#### 2. Pre-warming Script (Eager / Proactive)
- Before going live, run a script that loads the most important keys

```python
# Before deploy goes live:
hot_products = db.query("SELECT * FROM products ORDER BY views DESC LIMIT 1000")
for p in hot_products:
    cache.set(f"product:{p.id}", p, ttl=3600)
```

**Pros:** Cache is ready before traffic  
**Cons:** Must identify what to warm; warming script takes time to run

#### 3. Replay Traffic / Shadow Warming
- Replay recent production request logs against the new cache
- Simulates real traffic patterns to populate the most relevant keys

**Pros:** Very accurate — warms exactly what users will request  
**Cons:** Needs access to request logs; complex setup

#### 4. Gradual Traffic Rollout (Canary)
- Don't switch 100% of traffic to new deployment at once
- Start at 1% → 5% → 25% → 100%
- Cache fills gradually; DB load increases slowly

**Pros:** Safe, progressive, no need for a separate warming step  
**Cons:** Slower rollout; requires traffic-splitting infrastructure

#### 5. Cache Snapshot / Persistence
- Dump the cache to disk (Redis RDB/AOF) before restart
- Reload the snapshot on startup → cache is warm immediately

```bash
# Redis: save snapshot
redis-cli BGSAVE
# On restart, Redis auto-loads the RDB file
```

**Pros:** Near-instant warm-up  
**Cons:** Snapshot may be stale; only works for Redis-like systems with persistence

### What to Warm — Identifying Hot Keys
- **Access logs**: top N most-requested keys from the past 24h
- **Redis `MONITOR` or `--hotkeys`**: identify current hot keys in a live cache
- **Business logic**: known hot data — homepage, top products, config values, user sessions
- **Analytics**: top 1000 user IDs, top 100 product IDs, etc.

### Best Practices
- Always warm **before** flipping traffic to a new deployment
- Prioritize: home page > top products > user sessions > everything else
- Use gradual rollout as a complement, not a replacement, for warming
- Add TTL jitter even on pre-warmed keys to avoid synchronized expiry later
- Monitor cache hit rate after deploy — should be high within minutes, not hours

---

## Updated Quick Reference — All Topics

| Topic | One-Liner |
|---|---|
| **Distributed Caching** | Cache data across multiple nodes for scale, fault tolerance, and load balancing |
| **Consistent Hashing** | Place nodes on a ring so adding/removing nodes only remaps nearby keys |
| **Redis** | In-memory store with rich data structures, persistence, and replication |
| **Memcached** | Pure in-memory key-value; faster/simpler but no persistence or data structures |
| **Cache Invalidation** | Removing/updating stale cache entries when source data changes |
| **TTL-Based Invalidation** | Entries auto-expire after a set time — simple but has a staleness window |
| **Event-Driven Invalidation** | Explicitly delete/update cache on every DB write — immediate but more code |
| **CDC Invalidation** | Watch DB transaction log → auto-invalidate cache — clean but complex infra |
| **Cache Stampede** | Many requests hit DB at once when a popular cache key expires |
| **Mutex Lock** | Only one request fetches from DB on miss; others wait — prevents stampede |
| **TTL Jitter** | Randomize expiry times to prevent keys expiring all at once |
| **Cache Warming** | Pre-populate cache before traffic hits to avoid cold-start DB overload |
| **Gradual Rollout** | Shift traffic slowly so cache fills organically without DB shock |

---

## Extended Interview Q&A

**Q: What is consistent hashing and why does distributed caching use it?**
→ Consistent hashing places nodes on a virtual ring. Keys are also hashed to a position on the ring and assigned to the nearest node clockwise. When a node is added/removed, only the keys on that segment are remapped — vs. regular hashing where all keys move.

**Q: How do you invalidate cache across microservices?**
→ Use CDC (Change Data Capture) — listen to DB transaction log events and publish invalidation messages via a message bus (Kafka). Each service's cache consumer deletes/updates the relevant key.

**Q: What is the difference between cache invalidation and cache eviction?**
→ **Invalidation** = explicitly removing an entry because the underlying data changed (correctness concern). **Eviction** = removing an entry because the cache is full and needs space (capacity concern).

**Q: How would you prevent a cache stampede during a major product launch?**
→ Combination: pre-warm the cache before launch, use TTL jitter, add a mutex lock for the product page key, and deploy gradually (canary). Also have circuit breakers on the DB layer.

**Q: Redis vs Memcached — when do you choose which?**
→ Redis for: persistence, rich data types (sorted sets for leaderboards, lists for queues), pub/sub, Lua scripting. Memcached for: pure caching speed, horizontal scaling with no extra features needed.

**Q: What is stale-while-revalidate?**
→ Serve the cached (possibly stale) value immediately to avoid latency, while a background process fetches the fresh value and updates the cache. The next request gets fresh data.

---

*Source: AlgoMaster System Design Course — algomaster.io*