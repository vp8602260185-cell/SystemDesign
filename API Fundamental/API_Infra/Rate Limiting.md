# System Design: Rate Limiting


## What is Rate Limiting?

Protects services from being overwhelmed by too many requests from a single user or client. Ensures fair usage, guards against DoS/DDoS, and keeps backend systems stable.

---

## 5 Rate Limiting Algorithms

---

### 1. Token Bucket *(Most Popular)*

**How it works:**
- A bucket holds tokens up to a max capacity
- Tokens are added at a fixed rate (e.g. 10/sec)
- Each request consumes one token; if no tokens → request is dropped

```
[Bucket: ████████░░] capacity=10, tokens=8
Request arrives → consumes 1 token ✅
Bucket empty → request dropped ❌
```

**Java Implementation:**

```java
import java.util.concurrent.atomic.AtomicLong;

public class TokenBucket {
    private final long capacity;          // max tokens in bucket
    private final double refillRate;      // tokens added per second
    private AtomicLong tokens;
    private long lastRefillTime;

    public TokenBucket(long capacity, double refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = new AtomicLong(capacity);
        this.lastRefillTime = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        refill();
        if (tokens.get() > 0) {
            tokens.decrementAndGet();
            return true;   // request allowed
        }
        return false;      // request dropped
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double elapsedSeconds = (now - lastRefillTime) / 1000.0;
        long newTokens = (long) (elapsedSeconds * refillRate);
        if (newTokens > 0) {
            tokens.set(Math.min(capacity, tokens.get() + newTokens));
            lastRefillTime = now;
        }
    }

    // Usage
    public static void main(String[] args) {
        TokenBucket limiter = new TokenBucket(10, 2); // cap=10, 2 tokens/sec
        for (int i = 0; i < 15; i++) {
            System.out.println("Request " + i + ": " +
                (limiter.allowRequest() ? "✅ ALLOWED" : "❌ DENIED"));
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to implement | Memory scales with number of users |
| Allows short bursts up to bucket capacity | Traffic rate not perfectly smooth |

---

### 2. Leaky Bucket

**How it works:**
- Requests enter a queue (the "bucket") from the top
- Requests are **processed at a constant rate** (leak through the bottom)
- If queue is full → new requests are discarded immediately

```
Requests IN ↓↓↓
[Bucket queue  ]
         ↓ constant drip out → processed at fixed rate
Full? → new requests discarded ❌
```

**Java Implementation:**

```java
import java.util.LinkedList;
import java.util.Queue;

public class LeakyBucket {
    private final int capacity;           // max queue size
    private final long leakRateMs;        // process one request every N ms
    private final Queue<Long> queue;
    private long lastLeakTime;

    public LeakyBucket(int capacity, long leakRateMs) {
        this.capacity = capacity;
        this.leakRateMs = leakRateMs;
        this.queue = new LinkedList<>();
        this.lastLeakTime = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        leak(); // process pending requests first
        if (queue.size() < capacity) {
            queue.add(System.currentTimeMillis());
            return true;   // request queued ✅
        }
        return false;      // bucket full, request dropped ❌
    }

