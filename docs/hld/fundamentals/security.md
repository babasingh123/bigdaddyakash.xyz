# Security

Security in HLD breaks into four practical concerns: **authentication** (who is this user?), **authorization** (what are they allowed to do?), **encryption** (is the data confidential and tamper-evident?), and the standard catalog of **common threats** (SQL injection, XSS, CSRF, DDoS). This page covers the canonical patterns and what to mention in interview designs.

## Why it exists

Security is the only feature where being 99% correct is a failure. A leaked credentials store, an authorization bug, a missed XSS filter — any one of those can mean a breach, an outage, a regulatory fine, or worse. The patterns below exist because attackers exploit the same handful of mistakes again and again, and the industry's response is a small set of conventions that, applied consistently, make those mistakes hard.

A second reason this matters in HLD: security imposes architecture constraints that show up everywhere. Where does TLS terminate? Where do JWTs get validated? Which service has access to which DB? Get it wrong and you've embedded a vulnerability for the lifetime of the system.

## How it works

### Authentication

Authentication is the answer to *"who is sending this request?"* Three common patterns.

#### Session-based (cookies)

```
1. User submits credentials
2. Server validates, creates a session record (DB / cache)
3. Server returns a session ID in a Set-Cookie header
4. Client sends the cookie on every subsequent request
5. Server looks up the session, identifies the user
```

- **Pros**: Server can revoke sessions instantly (delete from cache). Cookie is opaque — no sensitive data on the client.
- **Cons**: Stateful. Each request needs a session store lookup. Sticky sessions or a shared cache (Redis) are required.
- **Use**: Traditional web apps, especially monoliths. Browser-first flows.

#### Token-based (JWT)

JSON Web Tokens encode the user's identity (and optionally permissions) into a signed token.

```
JWT structure: header.payload.signature

Header  : { "alg": "HS256", "typ": "JWT" }
Payload : { "sub": "user_42", "exp": 1716800000, "roles": ["admin"] }
Signature: HMAC-SHA256(header.payload, secret)
```

Flow:

```
1. User submits credentials
2. Server validates, creates a JWT signed with a server-side secret/key
3. Server returns the JWT to client
4. Client sends JWT in Authorization: Bearer header
5. Server verifies signature locally — no DB lookup
```

- **Pros**: Stateless. Easy to scale horizontally. No session store. Works well for APIs and across microservices (the gateway verifies once and passes the claims downstream).
- **Cons**: Hard to revoke. The token is valid until it expires. Mitigations: short expiration + refresh tokens; a revocation list keyed by jti; deny-list of compromised users.
- **Use**: API authentication, mobile clients, microservices.

!!! warning "Don't put secrets in JWT payloads"
    JWTs are signed, not encrypted by default. The payload is base64-encoded — anyone with the token can read it. Don't store passwords, PII, or anything sensitive in there.

#### OAuth 2.0

A delegated *authorization* framework — often used for authentication via OpenID Connect (OIDC), which layers identity on top.

```
1. User clicks "Sign in with Google"
2. Your app redirects to Google's authorization server
3. User grants consent on Google's UI
4. Google redirects back with an authorization code
5. Your server exchanges the code for an access token (and refresh token)
6. Your server fetches user info from Google's API
7. Issue your own session / JWT
```

Token types:

- **Access token** — short-lived (minutes to hour), used to call APIs.
- **Refresh token** — long-lived (weeks to months), used to obtain new access tokens without re-prompting the user.

**Use**: third-party logins, allowing other apps to access your API on behalf of users.

#### Comparison

| Pattern | State | Revocation | Best for |
|---------|-------|------------|----------|
| Session cookies | Server-side | Easy | Traditional web apps |
| JWT | Stateless | Hard (need expiry / blocklist) | APIs, microservices |
| OAuth 2.0 | Mix | Per-token | Third-party access, "sign in with X" |

### Authorization

Authentication tells you *who*. Authorization tells you *what they can do*. Three common models.

#### Role-Based Access Control (RBAC)

Users have **roles**; roles have **permissions**.

```
User Alice    → role: Admin   → permissions: read, write, delete
User Bob      → role: Editor  → permissions: read, write
User Charlie  → role: Viewer  → permissions: read
```

