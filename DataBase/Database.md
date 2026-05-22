# 🗄️ Databases — Complete Revision Notes

> Covers: Database Types · SQL vs NoSQL · ACID Transactions

---

## 1. Why Different Database Types Exist

Relational databases once handled everything. But as the internet scaled, new problems emerged:

- **Massive scale** — billions of users can't fit on one machine
- **Flexible schemas** — agile dev needs schema changes without migrations
- **Specialized access patterns** — full-text search, graph traversals, time-series analytics
- **Geographic distribution** — global apps need low-latency multi-region replication

This spawned the NoSQL movement and a rich ecosystem of specialized databases. Modern systems often use **multiple database types together** (polyglot persistence).

> **Interview tip:** Don't jump to a technology. First understand: data model, access patterns, scale requirements, consistency needs. The right DB type emerges from those.

---

## 2. Database Types — The Landscape

### 2.1 Relational Databases (RDBMS)

Data stored in **tables with rows and columns**. Relationships via foreign keys. Queried with SQL.

| Characteristic | Detail |
|---|---|
| Data Model | Tables, strict schema |
| Query Language | SQL (joins, aggregations, subqueries) |
| Consistency | Strong — full ACID |
| Schema | Schema-on-write (enforced at insert time) |
| Scaling | Primarily vertical; horizontal via read replicas or sharding |

**Strengths ✅**
- ACID guarantees → data integrity (critical for finance)
- Complex queries across multiple tables via joins
- Mature ecosystem — decades of tooling and optimization
- Strong consistency — reads always return latest committed data

**Weaknesses ❌**
- Sharding is complex and can sacrifice SQL features
- Schema changes are painful (migrations)
- Poor fit for hierarchical, sparse, or highly variable data

**Popular Options**

| DB | Best For | Notable Users |
|---|---|---|
| PostgreSQL | Complex queries, extensibility, JSON support | Apple, Instagram, Spotify |
| MySQL | Web apps, read-heavy workloads | Facebook, Twitter, Netflix |
| SQL Server | Enterprise Windows environments | Stack Overflow, Fortune 500 |
| Oracle | Large enterprises, complex transactions | Banks, airlines, governments |

**Choose when:** structured data with joins, ACID needed, stable schema, scale fits a few servers.

---

### 2.2 Key-Value Stores

Simplest database type — a **giant hash map**. Look up values by key. That's it.

| Characteristic | Detail |
|---|---|
| Data Model | Key → Value (string, JSON, binary) |
| Query Language | GET / PUT / DELETE by key |
| Consistency | Varies — Redis: strong on single node; DynamoDB: configurable |
| Schema | Schema-less |
| Scaling | Excellent horizontal scaling via partitioning |

**Strengths ✅**
- O(1) lookups — sub-millisecond latency
- Extremely simple to use
- Near-linear horizontal scalability

**Weaknesses ❌**
- Can only query by exact key — no value-based queries
- No relationships or joins
- No aggregations

**Popular Options**

| DB | Best For | Notable Feature |
|---|---|---|
| Redis | Caching, sessions, leaderboards | In-memory, rich data types, pub/sub |
| DynamoDB | Serverless, auto-scaling | Fully managed, consistent perf |
| Memcached | Simple caching | Pure distributed memory cache |
| etcd | Config, service discovery | Strong consistency (used by Kubernetes) |

**Choose when:** caching, session storage, user preferences, shopping carts, counters — any time you always know the exact key.

---

### 2.3 Document Databases

Store data as **semi-structured documents** (JSON/BSON). Each document is self-contained and can have a different structure.

| Characteristic | Detail |
|---|---|
| Data Model | JSON-like documents with nested structures |
| Query Language | Rich queries on document fields |
| Consistency | Configurable — often eventual for distributed setups |
| Schema | Schema-flexible (schema-on-read) |
| Scaling | Horizontal via sharding |

**Strengths ✅**
- Flexible schema — add fields without migrations
- Natural mapping to objects in code (no ORM impedance mismatch)
- Embed related data in one document — avoids joins
- Great developer productivity

