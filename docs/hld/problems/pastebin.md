# Design Pastebin

> **Difficulty:** Beginner · **Time:** 30–40 minutes

## Problem

Design a service like Pastebin where a user can:

- Upload a block of text and receive a unique URL back.
- Visit that URL and read the text.
- Optionally set an expiration time (10 min, 1 hour, 1 day, never).
- Handle **1M pastes per day** with a **100:1 read:write ratio** (heavy reads).

Most pastes are small (a stack trace, a config file) but the long tail
includes multi-megabyte logs and code dumps.

## Real-world intuition

Pastebin looks like the URL shortener's cousin — but the workload is
fundamentally different. A short URL is at most a few hundred bytes of
metadata; a paste can be 10 MB of base64. Stuffing 10 MB blobs into a
relational row balloons your database, kills replication lag, and ruins the
query planner's life.

So the architecturally interesting move is **separating the index from the
payload**. The database stores tiny rows (`{id, blob_pointer, expires_at}`),
and the actual text lives in object storage (S3, GCS) or a blob store. That
single decision drives everything else: how you serve reads (S3 + CDN, not
DB), how you expire pastes (delete the blob *and* the row), and how cheaply
you can scale (storage is pay-per-byte, not per-IOPS).

It also mirrors how every "user uploads stuff" product is built — Imgur,
Gist, Google Docs export, S3 itself. Learning the pattern here transfers
directly.

## Approach

We design around two facts: **most pastes are small**, and **reads are
overwhelmingly dominated by a hot tail** (the recently-created or
recently-linked paste).

### Capacity estimation

- **Writes:** 1M/day ÷ 86,400 ≈ **12 writes/sec** (avg), ~50/sec peak.
- **Reads:** 100M/day ≈ **1,200 reads/sec** (avg), ~5,000/sec peak.
- **Storage:** assume avg paste 10 KB → 1M × 10 KB = **10 GB/day**, **3.6 TB/year**.
  Long tail of large pastes pushes this higher; budget 5 TB/year.
- **Hot working set:** the top 1% of pastes (recent + viral) is what gets
  served — ~10 GB. Easily cacheable.

### Storage strategy

A two-tier approach:

```sql
CREATE TABLE pastes (
    paste_id        VARCHAR(8) PRIMARY KEY,    -- 8-char base62
    user_id         BIGINT,                    -- nullable, anonymous allowed
    storage_type    ENUM('inline','s3'),
    inline_body     TEXT,                      -- used iff storage_type='inline'
    s3_key          VARCHAR(255),              -- used iff storage_type='s3'
    size_bytes      INT,
    syntax          VARCHAR(32),               -- 'python', 'json', etc
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMP NULL,
    INDEX idx_expires (expires_at)
);
```

- **Small pastes (< 64 KB)** are inlined in MySQL. One round-trip, no S3 cost.
- **Large pastes** are streamed to S3 as `pastes/{first2chars}/{paste_id}`,
  and only the pointer is in MySQL.

!!! note "Why two tiers?"
    Inlining every paste makes the DB enormous and replication slow.
    Storing every paste in S3 adds an extra HTTP round-trip even for a
    200-byte snippet. The split gets you cheap operations for both extremes.

### Read path

```
GET /aB3kP9qZ
        │
        ▼
[Redis cache]  ── HIT ──▶ return body
        │
        MISS
        ▼
[MySQL row]  ─ inline ─▶ return body, populate cache
        │
        s3
        ▼
[S3 / CDN]  ──▶ return body, populate cache
```

For viral pastes we add a CDN in front of `GET /raw/{id}` so the actual blob
never touches our servers after the first fetch. The CDN respects the
`Cache-Control: max-age` we emit based on `expires_at`.

### Expiration: lazy + active

Two strategies, used together:

1. **Lazy delete on read** — if `expires_at < now()` when someone fetches,
   return 410 Gone and enqueue a delete job. This handles the common case for
   free.
2. **Active sweep (cron)** — every 5 minutes, `SELECT paste_id, s3_key FROM
   pastes WHERE expires_at < NOW() LIMIT 1000`. Delete the row, delete the
   S3 object. The `idx_expires` index makes this scan cheap.