- **Pros**: Simple to model and audit. Easy to express "all admins can do X".
- **Cons**: Can explode when fine-grained permissions matter. ("Editor in this project, Viewer in that one" → roles become tuples of (role, scope).)
- **Use**: Most internal/B2B apps; the default starting point.

#### Attribute-Based Access Control (ABAC)

Permissions depend on **attributes** of the user, resource, and environment.

```
allow if (user.department == document.department)
     and (user.clearance >= document.classification)
     and (request.time within business_hours)
```

- **Pros**: Maximum flexibility. Expresses contextual rules cleanly.
- **Cons**: Hard to audit ("who can read this?"); decisions are computed, not enumerated.
- **Use**: Healthcare, finance, government — anywhere context matters.

#### Access Control Lists (ACL)

Each resource carries an explicit list of which users have which permissions.

```
Document123 → [
    Alice : read+write
    Bob   : read
    Carol : owner
]
```

- **Pros**: Fine-grained per resource. Easy to grant/revoke individually.
- **Cons**: Doesn't scale to large user bases. Storing ACLs per-row gets expensive.
- **Use**: File sharing systems (Google Drive, Dropbox), per-document permissions.

#### Hybrid is common

Real systems use combinations: RBAC for coarse roles, ACLs for per-resource sharing, ABAC for contextual rules. Tools like AWS IAM or Google Zanzibar implement very rich combinations.

### Encryption

Three places encryption applies:

#### In transit (TLS/HTTPS)

- All traffic encrypted on the wire. Use TLS 1.2 or higher (1.3 preferred).
- Certificates issued by a trusted CA (Let's Encrypt for most public-facing services).
- Terminate TLS at the load balancer or CDN; backend traffic may use plain HTTP within a trusted network or another TLS hop for zero-trust.

#### At rest

- Disk-level encryption (LUKS, AWS EBS encryption, GCP CMEK) — almost free with managed cloud.
- Database-level encryption — TDE (Transparent Data Encryption) in Postgres/MySQL/MS SQL.
- Application-level encryption for very sensitive fields (PII, secrets) — encrypt before writing to the DB.
- **Key management**: AWS KMS, Google Cloud KMS, HashiCorp Vault. Rotate keys; never hardcode them.

#### End-to-end (E2E)

Only the sender and the intended recipient can decrypt — even the server can't read the data.

- **Example**: WhatsApp messages (Signal Protocol), Apple iMessage.
- **Implications**: server cannot do server-side search, recovery, or moderation on encrypted content.
- **Use**: messaging, password managers, anywhere user privacy is the product.

#### Hashing

A one-way function — you cannot recover the input from the hash.

```
input: "hunter2"
hash:  "$2b$12$EixZaYVK1fsbw1ZfbX3OXePaWxn96p36WQoeG6Lruj3vjPGga31lW"
```

- **Use for passwords**: bcrypt, Argon2, scrypt. Slow by design (resist brute force). Each hash includes a per-user **salt** so identical passwords hash differently.
- **Use for data integrity**: SHA-256 fingerprints, content-addressed storage (S3 ETags, git).

!!! warning "Never use MD5 or SHA-1 for passwords"
    Both are cryptographically broken and far too fast. They were never designed for password storage. Always use bcrypt/Argon2/scrypt — and ideally use a managed service (Cognito, Auth0, Clerk) rather than rolling your own.

### Common security threats

The OWASP Top 10 covers most of what you'll see in interviews. The headline four:

#### SQL Injection

User input is concatenated into a SQL string, and an attacker injects their own SQL.

```
# DANGEROUS
query = "SELECT * FROM users WHERE name = '" + user_input + "'"
# user_input = "x' OR '1'='1" → returns every row
```

**Defense**: parameterized queries / prepared statements. Always. Every ORM does this; raw drivers do too. Never concatenate user input into SQL.

```python
# safe
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))
```

#### XSS (Cross-Site Scripting)

Untrusted input is rendered into HTML / JS / DOM and executes as code in another user's browser.

```html
<p>Hello, {{ name }}</p>   <!-- name = "<script>steal_cookies()</script>" -->
```

**Defense**:

- Escape/encode user input before rendering. Modern frameworks (React, Vue, Angular) do this by default.
- Content Security Policy (CSP) headers — restrict what scripts can load.
- HTML sanitization for user-generated rich content (DOMPurify).

#### CSRF (Cross-Site Request Forgery)

An attacker tricks a logged-in user's browser into submitting a request to your site (relying on the user's session cookie).