**Weaknesses ❌**
- No cross-document joins — requires multiple round trips or denormalization
- Data duplication risk from embedding
- Flexibility without discipline → schema chaos

**Popular Options**

| DB | Best For | Notable Feature |
|---|---|---|
| MongoDB | General purpose, flexible schemas | Aggregation pipeline, change streams |
| CouchDB | Offline-first apps | Multi-master replication |
| Firestore | Mobile/web apps | Real-time sync, offline support |
| Amazon DocumentDB | MongoDB-compatible on AWS | Managed service |

**Choose when:** hierarchical/nested data, frequently evolving schema, content management, user profiles, rapid prototyping.

---

### 2.4 Wide-Column Stores

Organize data **by columns** rather than rows. Built for massive write throughput and petabyte-scale data.

| Characteristic | Detail |
|---|---|
| Data Model | Rows with dynamic columns; column families |
| Query Language | CQL (Cassandra), HBase API |
| Consistency | Tunable — eventual to strong |
| Schema | Flexible columns within column families |
| Scaling | Exceptional horizontal scaling — built for it |

**Strengths ✅**
- Petabytes of data across thousands of nodes
- High write throughput — append-only, no read-before-write
- Different rows can have different columns
- Multi-datacenter replication built-in

**Weaknesses ❌**
- Queries must align with primary key design — limited flexibility
- No joins — must denormalize
- Operationally complex at scale
- Eventual consistency — reads may be stale

**Popular Options**

| DB | Best For | Notable Feature |
|---|---|---|
| Apache Cassandra | High availability, write-heavy | Peer-to-peer, no single point of failure |
| Apache HBase | Hadoop integration, analytics | Strong consistency, HDFS backend |
| ScyllaDB | Cassandra-compatible, lower latency | C++ implementation |
| Google Bigtable | Google-scale workloads | Managed, inspiration for HBase |

**Choose when:** billions of rows, high write throughput, time-series/event logging/audit trails, multi-datacenter, IoT ingestion.

> **Interview tip:** If the interviewer says "millions of writes per second" or "multi-region availability" → think wide-column (Cassandra).

---

### 2.5 Graph Databases

Model data as **nodes (entities) and edges (relationships)**. Optimized for traversing connections.

| Characteristic | Detail |
|---|---|
| Data Model | Nodes + Edges with properties |
| Query Language | Cypher, Gremlin, SPARQL |
| Consistency | Typically ACID within a single graph |
| Schema | Flexible node and edge types |
| Scaling | Challenging — graph partitioning is hard |

**Strengths ✅**
- Relationships are first-class citizens — not just foreign keys
- Multi-hop traversals are fast (vs. expensive SQL joins)
- Intuitive modeling for connected data
- Pattern matching across relationships

**Weaknesses ❌**
- Overkill for simple CRUD without deep relationships
- Hard to scale horizontally (graph partitioning is an open problem)
- Different query languages to learn

**Popular Options**

| DB | Best For |
|---|---|
| Neo4j | General graph — mature ecosystem, Cypher |
| Amazon Neptune | AWS managed — SPARQL + Gremlin |
| TigerGraph | Large-scale graph analytics |
| ArangoDB | Multi-model (graph + document) |

**Choose when:** social networks (friends/followers), recommendation engines, fraud detection, knowledge graphs, network dependency analysis.

---

### 2.6 Specialized Databases

#### Time-Series Databases
Optimized for **timestamped data points** — metrics, IoT, monitoring, financials.

| DB | Best For |
|---|---|
| InfluxDB | Monitoring, metrics, IoT |
| TimescaleDB | PostgreSQL-compatible time-series |
| Prometheus | Kubernetes metrics + alerting |
| QuestDB | High-performance financial data |

**Use when:** storing metrics, sensor readings, stock prices, or any time-ordered data needing time-based aggregations.

#### Search Engines
Optimized for **full-text search, relevance ranking, faceted navigation**.