The index on `expires_at` is critical — without it, the cron job becomes a
full table scan and eventually OOM-kills the DB.

!!! warning
    Never rely on lazy expiration alone. Pastes that are never read again
    will sit forever, costing you storage. Conversely, never rely on the
    cron alone — between sweeps, expired content is still readable, which
    can be a compliance issue.

### Architecture diagram

```
                 ┌──────────────────────────────┐
                 │            CDN               │
                 │   (caches /raw/{id} fetches) │
                 └──────────────┬───────────────┘
                                │
                         ┌──────▼───────┐
                         │ Load Balancer│
                         └──────┬───────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
            ┌────────┐      ┌────────┐      ┌────────┐
            │ API 1  │      │ API 2  │ ...  │ API N  │
            └───┬────┘      └───┬────┘      └───┬────┘
                │               │               │
                └───────┬───────┴───────┬───────┘
                        ▼               ▼
                 ┌─────────────┐  ┌──────────────────┐
                 │   Redis     │  │   S3 / blob      │
                 │ (hot cache) │  │  (large pastes)  │
                 └──────┬──────┘  └──────────────────┘
                        │ miss
                        ▼
                 ┌──────────────────────┐
                 │  MySQL (sharded by   │
                 │   hash(paste_id))    │
                 └──────────┬───────────┘
                            │
                            ▼
                  ┌──────────────────┐
                  │  Cleanup cron    │
                  │  (every 5 min,   │
                  │   expires_at idx)│
                  └──────────────────┘
```

## Trade-offs

- **Inline vs. always-S3.** Inlining small pastes saves an S3 hop but bloats
  the DB. Always-S3 simplifies the storage layer but adds a network round-trip
  to every read. The 64 KB threshold is empirical — measure your distribution
  and adjust.
- **Lazy vs. eager expiration.** Lazy keeps the write path simple but leaks
  storage. Eager (cron) reclaims storage promptly but adds a background job.
  In practice you need both; the cron is the source of truth, lazy is a UX
  safety net.
- **Public-by-default vs. private.** Pastebin is famously a place where
  secrets leak. Adding optional encryption (client-side key in the URL fragment)
  costs nothing on the server and dramatically improves the security posture.
- **CDN caching expiring content.** A CDN happily caches a paste for 24 hours,
  but if the paste was supposed to expire in 10 minutes, the CDN still serves
  it. Either set `Cache-Control: max-age` to `min(expires_in, 600)` or
  purge on expiration.

## Edge cases & gotchas

- **ID collisions.** Same as URL shortener — use a key-pool service or
  random + retry. 8 chars of base62 is $2 \times 10^{14}$ possibilities,
  collisions are negligible.
- **Hot paste / viral link.** A single paste hitting HN can take 50k QPS.
  The S3 + CDN combo handles this without touching your DB.
- **Massive paste DoS.** Without a size cap, a malicious user can upload
  10 GB blobs. Enforce a 10 MB hard limit at the API gateway and reject
  before reading the body.
- **Syntax highlighting at scale.** Rendering syntax-highlighted HTML on
  every read wastes CPU. Render once on write (or first read), cache the
  HTML in Redis with the same TTL as the paste.
- **Search.** If you allow public paste search, do not put `LIKE '%foo%'`
  on MySQL. Push to Elasticsearch and accept eventual consistency.
- **Deleting from S3 must come *after* deleting from DB**, otherwise a
  concurrent reader can resolve a pointer to a missing object. The cleanup
  cron should `DELETE FROM pastes` first, then S3 `DeleteObject`.

## Linked concepts

- [SQL databases](../fundamentals/databases/sql.md) — paste metadata, expiration index
- [Caching strategies](../fundamentals/caching.md) — hot pastes in Redis
- [CDN](../fundamentals/cdn.md) — viral pastes served from the edge
- [API design](../fundamentals/api-design.md) — REST endpoints, 410 Gone for expired
- [Rate limiting](../fundamentals/rate-limiting.md) — preventing spam pastes
- [Database scaling](../fundamentals/databases/scaling.md) — sharding by `paste_id`
