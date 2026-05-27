# Design Uber

> **Difficulty:** Architect · **Time:** 60+ minutes

## Problem

Design a ride-sharing service. The system must support:

- Riders requesting rides from a pickup location.
- Drivers receiving and accepting nearby ride requests.
- Real-time location tracking of drivers (and trips).
- ETA calculation for both "driver arriving" and "arriving at destination."
- Dynamic surge pricing based on supply/demand.
- Trip payment as an atomic, auditable transaction.

Scale targets:

- **100M riders**
- **5M drivers**
- Real-time location updates from every active driver.

## Real-world intuition

Uber is the canonical *geo-distributed real-time* design problem. Four
properties together make it harder than anything in the social-media
family:

1. **The world is the data structure.** A driver's current location is
   meaningful only relative to nearby riders. You cannot answer "give me
   the 5 closest available drivers to this lat/lng" by scanning a table —
   you need a spatial index that is updated tens of millions of times per
   second.
2. **Updates are constant.** Each of 5M drivers pushes location every 4–5
   seconds → ~1M writes/sec just for "where am I now." This is a
   write-heavy real-time problem disguised as a CRUD app.
3. **Matching is a global optimization, but must execute in <1 second.**
   When a rider requests, you have ~10 seconds to find a driver, get them
   to accept, and start the trip. The matching algorithm is interesting
   precisely because it must be both clever and fast.
4. **Money is involved.** Trips are real-world transactions. Payment must
   be ACID-ish — no double-charges, no lost driver payouts. So buried inside
   this "real-time geospatial" system is a hard distributed-transaction
   problem.

Surge pricing is the cherry on top: a stream-processing pipeline that
modulates the matching engine's economics in real time.

## Approach

### Capacity estimation

- **Location updates:** 5M drivers × 1 update / 4s = **~1.25M writes/sec**.
- **Ride requests:** 100M riders × ~1 ride/day = **~1,200 requests/sec**
  avg, ~10x at weekend peaks.
- **Matching queries:** every request triggers a "find nearby drivers"
  geospatial query against current state — must be < 200 ms.
- **Trip records:** ~100M/day = **~150 GB/day** of trip metadata.
- **Storage for live location:** small (one row per driver, 5M rows). Lives
  in Redis or a sharded geospatial index, not on disk.

### Data model

```
-- Mutable, lives in Redis / in-memory geo index
driver_state(driver_id, lat, lng, status, updated_at, geohash, vehicle_type)

-- Sharded by ride_id (snowflake)
rides(ride_id PK, rider_id, driver_id, status, fare,
      pickup_lat, pickup_lng, dropoff_lat, dropoff_lng,
      requested_at, accepted_at, started_at, ended_at,
      surge_multiplier)

-- Append-only telemetry, sharded by ride_id
ride_locations(ride_id, ts, lat, lng)

-- Strong-consistency MySQL
users(user_id, type, name, …)
drivers(driver_id, license, rating, vehicle_type, …)
payments(payment_id, ride_id, amount, status, idempotency_key)
```

### Location ingestion: Redis + geohash

Drivers send `(driver_id, lat, lng)` every few seconds. We need:

- **Fast write** (1M+ writes/sec).
- **Spatial query** ("give me drivers within 3 km of this point").

Solution: a **geohash-indexed Redis cluster** (or QuadTree-backed service).

```
Driver update (lat=12.97, lng=77.59):
  geohash7 = "tdr1vsv"     # ~150m precision cell
  REDIS:
    SET driver_loc:{driver_id}  "{lat,lng,ts,status}"
    GEOADD geo_index {lat} {lng} {driver_id}   # Redis GEO
```

For matching, query `GEOSEARCH geo_index FROMLONLAT {lng} {lat} BYRADIUS 3 km`.
Redis handles this in O(log N + K). For very high QPS, shard the geo index
by geohash prefix.

