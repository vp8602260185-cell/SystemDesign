# Scaling Write Traffic: Revision Notes

Source: AlgoMaster.io, System Design Interviews course (Interview Patterns > Scaling Under Load). Notes are condensed and in my own words.

---

## 1. Where this pattern shows up

| Problem | What gets written |
|---|---|
| Uber | Driver location every few seconds |
| WhatsApp | Every message (billions/day) |
| YouTube | View counts, like counts |
| Ad click tracking | Every click (advertisers are billed on it) |
| Metrics/logging (Datadog) | Data points from thousands of servers |

## 2. Why writes are harder than reads

- Caches and CDNs do not help: every write carries **new data** that must be stored.
- Read replicas do not help: every write still goes to the primary, and each replica must also apply it.
- Each write does more work than it looks: write-ahead log, table update, **every index** update, sometimes row locks.
- Example: 5M drivers, 1 update every 4 sec = about **1.25M writes/sec**. One well-tuned relational DB handles tens of thousands of simple inserts/sec.
- So: make each write cheaper, and spread writes across machines.

## 3. Make each write cheaper

**Keep only the indexes you need**
- Every index is updated on every insert (5 indexes = much more work than 1).
- Read-heavy: extra indexes are worth it. Write-heavy: keep only what queries actually use.

**Choose a write-friendly database**

| | B-tree (PostgreSQL, MySQL) | LSM tree (Cassandra, ScyllaDB, RocksDB) |
|---|---|---|
| Write path | Updates pages in place, scattered on disk | Append to memory + log, flush sorted files sequentially |
| Read path | Fast point and range reads | May check several files |
| Background work | Little | Compaction (uses disk and CPU) |
| Good fit | Mixed workloads, transactions | High-volume append-only data |

- Sequential writes are much faster than scattered ones.
- LSM reads are helped by Bloom filters and compaction.
- Why Cassandra suits location history, messages, metrics.

## 4. Batching

- Each write has a fixed cost (network round trip, transaction, log write). Batch of 1,000 pays it once instead of 1,000 times.
- Example: metrics service buffers data for 1 second, then does one bulk write.
- Flush on **whichever comes first**: size limit (efficient under heavy load) or time limit (no long wait under light load).
- Tools: multi-row `INSERT ... VALUES`, PostgreSQL `COPY`, Cassandra batch statements.
- Trade-offs:
  - Adds **delay** (data waits in the buffer).
  - Buffered data can be **lost** on a crash.
- Fine for metrics/logs. Not fine for payments (store safely before confirming).

## 5. Queues to absorb spikes

- Spikes (flash sale, big match, viral post) can overwhelm the DB.
- Put a queue (e.g. **Kafka**) between app and DB: app enqueues and replies right away; **consumers** write to the DB at a rate it can handle. Queue grows during spikes and drains afterwards.
- Kafka fits well: appends to disk logs, handles very high write rates. Topics are split into **partitions**; same key (e.g. driver ID) goes to the same partition, so order is kept.

**Trade-offs**
- **Delayed visibility:** data is not in the DB when the client gets the response.
- **Duplicates:** at-least-once delivery. Example: worker writes, crashes before acking, message is redelivered.
  - Fix: **idempotent writes** (unique event ID; DB ignores IDs it already stored).
- **Growing backlog:** a queue only absorbs *short* spikes.
  - Watch **consumer lag**.
  - If it keeps rising: add consumers (up to the partition count) or larger batches.
  - Still not enough: **backpressure** (bounded buffer telling producers to slow down or retry later).

## 6. Sharding

- When one DB cannot keep up, split data across multiple DBs (**shards**), each owning part of the data and its writes.
- Example: hash driver ID to pick the shard.
- Sizing: 1.25M writes/sec / 50k per shard = at least **25 shards** (more in practice for growth and spikes).

**Choosing the shard key**
- Should spread writes evenly **and** keep data read together on one shard.

| Shard key | Write spread | Data read together | Verdict |
|---|---|---|---|
| Driver ID (hashed) | Even | A driver's history is on one shard | Good |
| Timestamp (ranges) | All new writes hit newest shard | Time ranges easy to scan | Poor for writes |
| Random ID | Even | Related data scattered | Queries touch every shard |

- Prefer designs where most writes stay inside **one shard**. Cross-shard writes (e.g. money transfer between accounts on different shards) need a saga or two-phase commit.