| DB | Best For |
|---|---|
| Elasticsearch | Log analysis, full-text search |
| Apache Solr | Search infrastructure |
| Meilisearch | Fast, developer-friendly search |

**Use when:** users search through text, you need faceted filtering, or log aggregation/analysis.

#### Vector Databases
Store and search **high-dimensional vectors** — essential for AI/ML applications.

| DB | Best For |
|---|---|
| Pinecone | Managed vector search, serverless |
| Milvus | Open-source, large-scale |
| Weaviate | Semantic search |
| Qdrant | High-performance with filtering |

**Use when:** semantic search, recommendation systems, image similarity, RAG (Retrieval-Augmented Generation).

---

### 2.7 Quick Decision Table

| DB Type | Best For | Avoid When |
|---|---|---|
| **Relational** | ACID transactions, complex queries, structured data | Massive scale, schema flexibility needed |
| **Key-Value** | Caching, sessions, simple lookups | Complex queries or relationships needed |
| **Document** | Flexible schemas, nested data, rapid dev | Heavy joins, strict consistency |
| **Wide-Column** | Massive scale, high writes, time-series | Complex queries, strong consistency |
| **Graph** | Relationship traversals, social networks | Simple CRUD, massive horizontal scale |
| **Time-Series** | Metrics, IoT, monitoring | General-purpose storage |
| **Search** | Full-text search, log analysis | Primary data store |
| **Vector** | Semantic search, AI embeddings | Traditional relational queries |

### Polyglot Persistence
Modern systems use **multiple database types** — each for what it does best:
```
User data        → PostgreSQL (ACID, structured)
Sessions/cache   → Redis (fast key-value)
Product catalog  → MongoDB (flexible schema)
Activity logs    → Cassandra (high write throughput)
Search           → Elasticsearch (full-text)
Recommendations  → Neo4j (graph traversal)
```

---

## 3. SQL vs NoSQL

### Core Differences

| Dimension | SQL (Relational) | NoSQL |
|---|---|---|
| **Data Model** | Tables — rows and columns with fixed schema | Varies: documents, key-value, wide-column, graph |
| **Schema** | Schema-on-write — strict, enforced upfront | Schema-on-read — flexible, evolves freely |
| **Scalability** | Vertical primarily; horizontal via sharding (complex) | Horizontal by design — add nodes easily |
| **Query Language** | SQL — powerful, standardized, expressive | DB-specific APIs — simpler but less powerful |
| **Transactions** | Full ACID support | Varies — often eventual consistency; some support ACID |
| **Consistency** | Strong consistency by default | Often eventual consistency (BASE model) |
| **Joins** | Native multi-table joins | Generally no joins — denormalization required |
| **Maturity** | Decades of tooling, best practices | Newer, rapidly evolving |

### The BASE Model (NoSQL counterpart to ACID)
Many NoSQL databases follow **BASE** instead of ACID:

| | Meaning |
|---|---|
| **B**asically Available | System is available even during failures (may return stale data) |
| **S**oft State | State may change over time even without new input (replicas syncing) |
| **E**ventually Consistent | Given enough time with no updates, all replicas will converge |

### When to Choose SQL
- Data has clear relationships needing joins
- ACID transactions required (payments, inventory, bookings)
- Schema is stable and well-understood
- Complex reporting/analytics queries
- Regulatory compliance needing strong consistency
- Team has strong SQL expertise

### When to Choose NoSQL
- Massive scale beyond what a single RDBMS can handle
- Schema evolves frequently (agile, early-stage products)
- Specialized access patterns (graph traversal, full-text search, time-series)
- High write throughput is critical
- Hierarchical or nested data that maps poorly to tables
- You're okay with eventual consistency

### The Real Answer (Interview)
It's rarely SQL **vs** NoSQL — most production systems use **both**:
- SQL for core transactional data (users, orders, payments)
- NoSQL for scale-out concerns (caching, logs, search, recommendations)

---

## 4. ACID Transactions

**ACID** = the four properties that guarantee reliable database transactions. Even if the system crashes, network fails, or multiple users write concurrently — the data stays correct.

