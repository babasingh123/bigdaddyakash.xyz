# Design Netflix

> **Difficulty:** Advanced · **Time:** 50–60 minutes

## Problem

Design a global video streaming service. Users can:

- Browse a curated catalog of movies and TV episodes.
- Stream content in high quality with low startup latency.
- Get personalized recommendations.
- Manage profiles, parental controls, and watch history.
- Download for offline viewing.

Scale targets:

- **200M+ subscribers** worldwide.
- **Global availability** across 190+ countries.
- **Peak prime-time traffic** dominating internet bandwidth in many regions
  (Netflix is famously >15% of global internet egress at peak).

## Real-world intuition

Netflix is the opposite shape from YouTube in one critical way: **the
catalog is bounded and curated**. There are roughly tens of thousands of
titles, not billions of user uploads. That single fact unlocks the design:

1. **Every title can be pre-encoded** into every rendition, ahead of time.
   No just-in-time transcoding pipeline is needed for the watch path.
2. **Every popular title can be pre-positioned** at the edge. Netflix runs
   its own CDN — Open Connect — and ships physical appliances to ISPs.
   By 6 PM local time, the night's hits are already sitting inside the
   ISPs' networks.
3. **Recommendations are the product.** Unlike YouTube where search drives
   half of consumption, Netflix users primarily *browse rows*. The row
   order, the row contents, even the row titles are personalized by ML.

So the design problem is less "how do I serve bytes" (Open Connect solves
it) and more "how do I rank, route, resume, and recommend at global scale."

## Approach

### Capacity estimation

- **Subscribers:** 200M, ~70M concurrent at peak in primary regions.
- **Per-stream bitrate:** ~5 Mbps average for 1080p, 15+ Mbps for 4K HDR.
- **Egress at peak:** ~70M × 5 Mbps = ~350 Tbps — handled by Open Connect
  + commercial CDN partners; almost none from origin.
- **Catalog size:** ~50k titles × ~100 GB per title (multiple renditions
  + audio tracks + subtitles) ≈ **~5 PB** of master encoded content,
  replicated globally.
- **Watch history writes:** every active viewer emits a heartbeat every
  30s with `(profile_id, title_id, position)` → ~70M / 30 ≈ **2.3M
  writes/sec** at peak.

### Data model

```
-- Title catalog (cacheable, mostly read-only)
titles(title_id PK, name, description, languages[], renditions[], drm_keys, …)

-- Per-profile watch state, sharded by profile_id
watch_history(profile_id, title_id, last_position, last_watched_at)

-- Pre-computed per-profile recommendation rows
recommendations(profile_id, row_id, title_ids[], generated_at)

-- Profiles within an account
profiles(profile_id PK, account_id, name, avatar, maturity_level)

accounts(account_id PK, plan, billing_status, primary_country, …)
```

Open Connect's content placement is its own metadata layer (which title
exists on which appliance) but is mostly invisible to the application.

### Content delivery: Open Connect

```
[Encoding studio]
       │ master + encode (offline, days/weeks per title)
       ▼
[Central S3 / cold storage]
       │
       │ scheduled "fill" overnight (3 AM local)
       ▼
[ISP-hosted Open Connect Appliances (OCAs)]
       │
       ▼
[Client]  ─── HLS/DASH chunks served from OCA inside the ISP ───
```

When you press Play in Mumbai:

1. Client asks the API for a manifest.
2. API picks the best nearby OCA based on geo-routing and OCA health.
3. Player streams chunks directly from that OCA — usually a router-hop
   away inside the ISP — at near-LAN latency.

This is why Netflix's startup time is <1 sec in most regions: there is no
"WAN crossing" for popular content.

!!! note "Why ship hardware to ISPs?"
    Even a great commercial CDN has finite peering capacity with each
    ISP. Putting the bytes physically inside the ISP eliminates the
    peering bottleneck and dramatically improves p99 startup latency.

### Adaptive bitrate streaming

Same as YouTube: HLS/DASH manifests, 2–6 second chunks, player switches
rendition based on buffer health. Difference: Netflix can pre-encode
*per-title-per-scene* renditions (per-shot bitrate optimization), so the
average bitrate-to-quality ratio is much better than YouTube's.

### Recommendations

```
[Watch events] ──▶ Kafka ──▶ Flink ──▶ Feature store
                                            │
                                            ▼
                       [Daily Spark training jobs]
                                            │
                                            ▼
                       [Per-profile candidate generation]
                                            │
                                            ▼
                       [Cassandra: recommendations table]
                                            │
                                            ▼
[Home page request] ──▶ [Ranker: real-time re-ranks against
                            recent watch features]
                                            │
                                            ▼
[Personalized rows: "Because you watched X", "Trending in your country", …]
```

Two stages — **candidate generation** (offline, broad), then **ranking**
(online, narrow + fresh) — are the canonical recommender pattern. The
ranker can incorporate freshness signals ("you just finished episode 1")
that the overnight job can't.