**Adding shards later**
- `hash(key) % N` moves almost every key when N changes.
- **Consistent hashing:** shards sit on a ring, a key goes to the next shard around the ring, adding a shard moves only about 1/N of the data.

## 7. Hot shards and hot keys

**Bad shard key**
- Timestamp ranges or auto-increment IDs: all new writes go to the newest shard, others sit idle.
- Better for time-series: composite key like `(device_id, day)` (device ID spreads writes, day groups data for range scans).

**Single popular record**
- Example: celebrity video gets 100k likes/sec, all updating one counter row on one shard.
- Fix: **split the counter** into N smaller counters (e.g. 10). Each like updates one at random; total = sum of all.
- Cost: slightly more expensive reads, usually small because popular counts are cached.

## 8. Write less data in the first place

| Technique | Example | What you give up |
|---|---|---|
| **Aggregate** | Count views in memory or a stream processor, write "views += 500" every few seconds | A few seconds of counts if the aggregator crashes |
| **Overwrite** | Keep only each driver's latest location in Redis | Full history (unless stored elsewhere) |
| **Sample** | Store 1 in 10 location pings for trip history | Detail between stored points |

- StatsD-style agents aggregate counters per interval. Time-series DBs keep recent data at full detail and roll older data into larger buckets.
- Interview habit: ask "does every write need to be stored?" Yes for payments and chat messages. Often no for view counts, metrics, location pings.

## 9. Which writes can be lost?

Many speed-ups accept risk:
- In-memory batching can lose data on a crash.
- In-memory aggregation can lose a few seconds of counts.
- **Write-behind** (ack after writing to a fast buffer like Redis, save to DB later) can lose acknowledged writes if the buffer fails first.
- Kafka producer settings: `acks=all` (copied to several brokers, safer, slower) vs `acks=1` (one broker, faster, can lose data if it fails).

| Group | Examples | Confirm to user when | Techniques that fit |
|---|---|---|---|
| **Must never be lost** | Payments, orders, chat messages | Durable and replicated | Replicated DB, queue with `acks=all`, idempotent retries |
| **Small loss OK** | View counts, analytics events, location pings | As soon as accepted | In-memory batching, aggregation, overwriting, sampling |

- Say out loud which group each write type belongs to in an interview.

## 10. Worked example: Uber location pipeline

1. Driver phone sends a location every 4 sec.
2. **API server** puts it on a **Kafka topic** partitioned by driver ID.
3. Consumer group A updates each driver's latest location in **Redis** (overwrite).
4. **Matching service** reads Redis to find nearby drivers.
5. Consumer group B **batches** updates into a **Cassandra** cluster sharded by driver ID (trip history, analytics).

| Component | Problem it solves |
|---|---|
| Kafka | Absorbs spikes |
| Redis | Keeps only current state (1.25M updates/sec overwrite 5M entries instead of adding rows) |
| Batching consumers | Fewer DB writes |
| Sharded Cassandra (LSM) | Spreads remaining writes across machines |

- Two consumer groups read the same topic, so either can lag or restart independently.
- Partitioning by driver ID keeps each driver's updates in order.

## 11. Interview approach (step by step)

1. **Clarify:** peak writes/sec, steady or spiky, can writes be delayed or lost, store every write or only latest, how data is read.
2. **Estimate** peak write load.
3. **Reduce writes:** aggregation, overwriting, batching.
4. **Pick a database** that fits (LSM for high-volume append-only).
5. **Add a queue** if spiky or if the client does not need immediate storage.
6. **Shard:** justify the shard key, handle hot keys.
7. **State durability needs** per write type.

- Tie every component to a number or requirement from an earlier step.

---

## Quick-recall cheat sheet

- Writes do not scale like reads: caches and replicas do not remove the work of storing new data.
- Order of attack: **fewer indexes, LSM database, reduce writes, batch, queue, shard, handle hot keys, decide durability**.
- Batching trades **delay and possible loss** for throughput. Flush on size or time.
- Queue = spike absorber, not a fix for sustained overload. Watch **consumer lag**, add **backpressure**.
- Queues give at-least-once delivery, so make writes **idempotent** (unique event IDs).
- Shard key: **even spread + related data together**. Never use a monotonically increasing key for range sharding.
- Hot counter: **split into N counters**, sum on read.
- Use **consistent hashing** so adding shards moves only about 1/N of the data.
- Overwrite / aggregate / sample if not every write must be stored.
- Split writes into "never lose" vs "small loss OK" and say which techniques each allows.

---

## My Notes / Questions

- 