**Defense**:

- CSRF tokens — a per-session secret that must accompany state-changing requests.
- `SameSite=Strict` (or `Lax`) cookies — modern browsers block cross-site cookie sending.
- Don't rely solely on cookies for authentication on APIs — use bearer tokens, which aren't auto-sent.

#### DDoS (Distributed Denial of Service)

Massive volume of traffic from many sources, intending to exhaust your capacity.

**Defense**:

- CDN with DDoS protection (Cloudflare, AWS Shield).
- Rate limiting at the edge (see [Rate Limiting](rate-limiting.md)).
- Traffic filtering (WAF) — block known bad actors, suspicious patterns.
- Autoscaling — absorb bursts without falling over.

#### Honorable mentions

- **Broken authentication** — weak password policies, missing MFA, brute force without lockouts.
- **Sensitive data exposure** — logs that print credentials, error messages that leak structure, S3 buckets without ACLs.
- **Insecure deserialization** — accepting untrusted serialized objects (pickle, Java) and executing them.
- **Supply chain** — compromised npm/pip packages, dependency confusion.
- **Misconfiguration** — open S3 buckets, default credentials, debug endpoints in prod.
- **SSRF (Server-Side Request Forgery)** — your server is tricked into making requests on the attacker's behalf to internal endpoints.

## When to use

Always. Specifically:

- **Authentication**: every multi-user system needs it. Pick session cookies for browser-first apps, JWT for APIs and microservices, OAuth/OIDC for third-party sign-in.
- **Authorization**: every system with more than "owner / no-owner" granularity. Start with RBAC; add ACLs for per-resource sharing; add ABAC if context matters.
- **TLS**: every external-facing endpoint. Increasingly, every internal endpoint too (zero-trust).
- **Encryption at rest**: free with managed cloud — enable it.
- **CSRF protection**: every cookie-authenticated state-changing endpoint.
- **Rate limiting**: every public endpoint, with extra strictness on `/login`, `/signup`, and write endpoints.
- **Parameterized queries**: every database call. No exceptions.

## Trade-offs

- **Statelessness vs. revocation.** JWT scales; sessions revoke instantly. Pick based on which matters more, or layer them.
- **Convenience vs. security.** MFA, short sessions, frequent re-auth all annoy users; skipping them invites breaches. UX research and risk assessment, together.
- **E2E vs. server features.** Encrypting end-to-end disables server-side search, moderation, and recovery. WhatsApp accepted this; Slack/Gmail don't.
- **Roll your own vs. managed.** Auth0, Cognito, Clerk, Supabase Auth are dramatically safer than rolling your own. The trade-off is vendor lock-in and per-user pricing.
- **Logging vs. privacy.** You want logs to debug; you don't want logs to be a PII goldmine. Redact at source, log structurally, retain for the minimum needed.

!!! tip "Defense in depth"
    No single layer should be your only defense. WAF + rate limiting + auth + parameterized queries + input validation + encryption + monitoring — each layer assumes the others might fail. Many breaches happen because one critical layer was missing or misconfigured.

## Linked problems

- [URL Shortener](../problems/url-shortener.md) — API key auth, rate limiting per key, HTTPS everywhere.
- [Instagram](../problems/instagram.md) / [Twitter](../problems/twitter.md) — OAuth for third-party logins; JWT for mobile clients; per-resource ACLs for private accounts.
- [WhatsApp](../problems/whatsapp.md) — Signal-protocol end-to-end encryption; server can't read messages.
- [Dropbox](../problems/dropbox.md) — ACL-based sharing per file/folder; encryption at rest.
- [Netflix](../problems/netflix.md) — DRM for video content; session-based + token auth.
- [Uber](../problems/uber.md) — OAuth + JWT for API auth; PCI scope for payments; TLS throughout.
- [Notification System](../problems/notification-system.md) — secure storage of API keys for third-party providers (Twilio, SendGrid, FCM).
