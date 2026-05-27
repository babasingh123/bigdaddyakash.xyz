# Design a Web Crawler

> **Difficulty:** Advanced · **Time:** 50–60 minutes

## Problem

Design a web crawler that downloads pages at internet scale. It must:

- Crawl **billions** of URLs.
- Respect `robots.txt` and per-site **politeness** (don't hammer one
  domain).
- **Deduplicate** URLs (don't crawl the same page twice).
- **Prioritize** important URLs (popular pages first).
- Maintain **freshness** — recrawl pages whose content changes often.
- Be **horizontally scalable** across many crawler nodes and
  geographically distributed.

## Real-world intuition

A web crawler is what powers search engines (Googlebot, Bingbot),
archive services (Internet Archive), price aggregators, and SEO tools.
The "crawl the web" problem looks like a BFS textbook exercise — and at
small scale, it is. At billions of URLs, four things stop being trivial:

1. **The frontier doesn't fit in memory.** You cannot keep "all unvisited
   URLs" in a Python set. It must be a distributed, persistent queue
   that supports priority, sharding, and politeness scheduling.
2. **Politeness is the gating constraint.** You could theoretically pull
   200k URLs/sec from one node, but you'd take down half the websites
   you visit. Each host gets ~1 request/sec by convention; the
   architecture exists to spread crawling across hosts, not to hammer
   one.
3. **Duplicates are everywhere.** Same content, different URL
   (`?utm_source=…`). Mirror domains. Session IDs in paths. Without
   aggressive dedup the crawler wastes 70%+ of its budget.
4. **The web changes.** A naive "crawl-once" gives a stale index
   immediately. Real systems schedule recrawls per-URL based on observed
   change frequency.

## Approach

### Capacity estimation

- **Target:** 10B pages, average 100 KB per page = **1 PB** of raw HTML
  for one full crawl.
- **Throughput:** to recrawl over 30 days → 10B / (30 × 86400) ≈
  **3,800 pages/sec** sustained.
- **URL frontier size:** ~50B URLs (with link explosion before dedup).
- **Dedup index:** 50B URL hashes × ~16 bytes = **~800 GB**. Use a
  Bloom filter in memory for fast negative checks (~5 GB at 0.1% FPR
  per 10B items), database for ground truth.

### Components

```
                 ┌────────────────────────────────────┐
                 │           URL Frontier              │
                 │ (sharded priority queue per domain) │
                 └───────────────┬─────────────────────┘
                                 │ next URL (respects politeness)
                                 ▼
                       ┌─────────────────┐
                       │  Fetcher pool   │
                       │  (HTTP workers) │
                       └────────┬────────┘
                                │
                ┌───────────────┴────────────────┐
                ▼                                 ▼
       ┌────────────────┐                ┌────────────────┐
       │ HTML store     │                │ Parser         │
       │ (S3 / HDFS)    │                │ - extract URLs │
       └────────────────┘                │ - dedup        │
                                          └──────┬─────────┘
                                                 │
                                                 ▼
                                ┌────────────────────────────┐
                                │ URL Filter / Normalizer    │
                                │ - canonicalize             │
                                │ - check Bloom filter       │
                                │ - check robots.txt allows  │
                                └──────────┬─────────────────┘
                                           │ new URLs
                                           ▼
                                  back into URL Frontier
```

### URL frontier

The frontier is the heart of the system. Two-level structure:

```
Frontier
├── Front queues (one per priority class, e.g., 5 levels)
│   - URLs picked from front queues by priority
└── Back queues (one per host)
   - Each fetcher pulls from a single back-queue per worker
   - Politeness scheduler enforces min interval per host
```

When a URL is dequeued:

1. **Priority router** selects from a front queue weighted by priority.
2. **Host router** picks the back queue (one per host) and emits a URL
   only if `now > last_fetched_for_host + crawl_delay`.

This guarantees both prioritization (popular sites get crawled first)
*and* politeness (no host hammered).

For 50B URLs the frontier is sharded across many nodes (Kafka per
priority/host bucket is one common implementation).

### Politeness and robots.txt

```
For each domain:
  - Fetch /robots.txt once per day (cache).
  - Parse Allow / Disallow rules.
  - Honor Crawl-delay (default 1 req/sec per domain).
  - Honor Sitemap: hint (use it to seed the frontier).
```

The fetcher consults the cached robots.txt before each request. Disallowed
URLs are dropped silently.

!!! warning
    Ignoring `robots.txt` will get your IP range banned across the
    internet within hours. It's not optional.

### Duplicate detection

Three layers:

1. **URL canonicalization.** Strip session IDs, normalize trailing
   slashes, lowercase host, remove tracking params (`utm_*`). Same URL,
   different surface forms collapse to one.
2. **Bloom filter** in memory: "have I seen this URL hash?" Fast O(k)
   check. ~0.1% false-positive rate accepted (we'll re-skip a URL we
   actually didn't have, no big deal).
3. **Authoritative store** (sharded KV) for definitive yes/no when Bloom
   says "probably."

For content-level dedup (same content at different URLs), hash the
extracted text (e.g., SimHash) and reject near-duplicates.

### Distributed crawling

```
Frontier shards: by host_hash (politeness lives within a shard)
Fetcher nodes:   stateless workers consuming shards
Parser nodes:    consume "fetched" Kafka topic, extract links
```

Consistent hashing of `hash(host)` to fetcher nodes ensures that all
URLs of one host go to the same node — so politeness (rate-per-host) is
enforced locally without coordination. Adding/removing fetcher nodes
just reshuffles host buckets.

!!! note "Why per-host sharding?"
    If two nodes simultaneously crawl `example.com`, you can't enforce
    "1 req/sec/host" without distributed locks. Per-host sharding makes
    politeness a single-node concern.

### Freshness: recrawl scheduling

Per URL, store `last_fetched`, `last_modified`, `change_history`. Compute
an estimated recrawl interval (e.g., exponential backoff with a floor:
news.example.com recrawled hourly; an archived PDF every 30 days). A
background scheduler enqueues due URLs back into the frontier.

### URL processing pipeline

```
[Fetcher]
   │
   ▼
[HTML in S3]   [Kafka topic "fetched_pages"]
                       │
                       ▼
                [Parser worker pool]
                   - extract <a href> URLs
                   - extract text for content dedup
                   - update last_modified
                       │
                       ▼
                [URL normalizer + Bloom check]
                       │
                       ▼
                [Frontier (Kafka)]
```

Asynchronous and decoupled — a slow parser doesn't stall the fetcher,
and vice versa.

### Architecture diagram

```
                       ┌───────────────────────────────┐
                       │       URL Frontier            │
                       │   (Kafka, sharded by host)    │
                       └──────────────┬────────────────┘
                                      │
       ┌──────────────────────────────┼─────────────────────────┐
       ▼                              ▼                         ▼
  ┌──────────┐                  ┌──────────┐               ┌──────────┐
  │ Fetcher 1│                  │ Fetcher 2│       ...     │ Fetcher N│
  │  (hosts  │                  │ (hosts   │               │ (hosts   │
  │   A–F)   │                  │   G–M)   │               │   T–Z)   │
  └────┬─────┘                  └────┬─────┘               └────┬─────┘
       │ HTTP                        │                          │
       ▼                              ▼                          ▼
  [robots.txt cache]                                       [robots.txt cache]
       │
       ▼ raw HTML
  ┌──────────────────────┐
  │   S3 / HDFS          │
  │  (raw pages)         │
  └──────────────────────┘
       │
       ▼
  ┌──────────────────────┐
  │  "fetched" Kafka     │
  └──────────┬───────────┘
             │
             ▼
  ┌──────────────────────┐
  │ Parser workers       │
  │  - extract links     │
  │  - simhash content   │
  │  - dedup vs Bloom    │
  └──────────┬───────────┘
             │ new URLs
             ▼
  Frontier (close the loop)

  Side stores: URL metadata DB (sharded), Bloom filter cluster,
  content hash store
```

## Trade-offs

- **BFS vs. priority-driven crawl.** BFS gives breadth but mixes
  important and unimportant pages. Priority (PageRank, freshness,
  user-driven) crawls the valuable web faster. Real crawlers blend both.
- **Bloom filter vs. exact store.** Bloom catches 99.9% of duplicates
  with a tiny memory footprint. Exact store is needed for the few
  ambiguous cases. Combine.
- **Per-host single-node vs. distributed politeness.** Single-node is
  simple. Cross-node distributed politeness needs locks/Raft and is
  rarely worth it.
- **Headless browsers vs. raw HTML fetch.** Modern sites are JS-heavy;
  raw HTML misses content. Running headless Chrome per URL is 100×
  slower and more expensive. Selective: render only pages that need it
  (detected via heuristics or domain rules).
- **Centralized frontier vs. P2P.** Centralized (Kafka) is simpler.
  Truly P2P crawlers exist but are rare outside research.

## Edge cases & gotchas

- **Crawler traps.** Calendar pages with infinite next-link chains, or
  faceted-search URLs that combinatorially explode. Mitigate with
  per-host depth limits and URL-pattern blacklists.
- **Spider traps from JS.** SPAs that change the URL on every scroll
  produce billions of fake URLs. Skip URLs with too-many path segments
  or query params.
- **Soft 404s.** Page returns 200 with "not found" body. Detect by
  content heuristics and skip future crawls.
- **Geo-fenced content.** Pages behave differently based on requester
  IP. Crawler IPs are usually identifiable — site may serve a stripped
  version. Live with it.
- **Login-walled content.** Don't crawl. If a partner gives access, use
  a separate authenticated pipeline.
- **Compressed responses (gzip/br).** Always send `Accept-Encoding`;
  saves bandwidth massively.
- **Failed fetches.** Retry with exponential backoff. Permanent failures
  (DNS, certificate) get a long cooldown before retry.
- **Sitemap discovery.** robots.txt's `Sitemap:` line points to a tree
  of URLs the site owner wants you to crawl. Use it eagerly — best
  seeding for new domains.
- **Disk/network throughput.** Saving 1 PB requires careful disk
  layout. S3 multipart uploads, compression on write.
- **Identifying yourself.** Send a meaningful `User-Agent` with a
  contact URL. Site admins should be able to email you to opt out;
  good crawler citizenship.

!!! tip
    The web crawler interview reward goes to candidates who say
    "politeness" within the first 60 seconds. Most start with "BFS";
    the politeness scheduler is what makes the problem real.

## Linked concepts

- [Message queues](../fundamentals/message-queues.md) — Kafka as the URL frontier
- [Misc patterns](../fundamentals/misc-patterns.md) — Bloom filter, consistent hashing
- [Caching strategies](../fundamentals/caching.md) — robots.txt cache, content dedup
- [Distributed systems](../fundamentals/distributed-systems.md) — sharding, partitioning by host
- [Rate limiting](../fundamentals/rate-limiting.md) — politeness as a per-host limiter
- [Search & indexing](../fundamentals/search-indexing.md) — what we feed the search index
- [Networking](../fundamentals/networking.md) — DNS, HTTP, gzip negotiation
- [Database scaling](../fundamentals/databases/scaling.md) — sharded URL metadata store
