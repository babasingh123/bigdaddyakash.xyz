# Design a URL Shortener (LLD focus)

> **Time:** 30–40 minutes

## Problem

Design the class structure of a URL shortener (bit.ly, tinyurl). The system must:

- Accept a **long URL** and return a unique **short code** (e.g., `bit.ly/aBcD12`).
- **Resolve** a short code back to its long URL on lookup.
- Support **custom aliases** (user-chosen short codes).
- Optionally support **expiration**, **analytics** (click counts, referrers), and **per-user** namespaces.
- Use a **cache** to make hot lookups fast.

!!! note "LLD vs HLD"
    The HLD version of this problem is about sharding, geo-replication, and key-generation services. This page is the LLD view — the class structure that lives inside one process. The interesting design choices are around code generation strategy and the cache abstraction.

## Real-world intuition

URL shorteners are deceptively simple in concept and rich in design choices: how do you generate codes without collisions? How do you keep the lookup path fast? How do you support custom aliases without collision-checking on every insert? The right abstractions — pluggable hash generator, cached store — make all of these one-class changes.

## Key classes

- `URLShortener` — top-level singleton; `shorten(long, opts)`, `resolve(code)`.
- `URLMapping` — pairs a short code with a long URL, plus metadata (creation time, expiration, owner, click count).
- `HashGenerator` (interface) — generates short codes. Concrete: `Base62Counter` (sequential), `RandomBase62` (random), `Md5Truncated` (deterministic).
- `Cache` (interface) — fast lookup for hot codes. Concrete: `LruCache` (in-memory), `RedisCache` (remote, in HLD).
- `URLRepository` (interface) — persistent storage; `save(mapping)`, `findByCode(code)`. Today a `Map`, tomorrow a DB.
- `AnalyticsService` (optional) — records clicks; observer of resolve events.

## Design patterns used

- **Singleton (URLShortener)** — one process-wide shortener; clean entry point.
- **Strategy (HashGenerator)** — multiple valid code-generation algorithms. Strategy makes swapping trivial.
- **Repository pattern (URLRepository)** — abstracts the storage layer behind a clean interface; supports swap-in for DB, caching layer, etc. (This is DIP applied to persistence.)
- **Decorator (Cache as a wrapper around the repository)** — a `CachedURLRepository` wraps the underlying repo, checking the cache first. Decorator composes nicely.

## Class diagram (sketch)

```text
   ┌──────────────┐
   │ URLShortener │ «Singleton»
   ├──────────────┤
   │ -repo        │──▶ URLRepository «interface»
   │ -generator   │──▶ HashGenerator «interface»
   │ -cache       │──▶ Cache «interface»
   │ +shorten(u)  │
   │ +resolve(c)  │
   └──────────────┘

   ┌──────────────┐
   │  URLMapping  │
   ├──────────────┤
   │ -code        │
   │ -longUrl     │
   │ -createdAt   │
   │ -expiresAt   │
   │ -clicks      │
   └──────────────┘
```

## Code sketch

