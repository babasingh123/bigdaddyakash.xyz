# Design WhatsApp

> **Difficulty:** Advanced · **Time:** 50–60 minutes

## Problem

Design a real-time messaging app. The system must support:

- One-to-one and group messaging with millisecond delivery latency.
- Message states: sent (✓), delivered (✓✓), read (✓✓ blue).
- Media sharing (images, videos, audio, documents).
- End-to-end encryption — the server must not be able to read messages.
- Offline message delivery (queue while user is offline).
- Push notifications to mobile devices.

Scale targets:

- **2B users**
- **100B messages per day**

## Real-world intuition

WhatsApp's defining constraint is **end-to-end encryption**. The server is
a dumb pipe — it can route ciphertext, store ciphertext, and notify devices
that ciphertext is available, but it cannot search, moderate, or rank
messages. This simplifies *and* complicates the design:

- **Simplifies:** no need for content indices, content-based ranking, or
  expensive ML on message text.
- **Complicates:** features that look free in a non-E2E system (server-side
  search, server-side notification previews, group fan-out optimization)
  must move to the device or be done with E2E-compatible tricks.

The other defining property is **persistent connections at billion-user
scale**. Each active user holds a TCP/WebSocket connection. Managing 2B
sockets (even with only a fraction online at any moment) is the hardest
infra problem WhatsApp solves. The famous Erlang-based connection-server
architecture exists because each Erlang process is cheap enough to hold
one user connection.

## Approach

### Capacity estimation

- **Concurrent connections:** ~500M+ active at any time.
- **Messages/sec:** 100B/day ≈ **1.15M messages/sec** avg, ~3M+ peak.
- **Message size (text):** ~200 bytes. Daily text payload ≈ **20 TB/day**
  (small).
- **Media:** ~10% of messages carry media; average ~500 KB → **~5 PB/day**
  of media bytes. Lives in S3 + CDN, not the message store.
- **Storage:** message metadata ~250 bytes/row → 100B × 250B = **25 TB/day**,
  ~9 PB/year (Cassandra).

### Data model

```
-- Sharded by user_id (the recipient). Cassandra.
messages(
    user_id,            -- partition key
    msg_id,             -- TIMEUUID, clustering DESC
    conversation_id,
    sender_id,
    ciphertext,         -- encrypted with recipient's session key
    created_at,
    delivered_at,
    read_at
)

-- Group metadata
groups(group_id PK, member_ids[], admins[], settings)

-- Connection routing table
user_connections(user_id PK, server_id, connected_at)
```

Sharding by `user_id` makes "fetch my new messages" a single partition
read. Cassandra is the natural choice — append-heavy, time-ordered,
partition-scoped queries.

### Connection layer

```
[Mobile client]
     │   persistent WebSocket / TCP
     ▼
[Connection Server N]
     │   each server holds ~1M connections
     │   each user_id → exactly one server at a time
     ▼
[Routing service]   (knows user_id → server_id)
     │
     ▼
[Message Service]   (writes to Cassandra, fans out to recipient's connection)
```

Key idea: **the connection layer is a separate tier from the messaging
logic.** Connection servers are stateful (they own sockets); message
servers are stateless (they read/write Cassandra). This lets you scale
them independently and absorb connection storms without blowing up the
DB.

User-to-server mapping lives in a fast key-value store (Redis / DynamoDB)
that the message service consults to deliver a message to the right
connection server.

### Sending a 1-to-1 message

```
1. Alice client encrypts msg with Bob's session key → ciphertext
2. Alice posts ciphertext to her connection server
3. Connection server forwards to Message Service
4. Message Service:
   a. assigns msg_id, persists to messages[user=Bob]
   b. looks up Bob's server in routing table
   c. pushes ciphertext via Kafka topic → Bob's connection server
   d. Bob's connection server pushes over WebSocket
5. Bob's client decrypts; sends "delivered" ACK back
6. Alice gets ✓✓
```

If Bob is offline at step 4c, the message stays in Cassandra. When Bob
reconnects, his connection server fetches "messages where
delivered_at IS NULL" for his user_id.

!!! note "Why ack chains, not strong consistency?"
    The "sent / delivered / read" states are inherently distributed —
    Alice's device, the server, and Bob's device each contribute one
    update. Modeling them as a chain of idempotent ACKs (each device
    publishes its state, the server propagates) handles flaky networks
    naturally.

### Group messaging

WhatsApp uses **client-side fan-out** because of E2E encryption. There is
no group key the server can use to re-encrypt — each member has a pairwise
session key with each other member (via the Signal protocol's group
sender-key mechanism).

```
Alice sends "hello" to group G of N members:
   for each member M in G:
       encrypt with sender-key for M
       publish (M, msg_id, ciphertext) to message service
```

The server sees N writes (one per recipient), routes each to that
recipient's connection. Group size is capped (e.g., 1024 in WhatsApp,
256 historically) to keep this fan-out tractable on the client.

!!! warning "No celebrity fan-out trick"
    Unlike Twitter, we cannot fan-out-on-read at the server, because the
    server has nothing to read — it has ciphertext for each recipient
    only. This is the cost of E2E.

### Media sharing

Media is **not** sent over the WebSocket. Flow:

```
1. Alice encrypts the image with a random media key K.
2. Alice uploads ciphertext to S3 via a pre-signed URL.
3. Alice sends Bob a message containing {S3 URL, key K (E2E-encrypted)}.
4. Bob fetches ciphertext from CDN/S3, decrypts with K.
```

The CDN serves only ciphertext blobs; the key never touches the server.
This is why "WhatsApp media" can be CDN-accelerated despite E2E.

### Read receipts

Per-message states `{sent_at, delivered_at, read_at}` are stored alongside
the message. When Bob reads, his client emits a "read" event for each
unread `msg_id`. Batch these into one event per conversation per sync.

### Architecture diagram

```
   [Mobile A]                                          [Mobile B]
        │ WebSocket                                       ▲
        ▼                                                 │
  ┌─────────────┐                                  ┌──────┴───────┐
  │ Connection  │                                  │ Connection   │
  │ Server (A)  │                                  │ Server (B)   │
  └─────┬───────┘                                  └──────▲───────┘
        │                                                  │
        ▼                                                  │
  ┌──────────────────────────────────────────────────────────────┐
  │                       Kafka (routing)                        │
  └──────┬───────────────────────────────────────────────▲───────┘
         │                                                │
         ▼                                                │
  ┌─────────────────────┐    ┌────────────────────┐      │
  │ Message Service     │───▶│  Cassandra         │      │
  │ (stateless)         │    │  messages[user]    │      │
  └─────┬───────────────┘    └────────────────────┘      │
        │ lookup recipient's server                       │
        ▼                                                 │
  ┌──────────────┐                                        │
  │ Routing KV   │  (user_id → server_id)                 │
  │ (Redis/Dyn)  │                                        │
  └──────────────┘                                        │
                                                          │
       Media: client ─▶ S3 (encrypted) ─▶ CDN ─────────── ┘
       Push:  Message Service ─▶ FCM/APNS for offline users
```

## Trade-offs

- **E2E encryption costs server features.** No server-side search, no
  cloud-side backup readable by the company, no content moderation at the
  message level. Backups are encrypted with a user-held key.
- **Client fan-out vs. server fan-out for groups.** Client fan-out
  preserves E2E but limits group size and burns user bandwidth. Server
  fan-out scales bigger but breaks the E2E guarantee. The choice is
  philosophical, not technical.
- **WebSockets vs. push-only.** WebSockets give millisecond latency but
  cost battery. Mobile clients use a hybrid: WebSocket while app is
  foreground, FCM/APNS push to wake the app when backgrounded.
- **Strong vs. eventual consistency on receipts.** Eventual — a "delivered"
  ACK can lag by seconds without anyone noticing. The DB updates are
  idempotent on `(msg_id, state)`.
- **Cassandra vs. SQL.** Cassandra wins for the read pattern
  ("partition=user, give me last N msgs by time"). SQL would require
  expensive sharding plus secondary indexes for the same access pattern.

## Edge cases & gotchas

- **User on multiple devices.** WhatsApp's multi-device support shares
  keys across linked devices via the Signal protocol's device-key
  rotation. Each device decrypts independently.
- **Out-of-order delivery.** Network reorders messages. Clients use
  `msg_id` (TIMEUUID) to sort on receive; the *server* never reorders.
- **Missed deliveries on connection drop.** Recipient reconnects and asks
  "what's new since cursor X". Server replays from Cassandra.
- **Replay of messages after key rotation.** Old messages were encrypted
  with the old key; the device must still have it. WhatsApp keeps key
  history per session.
- **Group membership changes.** Add/remove triggers a fresh sender-key
  exchange among remaining members so removed users cannot decrypt new
  messages.
- **Push notification previews.** Show "1 new message" instead of content
  for E2E-locked previews. iOS/Android extensions can decrypt on-device
  to show content without breaking E2E.
- **Disappearing messages.** Server tombstones after TTL; clients enforce
  delete locally. Server cannot truly "delete" what users have already
  received and stored locally.
- **Spam at scale.** No content visibility means anti-spam relies on
  metadata: send-rate, contact-overlap, abuse reports. ML on graph
  patterns, not on text.
- **WhatsApp Web.** A relay: phone holds the keys, web acts as a
  thin terminal sending/receiving ciphertext via the phone. Eliminates
  the "server holds your messages" problem.

!!! tip
    Anchor your interview on three things: E2E encryption, persistent
    connections, and Cassandra partition design. If you mention all
    three with the right reasoning, you've covered the spine.

## Linked concepts

- [API design (WebSockets)](../fundamentals/api-design.md) — persistent connections
- [Message queues](../fundamentals/message-queues.md) — Kafka between connection and message tiers
- [NoSQL (Cassandra)](../fundamentals/databases/nosql.md) — per-user message partitions
- [Database scaling](../fundamentals/databases/scaling.md) — sharding messages by user_id
- [CDN](../fundamentals/cdn.md) — encrypted media delivery
- [Security](../fundamentals/security.md) — E2E (Signal protocol), key management
- [Distributed systems](../fundamentals/distributed-systems.md) — eventual consistency, ack chains
- [Microservices](../fundamentals/microservices.md) — connection vs. message vs. media services