!!! note "Why geohash, not lat/lng B-tree?"
    A B-tree on lat (or lng) helps with one dimension but not both. A
    geohash collapses 2D into 1D such that nearby points share prefixes,
    so a prefix scan = a spatial scan. QuadTree gives similar behavior
    via spatial subdivision; both are valid.

### Matching algorithm

```
Rider POST /request {pickup, vehicle_type}
    │
    ▼
[Dispatch Service]
    │
    │ 1. GEOSEARCH within radius for available drivers
    │ 2. Filter by vehicle_type, rating, ETA
    │ 3. Score each candidate (distance, acceptance rate, route fit)
    │ 4. Issue request to top driver (or top-K in parallel)
    ▼
[Driver app receives push + WebSocket message]
    │
    │ 30s to accept; on decline or timeout → next candidate
    ▼
[Driver accepts]
    │
    ▼
[rides table: status=ACCEPTED]
    │
    ▼
[Rider notified, trip begins]
```

Scoring is a tiny ML model in production — features include ETA, driver
acceptance rate, vehicle type, current surge level, and "balanced matching"
(don't always pick the same driver). The dispatch must be **idempotent**:
the same `request_id` retried must not match two drivers.

### ETA and routing

ETA is a separate service backed by a road-network graph (OSM data + Uber's
own traffic data). Two flavors:

- **Driver → pickup ETA** at match time. Uses live traffic; cached per
  origin-destination cell.
- **Trip ETA** (pickup → dropoff). Used for fare quoting and rider UX.

Algorithm: bidirectional Dijkstra or A* on a contraction-hierarchy graph.
Pre-computed shortcuts make queries millisecond-level. Live traffic adjusts
edge weights every minute via stream processing of in-trip locations.

### Real-time channel: WebSockets

Drivers and riders both hold open WebSocket connections to a "connection
service" fleet:

```
[Driver app]  ─── WebSocket ─── [Connection Server N] ─── Kafka
[Rider app]   ─── WebSocket ─── [Connection Server M] ─── Kafka
```

The dispatch service publishes "ride offered to driver X" to Kafka; the
connection server that owns driver X's connection pushes the message. This
decouples ride dispatch from connection placement, so a driver can be on
any connection server and still get the message.

### Surge pricing: stream processing

```
[Ride requests] ─▶ Kafka ─▶ Flink ─▶ per-cell (geohash5) request rate
[Driver state]  ─▶ Kafka ─▶ Flink ─▶ per-cell available-driver count
                                           │
                                           ▼
                              demand / supply ratio in 5-min window
                                           │
                                           ▼
                              [surge_multiplier per cell] → Redis
```

The Dispatch service reads `surge_multiplier` from Redis when quoting fare.
This makes pricing change within seconds of a demand spike (e.g., concert
ending) and revert as drivers move in.

### Payments: saga, not 2PC

A trip ends. Now:

1. Charge rider's payment method.
2. Credit driver's payout balance.
3. Take Uber's fee.
4. Record the trip as PAID.

This crosses services (Payments, Drivers, Ledger). Two-phase commit is too
slow and brittle. Use the **saga pattern**:

```
PAYMENT_CHARGED ─▶ DRIVER_CREDITED ─▶ LEDGER_RECORDED ─▶ COMPLETED

If DRIVER_CREDITED fails:
   compensate: PAYMENT_REFUNDED, mark ride as PAYMENT_DISPUTED
```

Each step is idempotent via an `idempotency_key = ride_id + step_name`.
Retries are safe.

### Architecture diagram

```
   ┌───────────────┐                              ┌────────────────┐
   │  Driver app   │◀─────WebSocket──────────────▶│ Connection     │
   └──────┬────────┘                              │ Server fleet   │
          │ location every 4s                     └──────┬─────────┘
          ▼                                              │
   ┌──────────────────┐                                 │ pub/sub
   │ Location Ingest  │                                 │
   │ (lightweight)    │──▶ Redis Geo / QuadTree        │
   └──────────────────┘    (driver state)              │
                                                       ▼
   ┌───────────────┐    ┌──────────────────┐    ┌──────────────┐
   │  Rider app    │───▶│ Dispatch Service │───▶│   Kafka      │
   └───────────────┘    │ (matching)       │    └──────┬───────┘
                        └──────┬───────────┘           │
                               │                       ▼
                               ▼                ┌──────────────┐
                        ┌──────────────┐        │ Flink: surge │
                        │ ETA Service  │        │ + telemetry  │
                        │ (graph alg.) │        └──────┬───────┘
                        └──────────────┘               │
                                                       ▼
                                                ┌─────────────┐
                                                │   Redis     │
                                                │ surge map   │
                                                └─────────────┘

   [Trip ends] ─▶ Payments Service ─▶ saga ─▶ [Driver payout, Ledger]
                                                        │
                                                        ▼
                                                   MySQL (rides,
                                                   payments, users)
```

## Trade-offs

- **Redis Geo vs. PostGIS vs. custom QuadTree.** Redis Geo is fast and
  simple but limited radius/feature set. PostGIS is rich but disk-bound.
  Big systems eventually build their own in-memory QuadTree (Uber's
  Ringpop-based H3 grid is the public example).
- **Push to top-1 vs. top-K candidates.** Push to top-1 minimizes wasted
  driver notifications but is slow if the driver declines. Push to top-K
  in parallel is fast but creates "race" semantics — must invalidate
  losers cleanly.
- **Strong vs. eventual consistency.** Driver location is eventually
  consistent (a few seconds stale is fine). Ride state must be strongly
  consistent — a driver can't accept two rides at once. Use a per-driver
  conditional update.
- **WebSocket vs. polling.** Polling at 5-second intervals would melt the
  load balancers and double the latency budget. WebSockets are mandatory.
- **Saga vs. 2PC for payment.** 2PC is "correct" but blocks. Saga is async
  with compensating actions — the only sane choice for cross-service
  trip payment.

## Edge cases & gotchas

- **Driver goes offline mid-trip.** The connection service detects the
  drop, marks driver "stale," but the rider keeps the ride. When driver
  reconnects, replay missed events.
- **Cancellations.** Rider cancels after accept → enqueue a
  cancellation fee saga; driver is freed to accept new rides.
- **Geohash boundary.** A driver 100m from a query point but in the next
  geohash cell would be missed by a naive prefix scan. Always query
  surrounding cells too (8 neighbors).
- **Double-accept race.** Two riders' dispatch flows both pick the same
  driver. Use Redis `SETNX driver:{id}:busy` with TTL to win exclusively;
  the loser retries.
- **Surge gaming.** Drivers logging off to drive surge up. Detect via
  cluster behavior and dampen the multiplier rise.
- **Geographic sharding.** A trip in Mumbai shouldn't be served by a
  Frankfurt cluster. Shard by city/region; cross-region travel (rare) is
  handled via a "trip metadata" central service.
- **Replay of payment events.** All payment operations must be idempotent
  on `idempotency_key`. Network blips will cause retries.
- **Fraud (fake GPS, ghost rides).** ML-based anomaly detection on
  driver telemetry; reverse-trip simulation flagging.

!!! warning
    A common interview mistake: treating "Uber" as a CRUD app. The
    interviewer wants to see *spatial indexing*, *real-time matching*,
    *surge as a stream pipeline*, and *payments as a saga*. Lead with the
    geospatial story.

## Linked concepts

- [Search & indexing](../fundamentals/search-indexing.md) — geospatial indexes, QuadTree, geohash
- [Caching strategies](../fundamentals/caching.md) — Redis for driver state, surge map
- [API design](../fundamentals/api-design.md) — WebSockets for real-time push
- [Message queues](../fundamentals/message-queues.md) — Kafka for dispatch + telemetry
- [Data processing](../fundamentals/data-processing.md) — Flink for surge pricing
- [Distributed systems](../fundamentals/distributed-systems.md) — saga, consistency models
- [Microservices](../fundamentals/microservices.md) — dispatch, ETA, payments as separate services
- [SQL databases](../fundamentals/databases/sql.md) — ACID for trip + payment records