```java
class URLMapping {
    private final String code;
    private final String longUrl;
    private final Instant createdAt;
    private final Instant expiresAt;       // nullable
    private final String ownerId;          // nullable
    private long clicks = 0;

    public boolean isExpired() {
        return expiresAt != null && Instant.now().isAfter(expiresAt);
    }
    public void recordClick() { this.clicks++; }
}

interface HashGenerator {
    String next();   // generate a fresh code
}

class Base62Counter implements HashGenerator {
    private final AtomicLong counter = new AtomicLong(1);

    public String next() {
        long n = counter.getAndIncrement();
        return toBase62(n);
    }
    private String toBase62(long n) {
        String alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        StringBuilder sb = new StringBuilder();
        while (n > 0) {
            sb.append(alphabet.charAt((int)(n % 62)));
            n /= 62;
        }
        return sb.reverse().toString();
    }
}

class RandomBase62 implements HashGenerator {
    private final SecureRandom rng = new SecureRandom();
    private final int length;

    public RandomBase62(int length) { this.length = length; }

    public String next() {
        String alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
        StringBuilder sb = new StringBuilder(length);
        for (int i = 0; i < length; i++) {
            sb.append(alphabet.charAt(rng.nextInt(62)));
        }
        return sb.toString();
    }
}

interface URLRepository {
    void save(URLMapping m);
    Optional<URLMapping> findByCode(String code);
}

class InMemoryURLRepository implements URLRepository {
    private final ConcurrentMap<String, URLMapping> store = new ConcurrentHashMap<>();

    public void save(URLMapping m) {
        if (store.putIfAbsent(m.code(), m) != null) {
            throw new CodeCollisionException(m.code());
        }
    }
    public Optional<URLMapping> findByCode(String code) {
        return Optional.ofNullable(store.get(code));
    }
}

class CachedURLRepository implements URLRepository {
    private final URLRepository inner;
    private final Cache<String, URLMapping> cache;

    public CachedURLRepository(URLRepository inner, Cache<String, URLMapping> cache) {
        this.inner = inner;
        this.cache = cache;
    }
    public void save(URLMapping m) {
        inner.save(m);
        cache.put(m.code(), m);
    }
    public Optional<URLMapping> findByCode(String code) {
        URLMapping cached = cache.get(code);
        if (cached != null) return Optional.of(cached);
        Optional<URLMapping> fromInner = inner.findByCode(code);
        fromInner.ifPresent(m -> cache.put(code, m));
        return fromInner;
    }
}

class URLShortener {
    private static final URLShortener INSTANCE = new URLShortener(/* wired dependencies */);
    private final URLRepository repo;
    private final HashGenerator generator;

    private URLShortener(URLRepository r, HashGenerator g) { /* ... */ this.repo = r; this.generator = g; }

    public String shorten(String longUrl, String customAlias, Duration ttl, String ownerId) {
        String code = (customAlias != null) ? customAlias : generator.next();
        URLMapping m = new URLMapping(code, longUrl, Instant.now(),
                                      ttl != null ? Instant.now().plus(ttl) : null,
                                      ownerId);
        repo.save(m);   // throws on collision; retry with a fresh generated code if not custom
        return code;
    }

    public String resolve(String code) {
        URLMapping m = repo.findByCode(code).orElseThrow(NotFoundException::new);
        if (m.isExpired()) throw new ExpiredException();
        m.recordClick();
        return m.longUrl();
    }
}
```

## Key design decisions

1. **Hash generator as Strategy** — sequential base62 (compact, predictable, leaks order), random (uniform, possible collisions need retry), MD5-truncated (deterministic, useful for idempotency). All three are valid and the right choice depends on requirements. Strategy lets them coexist.
2. **Repository pattern + caching decorator** — `URLRepository` is the abstraction; `InMemoryURLRepository`, `DatabaseURLRepository`, and `CachedURLRepository` are all interchangeable. The cached version is a decorator — composes with any underlying repo. Classic DIP + decorator combination.
3. **Custom aliases use the same `save` path** — they just skip the generator. Collisions throw and surface to the caller. No special-case "alias store" needed.
4. **Expiration is data, not a separate state machine** — `expiresAt` is just a timestamp on `URLMapping`. Resolve-time check is simpler than a "garbage collector" thread.
5. **Click count on the mapping** — fine for in-memory; for production scale, fan out to a separate analytics service. The class structure doesn't change.
6. **Singleton for the entry point, dependencies injected** — the public API is a singleton, but its internals (repo, generator, cache) are constructor-injected. Tests substitute fakes.

## Strategy comparison: code generation

| Strategy | Pros | Cons |
|----------|------|------|
| **Sequential Base62 counter** | Compact codes; no collisions; deterministic | Leaks order of creation; needs coordinated counter in distributed systems |
| **Random Base62** | Uniform, hard to enumerate, no coordination | Must check for collision; longer codes for low collision probability |
| **MD5 truncated** | Deterministic — same URL → same code (idempotent shortening) | Reveals the input URL is hashed; truncation collisions possible |

## Extensibility

**Add a database backend:**

1. `class PostgresURLRepository implements URLRepository`.
2. Wire into `URLShortener`. Cache decorator wraps it.

**Add Redis as the cache:**

1. `class RedisCache implements Cache<String, URLMapping>`.
2. Wire into `CachedURLRepository`. URLShortener unchanged.

**Add per-user analytics:**

1. `AnalyticsObserver` — subscribes to resolve events; aggregates per-mapping.
2. `URLShortener.resolve` fires an event after success.

**Add a different code format (e.g., word-based: `bit.ly/red-fox-42`):**

1. `class WordBasedGenerator implements HashGenerator`.
2. Swap in. Repository and shortener unaware.

**Add a different hashing strategy (deterministic + collision-resistant):**

1. `Md5TruncatedGenerator` — but to avoid truncation collisions, increase truncation length and retry with a salt on collision.

## Linked concepts

- [OOP](../fundamentals/oop.md) — small, focused classes; composition.
- [SOLID](../fundamentals/solid.md) — DIP (repository, cache, generator are all interfaces); OCP (new strategies).
- [Design Patterns](../fundamentals/design-patterns.md) — Singleton, Strategy, Decorator (cache wrapping repo).
