# System Design: REST API Design Best Practices

> **Topic:** REST API Design 

## 1. What is REST?

**REST (Representational State Transfer)** is an architectural style for building networked applications, introduced by Roy Fielding in his 2000 doctoral dissertation.

### Six Guiding Constraints

| Constraint | Description |
|---|---|
| **Stateless** | Every request must contain all information needed. Server stores no client session state. |
| **Client-Server** | UI and data storage are decoupled. Each can evolve independently. |
| **Uniform Interface** | Consistent resource-based URLs and standard HTTP methods across the API. |
| **Cacheable** | Responses must declare whether they are cacheable to reduce client-server interactions. |
| **Layered System** | Client doesn't know if it's talking to origin server, load balancer, or cache layer. |
| **Code on Demand** *(optional)* | Server can send executable code to client (e.g., JavaScript). |

### Why Statelessness Matters
- Enables **horizontal scaling** — any server can handle any request
- Simplifies **fault recovery** — no session to restore after a crash
- Improves **visibility** — every request is self-contained and inspectable

---

## 2. Resource Naming Conventions

Resources are the core abstraction in REST. URLs should identify **things (nouns)**, not actions (verbs).

### Use Nouns, Not Verbs

```
✅ GET  /users          → list users
✅ GET  /users/101      → get user 101
✅ POST /users          → create a user

❌ GET  /getUsers
❌ POST /createUser
❌ POST /deleteUser/101
```

### Use Plural Nouns

```
✅ /users
✅ /articles
✅ /orders

❌ /user
❌ /article
```

### Use Lowercase with Hyphens

```
✅ /blog-posts
✅ /user-profiles

❌ /blogPosts       (camelCase)
❌ /Blog_Posts      (underscores + PascalCase)
```

### Nested Resources for Relationships

Use nesting to express ownership/hierarchy, but avoid going beyond **2 levels deep**:

```
✅ GET /users/101/orders           → orders belonging to user 101
✅ GET /users/101/orders/55        → specific order of user 101

❌ GET /users/101/orders/55/items/3/reviews   → too deeply nested
```

When nesting gets deep, flatten with query params:
```
GET /reviews?userId=101&orderId=55
```

### Singleton vs Collection

```
/users          → collection
/users/101      → singleton (specific resource)
/users/101/profile  → singleton sub-resource
```

---

## 3. HTTP Methods

Each HTTP method has a defined semantic. Use them correctly and consistently.

| Method | Action | Idempotent | Safe | Body |
|---|---|---|---|---|
| `GET` | Read / fetch | ✅ Yes | ✅ Yes | No |
| `POST` | Create | ❌ No | ❌ No | Yes |
| `PUT` | Full replace/update | ✅ Yes | ❌ No | Yes |
| `PATCH` | Partial update | ✅ Yes* | ❌ No | Yes |
| `DELETE` | Delete | ✅ Yes | ❌ No | Optional |
| `HEAD` | Like GET but no body | ✅ Yes | ✅ Yes | No |
| `OPTIONS` | Discover allowed methods | ✅ Yes | ✅ Yes | No |

> **Idempotent** = calling it N times has the same effect as calling it once.  
> **Safe** = it does not modify server state.

### PUT vs PATCH

```json
// PUT /users/101 — replaces the entire resource
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}

// PATCH /users/101 — updates only specified fields
{
  "email": "newalice@example.com"
}
```

---

## 4. HTTP Status Codes

Always return the most specific, meaningful status code. Never return `200 OK` for an error.

### 2xx — Success

| Code | Name | When to Use |
|---|---|---|
| `200` | OK | Successful GET, PUT, PATCH, DELETE |
| `201` | Created | Successful POST that created a resource |
| `202` | Accepted | Request accepted for async processing |
| `204` | No Content | Successful DELETE or PUT with no response body |

### 3xx — Redirection

| Code | Name | When to Use |
|---|---|---|
| `301` | Moved Permanently | Resource has a new permanent URL |
| `304` | Not Modified | Cached version is still valid (ETag/If-None-Match) |

### 4xx — Client Errors

| Code | Name | When to Use |
|---|---|---|
| `400` | Bad Request | Malformed request, validation failed |
| `401` | Unauthorized | Missing or invalid authentication |
| `403` | Forbidden | Authenticated but not authorized |
| `404` | Not Found | Resource does not exist |
| `405` | Method Not Allowed | HTTP method not supported on this endpoint |
| `409` | Conflict | Duplicate resource, version conflict |
| `422` | Unprocessable Entity | Semantic validation errors |
| `429` | Too Many Requests | Rate limit exceeded |

