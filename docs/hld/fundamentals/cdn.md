# Content Delivery Network (CDN)

A CDN is a geographically distributed fleet of cache servers, sitting at the *edge* of the internet — physically close to users. Instead of every request crossing oceans to your origin servers, content is fetched from a nearby CDN node. The result is dramatically lower latency for the user and dramatically less load on your origin.

## Why it exists

Latency is gated by physics. A round trip from Tokyo to `us-east-1` (Virginia) is roughly 150 ms — and that's before TLS handshakes, application processing, or DB queries. For a typical web page with dozens of assets (images, JS, CSS, fonts, video), that latency compounds into multi-second loads.

Two observations make CDNs work:

1. **Static content rarely changes.** An image, a video, a JavaScript bundle — once published, they're identical for every user. They can be cached freely.
2. **Most users are far from your data centers.** Even with multi-region deployments, the *long tail* of users sits far from any of your regions.

A CDN places copies of your static content in hundreds of "edge" locations worldwide. The closest edge to each user serves the content. Round trips drop from 150 ms to 10–20 ms, your origin servers do dramatically less work, and (incidentally) you absorb a lot of DDoS at the edge.

The economic argument is just as strong: CDN bandwidth is cheaper than origin bandwidth, and serving from cache is cheaper than serving from disk.

## How it works

### Request flow

```
1. User requests image from example.com/image.jpg
2. DNS resolves example.com to nearest CDN edge server
3. CDN checks its local cache
4. If HIT  → serve from cache (10–30 ms)
   If MISS → CDN fetches from origin, caches locally, serves to user
5. Future requests for the same asset from the same region: HIT
```

The first request from each region (a "cold" cache) still pays the origin round trip. Every subsequent request from a nearby user is served at edge speed. That asymmetry is why CDNs work brilliantly for popular content (most assets are hit many times) and less well for personalized or rarely-fetched content.

### Architecture

```
                    origin server (us-east-1)
                            ▲
                            │ fetch on miss
            ┌───────────────┼───────────────┐
            │               │               │
       ┌────┴────┐     ┌────┴────┐    ┌────┴────┐
       │ Edge US │     │ Edge EU │    │ Edge JP │
       │ (cache) │     │ (cache) │    │ (cache) │
       └────▲────┘     └────▲────┘    └────▲────┘
            │               │               │
        US users        EU users        JP users
```

Some CDNs add a middle layer ("shield" or "tiered cache") between regional edges and origin, so a popular asset is only fetched from the origin once globally — every edge then fetches from the shield.

### Key features

- **Edge locations** — hundreds to thousands of POPs (points of presence), often within milliseconds of any major population.
- **TTL** — every cached asset has a time-to-live, after which the edge revalidates with the origin. Common pattern: long TTLs on assets with content-hashed filenames (`app.a3f2.js`), short TTLs on HTML.
- **Cache invalidation / purge** — programmatic "forget this URL now" for emergency takedowns. Slow (can take minutes to propagate globally) and rate-limited, so design around content-hashed URLs instead.
- **Compression** — gzip / Brotli compression at the edge, so text assets fly over the wire small.
- **TLS termination** — the CDN terminates TLS, often with HTTP/2 or HTTP/3 between user and edge, plain HTTP between edge and origin (or another TLS hop).
- **Image optimization** — automatic resizing, format conversion (WebP/AVIF), responsive variants.
- **Smart routing** — Anycast + real-time path quality measurements to choose the fastest edge per request.
- **DDoS protection / WAF** — the CDN absorbs L3/L4 floods and filters L7 attacks before they ever reach your origin.

### Cacheability

Not every response should be cached. The HTTP semantics that govern this:

