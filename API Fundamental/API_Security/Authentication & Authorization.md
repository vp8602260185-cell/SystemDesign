# System Design: Authentication & Authorization

## The One-Line Difference

| | Question | Example |
|---|---|---|
| **Authentication (AuthN)** | *Who are you?* | Login with email + password |
| **Authorization (AuthZ)** | *What are you allowed to do?* | Only admins can delete users |

> Authentication always comes first. You can't authorize someone whose identity you haven't verified.

---

## Authentication

### Common Methods

| Method | How It Works | Best For |
|---|---|---|
| **Password** | User provides credentials; server validates against stored hash (bcrypt/Argon2) | Standard login flows |
| **API Key** | Static secret token sent in header per request | Server-to-server, developer APIs |
| **JWT** | Self-contained signed token with claims and expiry; server verifies signature — no DB lookup needed | Stateless web/mobile APIs |
| **OAuth 2.0** | Delegated auth — user grants a third-party app access to their resources without sharing password | "Login with Google/GitHub" |
| **MFA** | Combines two factors: something you know (password) + something you have (OTP/TOTP) or are (biometric) | High-security accounts |
| **SSO** | One login grants access to multiple apps via a central identity provider (IdP) e.g. Okta, Auth0 | Enterprise, internal tools |

### Typical Authentication Flow (JWT)

```
1. User sends credentials (email + password)
2. Server validates → hashes password → compares with DB
3. Server issues a signed JWT (header.payload.signature)
4. Client stores JWT (memory or HttpOnly cookie)
5. Client sends JWT in Authorization: Bearer <token> on every request
6. Server verifies signature + checks expiry → grants or denies access
```

---

## Authorization

### Common Models

| Model | How It Works | Best For |
|---|---|---|
| **RBAC** (Role-Based) | Access determined by assigned roles (admin, editor, viewer) | Most apps — simple and predictable |
| **ABAC** (Attribute-Based) | Access determined by attributes of user, resource, and environment (department, region, time) | Fine-grained, complex policies |
| **ACL** (Access Control List) | Explicit list of who can do what on a specific resource | File systems, simple object-level control |
| **Scope-Based** | OAuth scopes limit what a token can do (`read:users`, `write:orders`) | Third-party API integrations |

### Typical Authorization Flow

```
1. Authenticated request arrives with JWT
2. Server extracts identity + role/claims from token
3. Server checks: "Does this role have permission for this action on this resource?"
4. Policy engine returns ALLOW or DENY
5. Request proceeds or returns 403 Forbidden
```

---

## Authentication & Authorization in Action

**Example: E-commerce API**

```
POST /orders             → AuthN: valid JWT?  AuthZ: role=user? ✅
DELETE /admin/users/101  → AuthN: valid JWT?  AuthZ: role=admin? ✅ else 403
GET /orders/55           → AuthN: valid JWT?  AuthZ: is order owner or admin? ✅
```

---

## Security Best Practices

| Area | Best Practice |
|---|---|
| **Passwords** | Never store plaintext — always hash with bcrypt or Argon2 |
| **Tokens** | Keep JWTs short-lived (15–60 min); use refresh tokens for longevity |
| **Token storage** | Store in HttpOnly cookies (not localStorage) to prevent XSS |
| **Transport** | Always use HTTPS — never send credentials over plain HTTP |
| **MFA** | Enforce for sensitive operations and admin accounts |
| **Least privilege** | Grant only the minimum permissions a user/service needs |
| **Token revocation** | Maintain a token blocklist or use short expiry + refresh token rotation |
| **Secrets** | Never hardcode API keys — use environment variables or secrets managers |