### 5xx — Server Errors

| Code | Name | When to Use |
|---|---|---|
| `500` | Internal Server Error | Unexpected server-side failure |
| `502` | Bad Gateway | Upstream service returned invalid response |
| `503` | Service Unavailable | Server temporarily overloaded or down |
| `504` | Gateway Timeout | Upstream service timed out |

---

## 5. Request and Response Design

### Request Headers (Common)

| Header | Purpose |
|---|---|
| `Content-Type: application/json` | Body format being sent |
| `Accept: application/json` | Expected response format |
| `Authorization: Bearer <token>` | Auth credential |
| `If-None-Match: "<etag>"` | Conditional fetch for caching |

### Response Body Structure

Keep responses **consistent** across all endpoints. A good pattern:

```json
// Success — single resource
{
  "data": {
    "id": "101",
    "name": "Alice",
    "email": "alice@example.com"
  }
}

// Success — collection
{
  "data": [...],
  "meta": {
    "total": 250,
    "page": 1,
    "perPage": 20
  }
}

// Error
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is invalid",
    "details": [
      { "field": "email", "issue": "Must be a valid email address" }
    ]
  }
}
```

### Key Response Design Rules
- Always use **camelCase** for JSON keys (or pick one convention and stick to it)
- Use **ISO 8601** for dates: `"2024-07-15T10:30:00Z"`
- Never expose internal IDs or database keys directly if they carry security risk
- Include a `Location` header pointing to the new resource on `201 Created`

```
HTTP/1.1 201 Created
Location: /users/101
```

---

## 6. Pagination

Never return unbounded collections. Always paginate lists.

### Strategy 1: Offset-Based Pagination

```
GET /users?page=3&limit=20
GET /users?offset=40&limit=20
```

**Response:**
```json
{
  "data": [...],
  "meta": {
    "total": 250,
    "page": 3,
    "perPage": 20,
    "totalPages": 13
  }
}
```

✅ Simple to implement, supports jumping to any page  
❌ Unstable with fast-changing data (inserts/deletes shift pages)  
❌ Slow for large offsets (DB must skip N rows)

---

### Strategy 2: Cursor-Based Pagination *(Preferred at scale)*

```
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ==
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTIwfQ==",
    "hasMore": true
  }
}
```

✅ Stable — inserts/deletes don't affect results  
✅ Consistent O(1) performance regardless of depth  
❌ Cannot jump to arbitrary pages  
❌ Slightly more complex to implement

---

### Strategy 3: Keyset Pagination

Similar to cursor-based but uses actual field values as the marker:

```
GET /users?after_id=100&limit=20
GET /posts?after_created_at=2024-07-01T00:00:00Z&limit=20
```

---

## 7. Filtering, Sorting, and Searching

All of these should be expressed via **query parameters**, not path segments.

### Filtering

```
GET /users?status=active
GET /orders?status=shipped&userId=101
GET /products?minPrice=100&maxPrice=500&category=electronics
```

### Sorting

```
GET /users?sort=createdAt             → ascending (default)
GET /users?sort=-createdAt            → descending (prefix with -)
GET /users?sort=lastName,-createdAt   → multi-field sort
```

### Searching

```
GET /users?q=alice
GET /articles?search=system+design
```

### Field Selection (Sparse Fieldsets)

Let clients request only the fields they need to reduce payload:

```
GET /users?fields=id,name,email
```

---

## 8. Versioning

APIs must evolve. Versioning lets you make breaking changes without disrupting existing clients.

### What Counts as a Breaking Change?
- Removing a field or endpoint
- Renaming a field
- Changing a field's data type
- Changing required/optional status of a parameter
- Changing HTTP method or status code

### Strategy 1: URL Path Versioning *(Most Common)*

```
GET /v1/users
GET /v2/users
```
✅ Explicit, easy to route, easy to test  
❌ Pollutes URLs; hard to version sub-resources cleanly

---

### Strategy 2: Query Parameter Versioning

```
GET /users?version=2
GET /users?api-version=2024-07-01
```
✅ Keeps URLs clean  
❌ Easy to forget; harder to enforce

---

### Strategy 3: Header Versioning

```
Accept: application/vnd.myapi.v2+json
API-Version: 2
```
✅ Clean URLs  
❌ Not visible in browser; harder to test manually

---

### Versioning Best Practices
- **Maintain at least 2 versions** concurrently during migration
- **Announce deprecations** early with `Deprecation` and `Sunset` response headers
- **Never silently break** — always give clients a migration window

