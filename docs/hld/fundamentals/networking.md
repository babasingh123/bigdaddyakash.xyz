# Networking & Protocols

Networking fundamentals — HTTP/HTTPS, TCP vs. UDP, DNS, and the family of real-time HTTP techniques — are the layers every HLD ultimately runs on. You don't need to recite RFCs in an interview, but you should know what each protocol guarantees, what it doesn't, and which to pick for which job.

## Why it exists

You're going to face questions like *"how do you push real-time updates to the client?"* or *"why is this API so slow over mobile?"* Without a mental model of the network stack, the answers are vague. With one, you can reason about whether the bottleneck is DNS resolution, TLS handshake, HTTP/1.1 head-of-line blocking, or just a slow database. The same model lets you pick the right protocol the first time — saving you the rewrite later.

## How it works

### HTTP / HTTPS

HTTP is the request-response protocol that the web is built on. Everything below is layered on top of TCP.

#### Basics

- **Stateless** — each request stands alone; the server doesn't remember the previous one. State lives in cookies, tokens, or session stores.
- **Methods**: `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `HEAD`, `OPTIONS`. See [API Design](api-design.md) for semantics.
- **Status codes**: `2xx` success, `3xx` redirect, `4xx` client error, `5xx` server error.
- **Headers** carry metadata: content type, authentication, caching directives, encoding.

#### HTTPS

HTTPS is HTTP over TLS. It gives you three things:

- **Encryption** — eavesdroppers can't read traffic.
- **Integrity** — tampering with traffic is detected.
- **Authentication** — the server proves it controls the domain (via X.509 certificate signed by a CA).

The cost: a TLS handshake adds 1–2 round trips at the start of each connection (TLS 1.3 reduced this; session resumption further reduces it). Modern infrastructure terminates TLS at the load balancer or CDN to keep the cost off your application servers. See [Security](security.md).

#### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Version | Transport | Multiplexing | Header compression | Notes |
|---------|-----------|--------------|-------------------|-------|
| HTTP/1.1 | TCP | One in-flight request per connection (pipelining is broken in practice) | None | Browsers open 6–8 connections per origin to parallelize. |
| HTTP/2 | TCP | Many streams on one connection | HPACK | Eliminates head-of-line blocking at the HTTP layer (but not TCP). Server push (largely abandoned). |
| HTTP/3 | QUIC (UDP) | Same as HTTP/2 | QPACK | No head-of-line blocking even at the transport layer. Faster connection setup. |

Most performance wins on the modern web come from HTTP/2 + a CDN. HTTP/3 is the next step, especially for mobile (better behavior over flaky networks).

### TCP vs UDP

The two transport-layer protocols. The choice depends on whether you'd rather *guarantee delivery* or *minimize latency*.

#### TCP (Transmission Control Protocol)

- **Connection-based**. The 3-way handshake (`SYN → SYN-ACK → ACK`) establishes a session.
- **Reliable**. Lost packets are retransmitted.
- **Ordered**. Bytes arrive in the order sent.
- **Flow-controlled** and **congestion-controlled** (TCP backs off when the network is overloaded).
- **Slower** because of all of the above.

**Use**: HTTP/HTTPS, database connections, file transfer (FTP/SFTP), SSH, anywhere you need correctness.

#### UDP (User Datagram Protocol)

- **Connectionless**. Just send a datagram, hope it arrives.
- **Unreliable** — no retransmission. Lost packet = lost packet.
- **Unordered**.
- **Tiny header overhead.** Much faster.

**Use**:

- DNS queries (a single packet, retry at the application layer if lost).
- Live video / VoIP (a lost frame is better than a delayed one).
- Online gaming (same reasoning).
- QUIC (HTTP/3) builds reliability *on top of* UDP, getting both speed and correctness.

| Property | TCP | UDP |
|----------|-----|-----|
| Connection | 3-way handshake | None |
| Reliability | Yes (retransmits) | No |
| Order | Preserved | Not preserved |
| Header overhead | ~20 bytes | ~8 bytes |
| Flow control | Yes | No |
| Use case | HTTP, DB, SSH, file transfer | DNS, video, gaming, QUIC |

### DNS (Domain Name System)

DNS is the internet's phone book: it translates human-readable domain names to IP addresses.

```
example.com → 93.184.216.34
```

#### Hierarchy

```
                root DNS (.)
                    │
              ┌─────┴──────┐
              │            │
           TLD (.com)   TLD (.org)
              │
       Authoritative DNS for example.com
              │
       A:    93.184.216.34
       AAAA: 2606:2800:220:1::1
       MX:   mail.example.com
       TXT:  "v=spf1 ..."
