# Scaling Read Traffic: Revision Notes

Source: AlgoMaster.io, System Design Interviews course (Interview Patterns > Scaling Under Load). Notes are condensed and in my own words.

---

## 1. The core idea

- Many systems are **read-heavy** (often 100:1 or more): URL shortener, social feeds, YouTube pages, e-commerce product pages, ticket booking pages.
- **Goal of every technique: serve most reads without hitting the main database.**
- Why it matters:
  - **Throughput:** one DB handles only a few thousand to a few tens of thousands of simple queries/sec.
  - **Latency:** user-facing reads need tens of ms; queued queries miss that.
  - **Cost:** DB capacity is expensive; memory/CDN is cheap per request.

Example numbers (URL shortener): 1B redirects/day, about 11,500 reads/sec average, about 35,000/sec at peak (3x).

## 2. Step 0: use the database efficiently first

| Problem | Fix |
|---|---|
| Full table scan | Add an index on the lookup column |
| N+1 queries | JOIN, or batch fetch with `IN (...)` |
| Large result set | `LIMIT` and pagination |
| Expensive aggregation on every read | Precompute it |

- Composite index: filter columns first, then sort columns.
- Use `EXPLAIN ANALYZE` to find bad queries.
- N+1 example: 20 posts + 1 query per author = 21 queries instead of 2.
- In an interview, just mention it briefly and move on.

## 3. Read replicas

- Replica = synced copy of the **primary**. Writes go to primary, reads spread across replicas.
- Roughly: 1 DB at 10k reads/sec + 4 replicas is about 50k reads/sec.
- Limits:
  - Not perfectly linear (each replica applies every write, adds storage and ops cost).
  - **Does not fix a hot single row**: every replica gets the same load for a popular item. That needs caching.
- Routing: in the app, or via a DB proxy (e.g. ProxySQL).

## 4. Replication lag

- Replication is usually **asynchronous**: the primary confirms the write without waiting for replicas. Lag is usually ms, but can reach seconds under load.
- Synchronous replication removes most lag but raises write latency and lets a slow replica block writes.
- **Read-your-own-writes problem:** user updates a profile picture, page reloads, read hits a stale replica, user sees the old picture and thinks it failed.

| Fix | How | Trade-off |
|---|---|---|
| Primary after a write | Send that user's reads to primary for a few seconds | Some extra primary load |
| Own data from primary | User's profile, settings, orders always from primary | More primary load |
| Lag monitoring | Pull lagging replicas out of rotation | Fewer replicas under load |

- Interview tip: ask which reads must be fresh and which can be a second or two stale.

## 4b. Caching: the first layer to add

- Cache = in-memory store (Redis, Memcached), often under 1 ms.
- Replicas still run a full query per read; a cache avoids the query.

**Cache-aside (most common):**
1. App checks cache.
2. Hit: return it.
3. Miss: read from DB, store in cache (with TTL), return it.

**Cache hit ratio (the number that matters):**
- 35,000 reads/sec at 95% hit = **1,750/sec** to DB.
- At 90% hit = 3,500/sec (twice as much).
- State the expected hit ratio in an interview, not just "add a cache".
- Works best when a small set of data gets most of the traffic.

| Good to cache | Poor to cache |
|---|---|
| Short link to long URL mappings | Bank balances |
| Product details | Exact checkout prices |
| Posts, profiles | One-time auth tokens |
| Config data | Rarely read data |

## 5. Keeping the cache fresh

- Problem: data lives in two places, so the cache can go stale.
- **TTL:** entries expire (e.g. 5 min). Simple, but stale up to the TTL.
- **Delete on write:** update DB, then delete the cache entry. Next read reloads. Safer than writing the new value into the cache (racing updates can leave an old value there).
- **Use both:** delete on write + TTL as a safety net if a delete is missed.

| Strategy | Staleness | Write cost | Complexity | Good fit |
|---|---|---|---|---|
| TTL only | Up to TTL | None | Low | Minutes of staleness OK |
| Delete on write + TTL | Very short | One extra delete | Low | Most apps |
| Write-through | Very short | Higher | Medium | Small read models that must stay fresh |
| Event-based (e.g. Kafka) | Short | Low for writer | High | Many services caching same data |

- **Write-through:** cache updated in the same write path; slower writes, and you must handle DB success + cache failure.
- **Event-based:** a change event is published and each service deletes its own copy.