```
Deprecation: true
Sunset: Sat, 31 Dec 2024 23:59:59 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

---

## 9. Authentication and Authorization

### Common Authentication Mechanisms

#### API Keys
Simple, stateless, easy to generate. Sent in header or query param.
```
Authorization: ApiKey sk_live_abc123
X-API-Key: sk_live_abc123
```
✅ Simple  ❌ No expiry, hard to rotate, no user identity

---

#### JWT (JSON Web Token)
Self-contained token with claims and expiry. Most common for user-facing APIs.
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```
Structure: `header.payload.signature` (Base64 encoded, dot-separated)

✅ Stateless, verifiable without DB lookup, carries claims  
❌ Cannot be revoked without a blocklist; large if payload is big

---

#### OAuth 2.0
Standard protocol for **delegated authorization** — letting a third-party app access resources on behalf of a user.

Flows:
- **Authorization Code** — web/mobile apps (most secure)
- **Client Credentials** — machine-to-machine (service accounts)
- **Device Code** — TV/CLI apps without a browser

---

#### mTLS (Mutual TLS)
Both client and server present certificates. Used for internal service-to-service auth.

---

### Authorization Patterns

| Pattern | Description |
|---|---|
| **RBAC** (Role-Based) | Access determined by role (admin, user, editor) |
| **ABAC** (Attribute-Based) | Access determined by attributes (department, region, resource owner) |
| **Scope-Based** | OAuth scopes limit what an access token can do (`read:users`, `write:orders`) |

---

## 10. Error Handling

A great error response tells the client **what went wrong**, **why**, and ideally **how to fix it**.

### Error Response Structure

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "issue": "Must be a valid email address",
        "value": "not-an-email"
      },
      {
        "field": "age",
        "issue": "Must be at least 18",
        "value": 15
      }
    ],
    "traceId": "abc123xyz",
    "docsUrl": "https://api.example.com/errors/VALIDATION_ERROR"
  }
}
```

### Key Error Handling Principles
- Use **machine-readable error codes** (`VALIDATION_ERROR`, `RESOURCE_NOT_FOUND`) — not just messages
- Include a **traceId** / **requestId** for debugging and log correlation
- Include a **docsUrl** for complex errors with remediation guidance
- **Never expose stack traces**, internal paths, or database errors to clients
- **Log full details server-side**; return only safe, actionable info to the client
- Be consistent — use the same error envelope shape across all endpoints

---

## 11. Rate Limiting

Rate limiting protects your API from abuse, prevents runaway clients, and ensures fair usage.

### Common Rate Limiting Strategies

| Strategy | How It Works | Pros | Cons |
|---|---|---|---|
| **Fixed Window** | Count resets every fixed period (e.g., 1000 req/hour) | Simple | Burst at window boundary |
| **Sliding Window** | Rolling time window per request | Smoother | More memory |
| **Token Bucket** | Tokens added at fixed rate; request consumes one token | Allows bursts up to bucket size | Slightly complex |
| **Leaky Bucket** | Requests queued and processed at fixed rate | Smoothes traffic | Queue can overflow |

### Rate Limit Response Headers

Always communicate limits to clients via headers:

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 342
X-RateLimit-Reset: 1720000000
Retry-After: 3600
```

- Return **`429 Too Many Requests`** when limit is exceeded
- Include `Retry-After` to tell clients when to retry
- Consider separate limits for different tiers (free vs. paid) and operations (reads vs. writes)

---

## Quick Reference: API Design Checklist

```
Resource Naming
□ Use plural nouns for collections
□ Use lowercase-hyphenated URLs
□ Nesting max 2 levels deep

HTTP Methods
□ GET for reads, POST for creates
□ PUT for full replace, PATCH for partial update
□ DELETE for removal

Status Codes
□ 201 + Location header on create
□ 204 on successful delete
□ 400 for validation, 401 for auth, 403 for forbidden, 404 for missing

Request / Response
□ Consistent JSON envelope across endpoints
□ ISO 8601 dates
□ camelCase keys

Pagination
□ All list endpoints are paginated
□ Prefer cursor-based for large/fast-changing datasets

Versioning
□ Version in URL path (/v1/)
□ Deprecation + Sunset headers for old versions

Authentication
□ Bearer JWT for user APIs
□ API keys for server-to-server simple integrations
□ OAuth 2.0 for third-party delegation

Error Handling
□ Machine-readable error codes
□ traceId in every error response
□ No stack traces or internals leaked

Rate Limiting
□ 429 with Retry-After on limit exceeded
□ Rate limit headers on every response
```