A **transaction** is a group of operations treated as a single unit:
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- debit Alice
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- credit Bob
COMMIT;
```
Either both happen, or neither does.

---

### A — Atomicity

> **"All or nothing"** — a transaction either completes fully or has no effect at all.

If any step in a transaction fails, the entire transaction is **rolled back** — the DB is left exactly as it was before.

**Example:**
```
Transfer $500 from Alice → Bob
Step 1: Debit Alice  ✅
Step 2: Credit Bob   ❌ (system crash)

Without atomicity: Alice loses $500, Bob gets nothing. ← data corruption
With atomicity:    Both operations rolled back → no money lost
```

**How DBs implement it:**
- **Write-Ahead Log (WAL)** — every change is logged before being applied
- On crash, DB replays the log to complete or undo the transaction
- **Undo log** — stores the old values so rollback is possible

---

### C — Consistency

> **"Data must always be valid"** — a transaction takes the DB from one valid state to another, never leaving it in a broken intermediate state.

Consistency is enforced by rules you define: constraints, foreign keys, triggers, cascades.

**Examples of consistency rules:**
- Account balance must never go below 0
- Every order must reference a valid user (foreign key)
- A seat on a flight can only be booked once (unique constraint)

**How DBs implement it:**
- Constraints checked before COMMIT
- If any constraint fails → transaction is rejected
- Application-level rules enforced in transaction logic

> Note: In ACID, Consistency is the one property the *application* is partly responsible for — the DB enforces structural rules, but business rules must be coded correctly.

---

### I — Isolation

> **"Concurrent transactions don't interfere with each other"** — each transaction sees a consistent snapshot, as if it ran alone.

Without isolation, concurrent transactions can cause **anomalies**:

#### Concurrency Anomalies

| Anomaly | What Happens | Example |
|---|---|---|
| **Dirty Read** | Read uncommitted data from another transaction | See a transfer that hasn't committed yet |
| **Non-Repeatable Read** | Same row read twice returns different values (another TX committed between reads) | Balance reads as $100, then $50 in same TX |
| **Phantom Read** | A query returns different rows on second run (another TX inserted/deleted rows) | Count of active users changes mid-transaction |
| **Lost Update** | Two TXs read-modify-write same row; one overwrites the other's change | Both users update profile → one update is lost |

#### Isolation Levels (weakest → strongest)

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | Fastest |
| **Read Committed** | ❌ Prevented | ✅ Possible | ✅ Possible | Fast |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | ✅ Possible | Moderate |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Slowest |

**Default levels in common DBs:**
- PostgreSQL → Read Committed
- MySQL InnoDB → Repeatable Read
- SQL Server → Read Committed
- Oracle → Read Committed

**How DBs enforce isolation:**
- **MVCC (Multi-Version Concurrency Control)** — each transaction sees a snapshot of the DB at a point in time; writers don't block readers
- **Locking** — shared locks for reads, exclusive locks for writes
- **Serializable Snapshot Isolation (SSI)** — PostgreSQL's modern approach — detects conflicts and aborts conflicting transactions

---

### D — Durability

> **"Committed data is never lost"** — once a transaction commits, it survives crashes, power failures, and restarts.

**Example:**
```
COMMIT; ← user sees success
*server crashes 1 second later*
On restart → all committed data is still there
```

**How DBs implement it:**
- **Write-Ahead Log (WAL)** — changes written to durable log on disk before being applied to data files
- On crash → replay WAL to recover all committed transactions
- **Fsync** — forces OS to flush write buffers to physical disk (not just OS cache)
- **Replication** — copy data to multiple servers; if primary dies, replica takes over with all committed data

**Durability vs Performance tradeoff:**
- Fsync is slow — some systems allow disabling it for speed (risky — data loss on crash)
- Asynchronous replication is faster but replica may lag slightly behind primary

---

### ACID Summary

| Property | Guarantee | Mechanism |
|---|---|---|
| **Atomicity** | All or nothing | WAL + undo log + rollback |
| **Consistency** | Valid state only | Constraints + application logic |
| **Isolation** | No concurrency anomalies | MVCC + locking + isolation levels |
| **Durability** | Committed data survives crashes | WAL + fsync + replication |

### ACID in NoSQL
- Most NoSQL DBs sacrifice some ACID properties for scale/speed
- **MongoDB** (v4+): multi-document ACID transactions supported
- **DynamoDB**: transactions supported but limited scope
- **Cassandra**: lightweight transactions (LWT) — limited
- **Redis**: atomic single-command operations; transactions via MULTI/EXEC (no rollback)

---

## 5. Interview Q&A

**Q: When would you choose MongoDB over PostgreSQL?**
→ MongoDB when schema evolves frequently, data is naturally nested/hierarchical, or you need to scale horizontally fast. PostgreSQL when you need ACID transactions, complex joins, or strong consistency.

**Q: What is the difference between Cassandra and MongoDB?**
→ Cassandra is a wide-column store optimized for massive write throughput and multi-datacenter deployments (eventual consistency). MongoDB is a document store optimized for flexible schemas and rich queries (configurable consistency). Very different use cases.

**Q: What does "eventual consistency" mean?**
→ Given enough time with no new writes, all replicas will converge to the same value. Reads immediately after a write may return stale data. Acceptable for many use cases (social feeds, product views) but not for financial transactions.

**Q: Explain the difference between Repeatable Read and Serializable isolation.**
→ Repeatable Read prevents dirty reads and non-repeatable reads but still allows phantom reads (new rows appearing). Serializable prevents all anomalies including phantoms — but with higher lock contention and lower throughput.

**Q: What is MVCC and why is it important?**
→ Multi-Version Concurrency Control maintains multiple versions of data simultaneously. Readers see a consistent snapshot without blocking writers; writers don't block readers. This is how PostgreSQL achieves high concurrency without excessive locking.

**Q: What happens if only half of an ACID transaction is written before a crash?**
→ The Write-Ahead Log records the intended changes before they're applied. On restart, the DB checks the WAL — incomplete transactions are rolled back, complete ones are replayed. No partial data is left.

**Q: What is polyglot persistence?**
→ Using multiple database types in one system, each for what it does best. E.g., PostgreSQL for user data, Redis for caching, Elasticsearch for search, Cassandra for event logs.

**Q: Can NoSQL databases be consistent?**
→ Yes — consistency is a spectrum. DynamoDB and MongoDB support strong consistency configurations. "NoSQL = eventually consistent" is a misconception; it's a tradeoff NoSQL *allows* you to make for scale, not a requirement.

---

## 6. Quick Reference — One-Liners

| Topic | One-Liner |
|---|---|
| **Relational DB** | Tables + SQL + ACID — the workhorse for structured, transactional data |
| **Key-Value Store** | Hash map at scale — O(1) lookups, great for caching and sessions |
| **Document DB** | JSON documents — flexible schema, nested data, avoid joins |
| **Wide-Column** | Column-family storage — petabyte scale, millions of writes/sec |
| **Graph DB** | Nodes + edges — fast relationship traversal for social/fraud/recommendations |
| **Time-Series DB** | Optimized for timestamped metrics — monitoring, IoT, finance |
| **Vector DB** | Store + search AI embeddings — semantic search, RAG pipelines |
| **SQL** | Structured, joins, strong consistency, vertical scale |
| **NoSQL** | Flexible, horizontally scalable, eventual consistency, specialized |
| **Atomicity** | All operations in a TX succeed, or none of them do |
| **Consistency** | Every transaction leaves the DB in a valid state |
| **Isolation** | Concurrent transactions don't see each other's intermediate state |
| **Durability** | Committed data survives any crash or failure |
| **MVCC** | Readers and writers don't block each other — multiple data versions in flight |
| **WAL** | Changes logged to disk before applied — enables crash recovery |
| **Polyglot Persistence** | Use multiple DB types in one system — right tool for each job |

---

*Source: AlgoMaster System Design Course — algomaster.io*