## 6. Cache stampede

- A very popular entry expires, so thousands of requests miss at once and all hit the DB with the same query.

| Technique | How | Trade-off |
|---|---|---|
| Lock on rebuild | One request reloads, others wait or get the old value | Slight added latency |
| Background refresh | Reload popular entries before expiry | Wasted work on entries no longer read |
| TTL jitter | Random variation so entries do not expire together | Does not fix a single hot key |

## 7. Hot keys

- In a distributed cache each key lives on **one server**. A celebrity post makes that server the bottleneck, and adding servers does not help.
- Fixes:
  - **Key copies:** `post:123:copy1`, `copy2`, `copy3`; pick one at random per read; delete all copies on write.
  - **Local in-memory cache on each app server** for very popular data, with a very short TTL (a few seconds). Slightly stale.
- **Negative caching:** cache "not found" for a short time so missing keys (deleted post, fake short code) do not keep hitting the DB.

## 8. CDN and edge caching

- CDN stores copies on servers near users. Good for images, video, JS, CSS, and also **public pages and API responses** that are the same for everyone.
- Cuts load on your servers and improves latency for distant users.
- Browser caching is controlled by `Cache-Control`; the same header guides the CDN.

| Content | Example header | Meaning |
|---|---|---|
| Versioned static files | `public, max-age=31536000, immutable` | Cache for a year; new version = new URL |
| Public page/article | `public, max-age=60, s-maxage=300` | Browser 60 s, CDN 300 s |
| User-specific data | `private, no-store` | Do not cache in CDN or browser |

- `s-maxage` applies only to shared caches (CDN).
- `stale-while-revalidate`: serve the old copy briefly while fetching a fresh one in the background.
- **Risk:** caching personal data. Mark it `private`, or split into a public cacheable part and a small user-specific part.

## 9. Precompute results (do the work at write time)

- **Fanout on write:** when someone posts, push it into each follower's precomputed feed. Reading a feed becomes a single list read.
- Celebrity accounts: writing to millions of feeds is too slow, so use a **hybrid** (normal users fan out on write; big accounts merged in at read time).
- **Counters:** keep a like count and update on each like, instead of counting on every view.
- **Materialized views / denormalized tables:** store the result of an expensive join or aggregation.
- Trade-off: more expensive writes and more storage, but usually worth it when reads dominate.

## 10. The layered picture

Read path:
1. **CDN** (hit: return immediately)
2. **App server** local in-memory cache
3. **Distributed cache** (Redis/Memcached)
4. **Read replica** or **primary**

| Layer | Max staleness | How it updates |
|---|---|---|
| Browser | Up to `max-age` | Expires, or new versioned URL |
| CDN | Up to `s-maxage` | Expires, or explicit purge |
| App server memory | A few seconds | Short TTL |
| Distributed cache | Up to TTL (usually less) | Delete on write + TTL |
| Read replica | Replication lag (ms) | Replication stream |

- Every layer you add is another place for stale data. Always state how stale it can be and how it is updated.

## 11. Interview approach (step by step)

1. **Clarify:** read/write ratio, peak reads/sec, freshness needed, is the response the same for all users, can one item suddenly go viral?
2. **Estimate** peak read load (is one DB enough?).
3. **Add caching:** what you cache, how it stays fresh, expected hit ratio.
4. **Add read replicas** for misses; explain replication lag handling.
5. **Add a CDN** for content that is the same for everyone.
6. **Cover edge cases:** stampedes, hot keys.

- Tie every component to a specific problem you identified. That beats listing technologies.

---

## Quick-recall cheat sheet

- Goal: keep reads off the main DB.
- Order: **indexes/queries, cache, replicas, CDN, precompute, edge cases**.
- Hit ratio math: DB load = peak reads x (1 - hit ratio).
- Replicas scale reads, not hot rows. Cache fixes hot rows.
- Async replication means lag, which means the read-your-own-writes problem.
- Cache freshness: **delete on write + TTL**.
- Stampede: lock, background refresh, TTL jitter.
- Hot key: key copies, local cache, negative caching for missing keys.
- CDN works for any response that is the same for all users. User-specific data is `private`.
- Fanout on write for feeds, with a hybrid for celebrities.

---

## My Notes / Questions

-