### Watch history and resume

Every 30 seconds the player POSTs a heartbeat:

```json
{"profile_id":"p1","title_id":"t9","position_sec":1284}
```

Heartbeats land in Cassandra (`partition_key=profile_id`, clustering by
`updated_at`). When the user opens any device, "Continue Watching" is a
single partition read.

!!! tip
    Use idempotent upserts on `(profile_id, title_id)` so duplicate
    heartbeats (network retries) don't corrupt state.

### Architecture diagram

```
                           ┌────────────────────────────────────┐
                           │   Open Connect Appliances (OCAs)   │
                           │  inside ISPs worldwide; serve      │
                           │  HLS/DASH chunks directly          │
                           └────────────┬───────────────────────┘
                                        │ stream chunks
                                        │
┌────────┐        ┌─────────────┐       │
│ Client │◀──────▶│  AWS Edge   │◀──────┘
└───┬────┘        │ (API + auth │
    │             │  metadata)  │
    │             └──────┬──────┘
    │                    │
    │                    ▼
    │           ┌─────────────────────────────┐
    │           │   Application layer (AWS)   │
    │           │ - Playback service          │
    │           │ - Catalog service           │
    │           │ - Recommendation ranker     │
    │           │ - Profile/billing services  │
    │           └────┬────────┬───────────────┘
    │                │        │
    │                ▼        ▼
    │           ┌────────┐  ┌────────────┐
    │           │  RDS / │  │ Cassandra  │
    │           │ Aurora │  │ (watch hx, │
    │           │(catalog│  │  recs)     │
    │           │ /accts)│  └────────────┘
    │           └────────┘
    │
    ▼
[Heartbeats] ─▶ Kafka ─▶ Flink ─▶ Cassandra (watch_history)
                          │
                          ▼
                  [Daily Spark recs training] ─▶ Cassandra
```

## Trade-offs

- **Own CDN vs. commercial CDN.** Building Open Connect is enormous capex
  and ops cost; pays off only at Netflix-level scale. Smaller streamers
  use AWS CloudFront + Akamai blends.
- **Pre-encode everything vs. on-demand.** Pre-encoding wastes storage on
  rarely-watched titles. But for a curated catalog, the storage is a
  rounding error compared to streaming cost — pre-encode wins.
- **Strong vs. eventual consistency.** Recommendations, watch counts, "row
  of trending now" are all eventually consistent. Billing is the only
  strong-consistency boundary, and it lives in a relational DB with ACID.
- **Push to edge vs. let CDN warm up.** Pre-positioning content via Open
  Connect fills is essential — during prime time you can't afford an OCA
  miss for the new Stranger Things episode. The "fills" happen overnight
  when bandwidth is cheap.
- **Per-title encoding bitrates.** Netflix invests heavily in per-shot
  bitrate optimization (different bitrate for action vs. dialog) — adds
  encoding cost but reduces lifetime bandwidth bills materially.

## Edge cases & gotchas

- **DRM.** Every chunk is encrypted; license servers issue per-session
  keys. Get DRM wrong and the entire content deal collapses. Use Widevine
  / FairPlay / PlayReady depending on platform.
- **Offline downloads.** Encrypted chunks downloaded to device; license
  has an expiration time. Need an "online check-in" every N days to keep
  watching.
- **Multi-profile in one household.** Recommendations must not leak — kid
  profile must never see violence based on parent's history. Profiles are
  separate recommendation domains entirely.
- **Account sharing detection.** Geo + device + heartbeat analytics
  detect simultaneous streams from far-apart locations.
- **Hot content surge** (a new season drops at midnight). The OCAs have
  been pre-filled, so the spike is absorbed. The API layer (manifest,
  playback URL generation) must scale separately — usually autoscaled
  ahead of release.
- **Resume across devices.** The watch-history Cassandra read must reflect
  the most recent heartbeat. Use last-write-wins with monotonic
  timestamps.
- **Regional licensing.** A title may be available in US/UK but not in
  Japan. Per-request geo-IP check enforces this; offline-downloaded
  content includes the licensed region in its DRM.
- **Bitrate adaptation gone wrong.** A device on bad Wi-Fi rapidly
  switching renditions creates "ladder thrash." Use hysteresis: switch up
  only after 30s of stable bandwidth above threshold.

## Linked concepts

- [CDN](../fundamentals/cdn.md) — Open Connect, edge caching, geo-routing
- [Caching strategies](../fundamentals/caching.md) — pre-fill, predictive caching
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — watch history, recs
- [Data processing](../fundamentals/data-processing.md) — Spark batch, Flink stream
- [Distributed systems](../fundamentals/distributed-systems.md) — global consistency model
- [Microservices](../fundamentals/microservices.md) — playback, catalog, billing
- [Security](../fundamentals/security.md) — DRM, encryption, signed manifests
- [Load balancing](../fundamentals/load-balancing.md) — geo-routing to nearest OCA