| Header | Behavior |
|--------|----------|
| `Cache-Control: public, max-age=31536000, immutable` | Cache for a year. Use for content-hashed assets. |
| `Cache-Control: private, no-store` | Don't cache. Use for personalized HTML. |
| `Cache-Control: no-cache` | Cache, but revalidate every time before serving. |
| `Cache-Control: s-maxage=60` | CDN-specific TTL (60s) while browsers cache longer. |
| `ETag` / `If-None-Match` | Conditional requests: edge can validate "is my copy still fresh?" without re-downloading. |
| `Vary: Accept-Encoding` | Cache separate copies per variant (gzip vs br vs none). Use sparingly — every Vary dimension multiplies the cache key space. |

A common pattern:

- Long TTL + content-hashed filename for static assets (`app.a3f2.js`).
- Short TTL or `no-cache` for HTML pages.
- `no-store` for authenticated, personalized responses.

### Cache key design

The CDN identifies a cached response by a *cache key* — by default, the URL. Two URLs that should share a cache must look identical to the CDN, which means watching out for:

- **Query string parameters** — by default, `?utm_source=twitter` makes a separate cache entry. Configure the CDN to ignore or normalize tracking parameters.
- **Cookies** — by default, varying on cookies fragments the cache aggressively. Strip cookies before they reach the CDN cache layer.
- **Headers** — `Vary: User-Agent` will create one cache entry per browser version, which is almost never what you want.

## When to use

CDNs are the right call when:

- Content is **largely static**: images, videos, CSS, JavaScript, fonts.
- Users are **geographically distributed**.
- A **single origin** is becoming a bandwidth or compute bottleneck.
- You need **DDoS protection** at the edge.
- You're serving **large file downloads** (software, game patches, podcasts).
- You're doing **video streaming** with HLS/DASH chunks — CDN caches chunks for the most-watched titles.

CDNs are less useful for:

- Highly personalized, per-user HTML (consider edge compute instead).
- Real-time bidirectional streams (WebSockets, gRPC streams) — though some CDNs proxy these now.
- Tiny services with little geographic spread of users.

!!! tip "Cache HTML too, if you can"
    Many teams cache only static assets and miss the biggest win: caching HTML for anonymous users. A landing page hit by an anonymous user can almost always be served from the edge, saving an origin round trip on the *most expensive* response.

## Trade-offs

- **Cost vs. egress savings** — CDNs cost money, but origin bandwidth costs more. For most public-facing sites, CDN is cheaper net.
- **Cache invalidation is hard.** Purge is slow and rate-limited; design with content-hashed URLs so you never need it.
- **Debugging gets harder.** "Why is this user seeing the old version?" becomes a multi-layer hunt: browser cache, ISP cache, CDN cache, origin.
- **Vendor lock-in.** Each CDN has its own purge API, edge logic (workers, functions), and quirks.
- **Privacy / compliance** — your CDN sees decrypted user traffic. Pick a vendor carefully if you serve regulated data.
- **Cold cache penalty** — the first hit in each region is slow. For low-traffic content this can be most hits. "Cache warming" via crawlers or shielded tier helps.

!!! warning "Don't cache anything personalized"
    A common production outage: caching a logged-in user's response at the CDN and accidentally serving it to other users. Always send `Cache-Control: private, no-store` on authenticated responses, or partition the cache by user (which usually defeats the point).

## Linked problems

- [YouTube](../problems/youtube.md) — adaptive bitrate streaming served by CDN; popular video chunks pushed proactively to edge.
- [Netflix](../problems/netflix.md) — predictive caching: titles likely to be watched are pre-positioned at edge POPs before primetime.
- [Instagram](../problems/instagram.md) — photo storage in S3, served via CDN with multiple precomputed thumbnails.
- [WhatsApp](../problems/whatsapp.md) — media (images, videos) uploaded to S3, delivered via CDN.
- [Pastebin](../problems/pastebin.md) — frequently accessed pastes served via CDN.
- [URL Shortener](../problems/url-shortener.md) — popular short URLs can be redirected at the CDN edge before hitting app servers.
- [Search Autocomplete](../problems/search-autocomplete.md) — precomputed suggestions per prefix cached at the edge for ultra-fast response.

**Tools**: CloudFlare, AWS CloudFront, Akamai, Fastly, Google Cloud CDN, Bunny.net.
