# Design a Notification System

> **Difficulty:** Intermediate · **Time:** 40–50 minutes

## Problem

Design a notification platform that delivers messages to users across
multiple channels. It must:

- Send notifications via **email**, **SMS**, and **push** (mobile + web).
- Respect **user preferences** per channel (opt-in / opt-out per type).
- Provide **at-least-once delivery** with retries.
- Honor **priority** — urgent (2FA codes) jump the line vs. promotional.
- Handle **100M notifications/day** with bursty traffic.

## Real-world intuition

A notification system looks like a simple wrapper around SendGrid / Twilio
/ FCM. The complexity emerges from three facts:

1. **You don't control delivery.** Every channel has third-party APIs with
   rate limits, latency, retries, and failure modes. Your job is to be
   robust *despite* the upstream provider — never to be limited *by*
   them.
2. **Notifications are a write-amplified fan-out.** One business event
   ("order shipped") may produce email + SMS + push + in-app banner,
   each translated, templated, and routed. The "service" is a fan-out
   engine more than a sender.
3. **Failure modes are user-facing.** A delayed 2FA code logs the user
   out. A duplicate promotional email gets you on a spam list. Idempotency
   and priority queues exist for very real, very angry reasons.

This is the canonical "worker queue" architecture interview — but the
interesting choices are around prioritization, deduping, and retry
strategy, not the basic queue.

## Approach

### Capacity estimation

- **Daily volume:** 100M notifications → ~1,200/sec avg, ~5,000/sec peak.
- **Channel split:** typically ~70% push, ~20% email, ~10% SMS.
- **Storage of pending + audit:** ~1 KB metadata × 100M = **100 GB/day**;
  with 90-day retention ~9 TB.
- **Provider rate limits:** SendGrid ~10k req/s on enterprise, Twilio
  even less per number. Architecture must respect these without losing
  notifications.

### Data model

```
notifications(
    notification_id PK,        -- idempotency key
    user_id,
    channel,                   -- email | sms | push | inapp
    template_id,
    payload (JSON),
    priority,                  -- urgent | high | normal | low
    status,                    -- queued | sending | sent | failed | dlq
    attempts,
    created_at,
    sent_at,
    error
)

user_preferences(
    user_id,
    channel,
    notification_type,         -- "order_shipped", "marketing", "2fa"
    enabled bool
)

user_contacts(
    user_id,
    email, phone, push_tokens[]
)
```

### Pipeline

```
[Producer service]                        [Provider APIs]
       │                                  ┌────────────┐
       │ POST /notify {user, type, ...}   │ SendGrid    │
       ▼                                  └────▲────────┘
┌─────────────────┐                            │
│ Notification API│                            │
│ - dedup check   │           ┌────────────────┴────┐
│ - prefs check   │           │   Email Workers     │
│ - template fill │   email   │ - templating        │
└────────┬────────┘  ──────▶  │ - rate limit ctrl   │
         │                    └────────┬────────────┘
         │  enqueue with priority      │
         ▼                              ▼
   ┌────────────────────────────────────────┐
   │       Kafka topics (one per channel)   │
   │  notif.urgent.email                    │
   │  notif.normal.email                    │
   │  notif.urgent.sms                      │
   │  ...                                   │
   └────────────────┬──────────────────────┘
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   [Email]       [SMS]        [Push]
   workers      workers      workers
       │            │            │
       │            ▼            ▼
       │       Twilio etc.    FCM / APNS
       ▼
   SendGrid

   On failure: retry with backoff → DLQ after N attempts
```

### Priority queueing

A naive single queue causes head-of-line blocking: a backlog of marketing
email delays a 2FA code by minutes. Solution: **separate queues per
priority class per channel.**

```
notif.urgent.sms        (workers always drain first)
notif.high.sms
notif.normal.sms
notif.low.sms
```

Workers consume urgent until empty, then high, then normal, etc. — a
classic priority-scheduler pattern. Optionally cap the percentage of
worker capacity allocated to low so an urgent surge can't starve
everything.

### Idempotency

The producer should send a `notification_id` (deterministic, e.g.,
`sha256(event_id + channel)`) so retried events don't double-send. The
worker checks `notifications.notification_id` exists → skip.

```
Producer retries "order shipped" → same notification_id → no duplicate email
```

!!! note "Why idempotency keys, not 'exactly-once' queues?"
    Truly exactly-once delivery across producers, queue, worker, and
    third-party provider is nearly impossible. Idempotency on the
    receiving side makes at-least-once delivery effectively
    exactly-once for the user.

### Retry strategy

```
attempt 1 → immediate
attempt 2 → 30s later
attempt 3 → 5m later
attempt 4 → 30m later
attempt 5 → 6h later  (only for transient errors)
otherwise → dead letter queue (manual triage / alerting)
```

Distinguish error types:

- **Transient** (5xx, network, rate-limited) → retry.
- **Permanent** (invalid phone, unsubscribed user) → DLQ immediately;
  optionally update user prefs.
- **Soft bounce** (mailbox full) → retry once, then DLQ.

### Preference and consent

Before queueing:

```
1. Look up user_preferences[user_id, channel, type]
2. If disabled → drop (audit log: "suppressed by preference")
3. Check global suppression list (unsubscribes, compliance lists)
4. Look up user_contacts → has channel-specific destination?
```