    private void leak() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastLeakTime;
        long requestsToLeak = elapsed / leakRateMs;
        for (int i = 0; i < requestsToLeak && !queue.isEmpty(); i++) {
            queue.poll(); // process (leak) one request
        }
        if (requestsToLeak > 0) {
            lastLeakTime = now;
        }
    }

    // Usage
    public static void main(String[] args) {
        LeakyBucket limiter = new LeakyBucket(5, 500); // cap=5, leak every 500ms
        for (int i = 0; i < 8; i++) {
            System.out.println("Request " + i + ": " +
                (limiter.allowRequest() ? "✅ ALLOWED" : "❌ DENIED"));
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Smooth, predictable output rate | Bursts dropped immediately — no flexibility |
| Prevents sudden spikes from hitting backend | Slightly more complex than Token Bucket |

---

### 3. Fixed Window Counter

**How it works:**
- Time is divided into fixed intervals (e.g. 1-minute windows)
- A counter resets to zero at the start of each window
- Requests increment the counter; if it exceeds the limit → deny until next window

```
Window:  [00:00 ───────── 01:00]  limit=100
Counter: 0 → 1 → 2 ... → 100 → DENY ❌
New window starts → counter resets to 0
```

**Java Implementation:**

```java
import java.util.concurrent.atomic.AtomicInteger;

public class FixedWindowCounter {
    private final int limit;              // max requests per window
    private final long windowSizeMs;      // window duration in ms
    private AtomicInteger counter;
    private long windowStart;

    public FixedWindowCounter(int limit, long windowSizeMs) {
        this.limit = limit;
        this.windowSizeMs = windowSizeMs;
        this.counter = new AtomicInteger(0);
        this.windowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();

        // Reset counter if window has passed
        if (now - windowStart >= windowSizeMs) {
            counter.set(0);
            windowStart = now;
        }

        if (counter.get() < limit) {
            counter.incrementAndGet();
            return true;   // ✅ allowed
        }
        return false;      // ❌ limit exceeded
    }

    // Usage
    public static void main(String[] args) {
        FixedWindowCounter limiter = new FixedWindowCounter(5, 60_000); // 5 req/min
        for (int i = 0; i < 8; i++) {
            System.out.println("Request " + i + ": " +
                (limiter.allowRequest() ? "✅ ALLOWED" : "❌ DENIED"));
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Easiest to implement | **Boundary burst problem** — a client can fire 100 req at 00:59 and 100 more at 01:00, getting 2× the limit in ~2 seconds |

---

### 4. Sliding Window Log

**How it works:**
- Store a timestamp log for every request
- On each new request: remove timestamps older than the window size, count remaining
- If count < limit → allow and log timestamp; else → deny

```
Window = 60s, Limit = 100
Log: [t-58s, t-45s, t-12s, t-3s, ...]  (53 entries)
New request → count=54 < 100 → ✅ allow, add timestamp
```

**Java Implementation:**

```java
import java.util.LinkedList;
import java.util.Deque;

public class SlidingWindowLog {
    private final int limit;              // max requests per window
    private final long windowSizeMs;      // window duration in ms
    private final Deque<Long> requestLog; // timestamps of allowed requests

    public SlidingWindowLog(int limit, long windowSizeMs) {
        this.limit = limit;
        this.windowSizeMs = windowSizeMs;
        this.requestLog = new LinkedList<>();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSizeMs;

        // Remove timestamps outside the current window
        while (!requestLog.isEmpty() && requestLog.peekFirst() <= windowStart) {
            requestLog.pollFirst();
        }

        if (requestLog.size() < limit) {
            requestLog.addLast(now);
            return true;   // ✅ allowed
        }
        return false;      // ❌ limit exceeded
    }

    // Usage
    public static void main(String[] args) {
        SlidingWindowLog limiter = new SlidingWindowLog(5, 60_000); // 5 req/min
        for (int i = 0; i < 8; i++) {
            System.out.println("Request " + i + ": " +
                (limiter.allowRequest() ? "✅ ALLOWED" : "❌ DENIED"));
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Most accurate — no boundary edge cases | Memory-intensive at high volume (stores every timestamp) |
| Works well for low-volume APIs | Searching/cleaning old timestamps adds overhead |

---

### 5. Sliding Window Counter *(Best Balance)*

**How it works:**
- Tracks request counts for **current** and **previous** windows
- Calculates a weighted sum based on how far into the current window we are:

```
weight = (% remaining in prev window) × prevWindowCount + currentWindowCount

Example: 75% into current window →
weight = 25% × prevCount + currentCount
```
- If `weight + 1 > limit` → deny request

**Java Implementation:**

```java
public class SlidingWindowCounter {
    private final int limit;              // max requests per window
    private final long windowSizeMs;      // window duration in ms
    private int prevWindowCount;
    private int currentWindowCount;
    private long currentWindowStart;

    public SlidingWindowCounter(int limit, long windowSizeMs) {
        this.limit = limit;
        this.windowSizeMs = windowSizeMs;
        this.prevWindowCount = 0;
        this.currentWindowCount = 0;
        this.currentWindowStart = System.currentTimeMillis();
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        long elapsed = now - currentWindowStart;

        // Slide the window forward if needed
        if (elapsed >= windowSizeMs) {
            prevWindowCount = currentWindowCount;
            currentWindowCount = 0;
            currentWindowStart = now;
            elapsed = 0;
        }

        // Weight = fraction of prev window still in range × prevCount + currentCount
        double prevWeight = 1.0 - ((double) elapsed / windowSizeMs);
        double weightedCount = (prevWeight * prevWindowCount) + currentWindowCount;

        if (weightedCount + 1 <= limit) {
            currentWindowCount++;
            return true;   // ✅ allowed
        }
        return false;      // ❌ limit exceeded
    }

    // Usage
    public static void main(String[] args) {
        SlidingWindowCounter limiter = new SlidingWindowCounter(5, 60_000); // 5 req/min
        for (int i = 0; i < 8; i++) {
            System.out.println("Request " + i + ": " +
                (limiter.allowRequest() ? "✅ ALLOWED" : "❌ DENIED"));
        }
    }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| More accurate than Fixed Window Counter | Slightly more complex to implement |
| More memory-efficient than Sliding Window Log | |
| Smooths out boundary bursts | |

---

## Algorithm Comparison

| Algorithm | Accuracy | Memory | Burst Handling | Complexity |
|---|---|---|---|---|
| **Token Bucket** | Medium | Medium | ✅ Allows bursts | Low |
| **Leaky Bucket** | High (smooth) | Low | ❌ Drops bursts | Low-Medium |
| **Fixed Window** | Low | Very Low | ❌ Boundary problem | Very Low |
| **Sliding Window Log** | ✅ Highest | ❌ High | ✅ Accurate | Medium |
| **Sliding Window Counter** | High | Low | ✅ Good | Medium |

---

## Key Takeaway

> Always communicate rate limits to API consumers via **response headers** so clients can implement retry and backoff strategies:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1720000000
Retry-After: 60        ← on 429 Too Many Requests
```

**Choose your algorithm based on:**
- **Scale + Simplicity** → Token Bucket or Fixed Window
- **Accuracy** → Sliding Window Log or Counter
- **Smooth output rate** → Leaky Bucket

