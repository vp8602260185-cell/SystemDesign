# System Design: JWT & Session vs Token Auth


## Part 1: JWT (JSON Web Token)

### What is a JWT?

A **compact, self-contained, signed token** used to securely transmit user identity and claims between parties. The server issues it on login; the client sends it on every request. No server-side session storage needed.

---

### Anatomy of a JWT

A JWT is three Base64URL-encoded parts separated by dots:

```
xxxxx.yyyyy.zzzzz
  │      │      │
Header  Payload  Signature
```

| Part | Contains | Example |
|---|---|---|
| **Header** | Algorithm + token type | `{ "alg": "HS256", "typ": "JWT" }` |
| **Payload** | Claims (user data + metadata) | `{ "sub": "101", "role": "admin", "exp": 1720000000 }` |
| **Signature** | HMAC or RSA sign of header + payload | Prevents tampering |

### Standard Payload Claims

| Claim | Meaning |
|---|---|
| `sub` | Subject — user ID |
| `iat` | Issued At — when token was created |
| `exp` | Expiration — when token becomes invalid |
| `nbf` | Not Before — token invalid before this time |
| `iss` | Issuer — who issued the token |
| `aud` | Audience — who the token is intended for |
| `jti` | JWT ID — unique token identifier |

> Add custom claims like `role`, `email`, `permissions` as needed.

---

### How JWT Works

```
1. User logs in with credentials
2. Server validates → issues signed JWT
3. Client stores JWT (HttpOnly cookie or memory)
4. Client sends: Authorization: Bearer <token>
5. Server verifies signature + checks exp claim
6. ✅ Valid → serve request | ❌ Invalid/expired → 401
```

> **Key insight:** The server never stores the token — it only needs the **secret key** to verify the signature. This makes JWT stateless and horizontally scalable.

---

### Security Best Practices

| Practice | Why |
|---|---|
| Short expiry (15–60 min) | Limits damage if token is stolen |
| Use refresh tokens | Allows long sessions without long-lived access tokens |
| Store in HttpOnly cookie | Protects from XSS (JS can't read it) |
| Always use HTTPS | Prevents token interception in transit |
| Never put sensitive data in payload | Payload is Base64-encoded, not encrypted — anyone can decode it |
| Use strong signing algorithm | Prefer RS256 (asymmetric) over HS256 for distributed systems |
| Maintain a token blocklist for revocation | JWTs can't be revoked by default — blocklist invalidated tokens |

---

## Part 2: Session vs Token Authentication

### The Core Problem

HTTP is **stateless** — each request is independent, the server has no memory of previous ones. After login, we need a way to remember, identify, and eventually expire a user's authenticated state across requests.

---

### Session-Based Auth (Stateful)

**How it works:**
1. User logs in → server creates a session object and stores it (Redis/DB)
2. Server sends back a **Session ID** in a cookie
3. Browser auto-sends cookie on every request
4. Server **looks up** Session ID → identifies user

```
Client                  Server                 Session Store
  │── POST /login ──────►│                          │
  │◄── Set-Cookie: sid ──│── STORE session ────────►│
  │                      │                          │
  │── GET /dashboard ───►│── LOOKUP sid ───────────►│
  │◄── 200 OK ───────────│◄── session data ─────────│
```

**Session Storage Options:**

| Storage | Pros | Cons |
|---|---|---|
| **In-Memory** | Fastest | Lost on restart, no horizontal scale |
| **Database** | Persistent | Slow, extra DB load |
| **Redis** *(recommended)* | Fast, scalable, built-in expiry | Extra infra |
| **File System** | Simple | Slow, no scale |

**Cookie Security Attributes:**

```
Set-Cookie: sid=abc123;
  HttpOnly;           ← JS can't read it (XSS protection)
  Secure;             ← HTTPS only
  SameSite=Strict;    ← CSRF protection
  Max-Age=3600
```

---

### Token-Based Auth (Stateless)

**How it works:**
1. User logs in → server issues a **signed JWT**
2. Client stores token; sends it in `Authorization` header
3. Server **verifies signature** — no lookup needed

```
Client                  Server
  │── POST /login ──────►│
  │◄── { token: JWT } ───│  (nothing stored server-side)
  │                      │
  │── GET /dashboard ───►│
  │   Authorization:      │── verify signature ✅
  │   Bearer <JWT>        │── read claims from token
  │◄── 200 OK ───────────│
```

**Token Storage Options:**

| Storage | XSS Risk | CSRF Risk | Notes |
|---|---|---|---|
| `localStorage` | ❌ High | None | Avoid for auth tokens |
| `sessionStorage` | ❌ High | None | Cleared on tab close |
| Memory (JS variable) | ✅ Low | None | Lost on refresh |
| **HttpOnly Cookie** | ✅ None | Medium | Best security; needs CSRF header |

---

### Head-to-Head Comparison

| Aspect | Session-Based | Token-Based |
|---|---|---|
| **State** | Stateful (server stores sessions) | Stateless (server stores nothing) |
| **Scalability** | Requires shared session store (Redis) | Any server validates any token |
| **Revocation** | ✅ Instant — delete session from store | ❌ Hard — need blocklist or wait for expiry |
| **Performance** | Network call to session store (~1–5ms) | CPU signature verify (<1ms) |
| **Cross-domain** | ❌ Cookie is domain-bound | ✅ Token works across any domain |
| **Mobile/SPA** | Awkward | Natural fit |
| **Active session tracking** | ✅ Easy — see all sessions, force logout | ❌ Difficult without extra storage |
| **Payload size** | Small cookie (~50 bytes) | Larger token (500+ bytes) |

---

### When to Use Which

**Use Session-Based when:**
- Traditional server-rendered web app (Django, Rails, Spring MVC)
- Need instant revocation (admin banning a user, forced logout)
- Need active session management ("logged in from 3 devices")
- Single-domain application — cookies work seamlessly

**Use Token-Based (JWT) when:**
- SPA (React, Vue, Angular) with separate API backend
- Mobile applications — no native cookie handling
- Microservices — services validate tokens independently
- Cross-domain APIs serving multiple clients
- Serverless / containerised — no persistent state between invocations
- Horizontal scaling without session sync overhead

---

### Hybrid Approach *(Production Best Practice)*

Most production systems combine both:

```
Access Token  → Short-lived JWT (15 min) — stateless, fast
Refresh Token → Long-lived (7–30 days) — stored in DB, revocable

Flow:
1. Login → issue access token + refresh token
2. Client uses access token for API calls
3. Access token expires → client sends refresh token
4. Server validates refresh token in DB → issues new access token
5. Logout → delete refresh token from DB (instant revocation) ✅
```

This gets you **stateless speed** for most requests + **revocation capability** when needed.