This check **must** happen before enqueueing so we don't waste worker
time and provider quota on notifications that are immediately filtered.

### Templating and localization

Templates live in a versioned store, not in code. Each `template_id` has
per-locale variants (`en`, `es`, `fr`). Workers render with provided
payload variables. This lets PMs change copy without redeploying.

### Provider rate limit handling

Each worker holds a **token bucket** for its provider's quota. When the
bucket empties, the worker sleeps until refill; messages stay queued.
A separate process can move the work to a backup provider if primary is
down for a while.

### Architecture diagram

```
                ┌───────────────────────────────────────────┐
                │             Producer Services             │
                │   (Order, Auth, Marketing, Billing, ...)  │
                └─────────────────────┬─────────────────────┘
                                      │ POST /notify
                                      ▼
                         ┌─────────────────────────┐
                         │   Notification API      │
                         │   - dedup               │
                         │   - prefs               │
                         │   - templating          │
                         │   - emit Kafka events   │
                         └──────────┬──────────────┘
                                    │
                ┌───────────────────┴──────────────────────┐
                ▼                                            ▼
        ┌──────────────────────────────────────────┐  ┌──────────────────┐
        │           Kafka (priority topics)         │  │ Notifications DB │
        │ urgent.email, urgent.sms, urgent.push... │  │ (audit, dedup)   │
        │ normal.*, low.*                          │  └──────────────────┘
        └────────┬──────────┬──────────┬───────────┘
                 ▼          ▼          ▼
            ┌────────┐ ┌────────┐ ┌────────┐
            │ Email  │ │ SMS    │ │ Push   │
            │workers │ │workers │ │workers │
            └────┬───┘ └────┬───┘ └────┬───┘
                 │          │          │
                 ▼          ▼          ▼
            SendGrid    Twilio      FCM / APNS
                 │          │          │
                 └──────────┴──────────┘
                            │
                            ▼
                  Outcome → update DB
                  Failures → retry / DLQ
                  Webhooks → bounce / unsubscribe handling
```

## Trade-offs

- **One queue per channel vs. per priority.** Per-priority avoids
  head-of-line blocking but means more topics to manage. Worth it for
  any system with user-facing urgent messages.
- **At-least-once vs. exactly-once.** At-least-once + idempotency keys
  is the pragmatic choice. Real exactly-once requires expensive
  end-to-end transactions.
- **Synchronous send vs. async queue.** Synchronous gives the producer
  immediate "sent" feedback (sometimes required for 2FA UX). Async is
  cheaper and survives provider outages. Many systems use sync for
  urgent paths only.
- **Centralized vs. per-team notification service.** Centralized
  enforces preferences, suppression lists, and rate limits uniformly.
  Per-team services duplicate logic and create compliance risk.
- **Pull vs. push to providers.** Email is push (we call SendGrid).
  Push notifications go via FCM/APNS which require token management
  (revoked tokens cause failures we must handle).

## Edge cases & gotchas

- **Bounce / unsubscribe webhooks.** Email providers send webhooks on
  bounces and unsubs. Update suppression list immediately to stay off
  blacklists.
- **Token expiry.** FCM/APNS tokens go stale; on `NotRegistered` error,
  remove from `user_contacts`.
- **Time-of-day rules.** Don't SMS a user at 3 AM (unless urgent).
  Worker honors per-user timezone and "quiet hours."
- **Rate spikes from marketing.** A "Black Friday email blast" pushes
  10M into normal priority. Throttle marketing producers; don't let
  them DoS the notification system.
- **Cross-channel deduplication.** User watching email + push will get
  notified twice. Some systems delay push by N seconds; if email is
  opened, push is suppressed.
- **Compliance.** GDPR / CAN-SPAM / TCPA require explicit consent,
  unsubscribe links, and audit logs. Bake into the API, not the workers.
- **Provider failover.** If SendGrid is down, route to backup (SES).
  Both should be configured with the same suppression list to stay
  consistent.
- **Audit and debugging.** "Why didn't user X get the email?" — every
  decision (preference miss, dedup, suppression, send, bounce) must be
  logged with timestamp and reason.
- **Cost.** SMS at $0.01 each × 10M/day = $100k/day. Costs justify
  caching, deduping, and aggressive preference filtering.

!!! tip
    The simplest answer that earns interview points: queues per priority,
    workers per channel, idempotency on `notification_id`, retries with
    exponential backoff, DLQ for triage. Articulate all five and you've
    designed a production-quality notification system.

## Linked concepts

- [Message queues](../fundamentals/message-queues.md) — Kafka topics, priority queues, DLQ
- [Microservices](../fundamentals/microservices.md) — notification as a shared service
- [API design](../fundamentals/api-design.md) — REST + webhook endpoints for providers
- [Misc patterns](../fundamentals/misc-patterns.md) — idempotency keys, circuit breakers
- [Rate limiting](../fundamentals/rate-limiting.md) — provider quotas, token bucket
- [Distributed systems](../fundamentals/distributed-systems.md) — at-least-once delivery
- [Caching strategies](../fundamentals/caching.md) — template + preference caches
- [Monitoring](../fundamentals/monitoring.md) — provider error rates, DLQ alerts