```

#### Record types

| Type | Maps domain to | Example |
|------|---------------|---------|
| `A` | IPv4 address | `example.com → 93.184.216.34` |
| `AAAA` | IPv6 address | `example.com → 2606:2800:220:1::1` |
| `CNAME` | Another domain (alias) | `www.example.com → example.com` |
| `MX` | Mail server | `example.com → mail.example.com` |
| `TXT` | Arbitrary text | SPF, DKIM, domain verification |
| `NS` | Authoritative name server | `example.com → ns1.dnshost.com` |
| `SRV` | Service location | Used by some protocols (SIP, XMPP) |

#### Resolution flow

```
1. Client asks OS for example.com
2. OS asks recursive resolver (often the ISP or 1.1.1.1)
3. Resolver checks its cache; if miss, asks the root, TLD, then authoritative
4. Authoritative returns the IP
5. Resolver caches it (with TTL) and returns to client
```

#### Caching

Caching is everywhere — browser, OS, resolver, ISP — gated by TTL. **TTL matters**:

- Long TTL (24h) = fewer queries, faster average response, but slow failover.
- Short TTL (60s) = fast failover, but more DNS load and slower P99.

A common pattern: long TTLs on stable infrastructure (`A` records pointing at a load balancer that won't change), short TTLs on resources you might need to move quickly.

#### DNS as load balancing

You can return multiple A records for the same name, and clients pick (typically the first). Combined with geo-routing (`Route 53` latency-based, etc.), this becomes the basis for [global load balancing](load-balancing.md). It's *coarse* — caching makes precise control impossible — but it's free and works at any scale.

!!! warning "DNS is not instant"
    A typical DNS change can take minutes to propagate due to resolver caching. Plan deploys (esp. failovers) with this in mind. Don't lower TTLs to 0 — many resolvers cap a minimum anyway, and you'll trade reliability for control you don't have.

### Real-time / push patterns

HTTP is request-response. When the server has something to say without being asked, there are four common patterns:

#### Polling

Client asks "anything new?" on a fixed interval.

- Simple, no special infrastructure.
- Wasteful — most requests come back empty. Higher load, higher latency.
- Use only for very low-frequency updates.

#### Long polling

```
1. Client sends a request
2. Server holds the connection open (no immediate response)
3. When data arrives, server responds
4. Client immediately sends a new request
```

- Approximates real-time push using only standard HTTP.
- Each "response" closes the TCP connection (less efficient than persistent options).
- A reasonable fallback for environments that block WebSockets.

#### Server-Sent Events (SSE)

```
GET /events
Accept: text/event-stream
↓
Server pushes a long-lived stream:
  data: {"type":"order","id":42}\n\n
  data: {"type":"notification","msg":"..."}\n\n
```

- One-way (server → client).
- Built on plain HTTP. Easy to deploy, works with HTTP/2.
- Automatic reconnection in browsers, last-event-ID for resume.
- Limited to text (UTF-8), can't send binary directly.

**Use**: live feeds, notifications, log streams — anything you'd otherwise build with WebSockets but don't need client → server channel for.

#### WebSockets

```
HTTP/1.1 GET /ws
Upgrade: websocket
↓ (after handshake)
Bidirectional, persistent TCP connection, frames either direction
```

- Bidirectional.
- Stateful — server holds one connection per client.
- Best for chat, multiplayer games, collaborative editing.

See [API Design: WebSockets](api-design.md) for more.

| Pattern | Direction | Real-time? | Server cost | Use case |
|---------|-----------|------------|-------------|----------|
| Polling | Client→Server | No (interval delay) | Low per req, high total | Status checks |
| Long polling | Effectively push | Yes (sec latency) | Medium | Legacy compatibility |
| SSE | Server→Client | Yes | Medium (long-lived connections) | Live feeds, notifications |
| WebSockets | Bidirectional | Yes | High (persistent state) | Chat, games, collab |

### Putting it together: a request lifecycle

For the canonical question "what happens when you type `https://example.com` and hit enter?":

```
1. DNS lookup: browser → OS → resolver → authoritative → IP
2. TCP handshake (3-way) with IP:443
3. TLS handshake (1 RTT for 1.3, 2 for 1.2)
4. HTTP request sent
5. Server processes (cache, app, DB)
6. Response streams back
7. Browser parses, fetches sub-resources (multiplexed on HTTP/2)
8. Render
```

Every step is a potential bottleneck and every step has well-known mitigations: DNS caching, connection reuse, TLS session resumption, HTTP/2 multiplexing, CDN, etc.

## When to use

- **HTTP/HTTPS** is the default for almost everything user-facing.
- **gRPC over HTTP/2** for internal microservice traffic where latency and throughput matter (see [API Design](api-design.md)).
- **UDP** when packet loss is preferable to delay: voice/video, gaming, DNS, telemetry.
- **WebSockets** for true bidirectional real-time.
- **SSE** for server-to-client streams where you don't need client-to-server channel.
- **Long polling** as a fallback for clients that can't hold persistent connections.

## Trade-offs

- **TCP is correct but heavy.** Each new connection costs RTT(s) just to set up.
- **UDP is fast but unreliable.** You build the reliability you need at the application layer (or use QUIC).
- **DNS caching trades freshness for performance.** Plan with TTL.
- **WebSockets / SSE are stateful.** They constrain how you load-balance and scale. State coordination across nodes (a chat message must reach all of a user's devices, on different servers) becomes the architecture problem.
- **HTTP/1.1 has head-of-line blocking.** A single slow asset can block others on the same connection. HTTP/2 fixed this at the HTTP layer but not at TCP; HTTP/3 (QUIC over UDP) fixes it fully.

!!! tip "Most "slow API" complaints are network, not server"
    Before micro-optimizing your service, measure the DNS, TLS, and TCP costs. A 200 ms response often has 150 ms of network setup. CDNs, keep-alive, HTTP/2, and TLS 1.3 are usually higher-leverage than shaving milliseconds off the handler.

## Linked problems

- [URL Shortener](../problems/url-shortener.md) — DNS, HTTP redirects (301 vs 302) are the core mechanics.
- [WhatsApp](../problems/whatsapp.md) — WebSockets for messaging, fallback to long polling on weak networks.
- [Uber](../problems/uber.md) — WebSockets for live location updates between rider/driver.
- [YouTube](../problems/youtube.md) / [Netflix](../problems/netflix.md) — adaptive bitrate streaming over HTTP (HLS/DASH); UDP-based protocols at the network layer.
- [Notification System](../problems/notification-system.md) — push protocols (APNs, FCM) layered on TCP/HTTP/2.
- [Web Crawler](../problems/web-crawler.md) — DNS caching, connection reuse, HTTP/2 are critical for throughput.
- [Search Autocomplete](../problems/search-autocomplete.md) — SSE-like patterns for streaming suggestions; long-poll fallback.
