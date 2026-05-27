# Design Dropbox

> **Difficulty:** Advanced · **Time:** 50–60 minutes

## Problem

Design a file storage and sync service. The system supports:

- Uploading and downloading files of arbitrary size.
- Syncing files automatically across a user's devices (desktop, mobile, web).
- File versioning — keep a history of changes.
- Sharing files and folders with other users (with permissions).
- Notifications when shared files change.

Scale targets:

- **500M users**
- ~**10 GB storage per user** average → **~5 EB** of total stored bytes.
- Constant background sync from millions of always-on clients.

## Real-world intuition

Dropbox solves a problem that looks simple ("upload a file, download it
later") but hides three nasty truths:

1. **Bandwidth and storage are absurdly correlated to user behavior.** A
   user editing a 1 GB Photoshop file shouldn't re-upload 1 GB every save.
   The interesting move is **chunking + delta sync** — split the file into
   fixed-size chunks, hash each chunk, and only ship chunks that actually
   changed. This is also how the same chunk shared across many users
   ("Important Slides.pdf") gets stored exactly once via deduplication.
2. **Sync is a distributed-systems problem disguised as file management.**
   Two devices editing offline must reconcile. The system must detect
   conflicts, never silently lose data, and converge across millions of
   clients without coordination.
3. **The metadata DB is the hot path.** Most operations don't read or
   write file bytes — they consult metadata: "do I have this version,
   what changed since cursor X, who has access." Metadata must be fast,
   indexed, and sharded; bytes can live in S3 cheaply.

## Approach

### Capacity estimation

- **Total stored:** 500M × 10 GB = **5 EB** in object storage. With 3×
  replication that's ~15 EB raw.
- **Daily new bytes:** assume 1% churn → **50 PB/day** of new chunks.
- **Metadata rows:** ~50–100 chunks per file × ~100 files per user =
  ~5,000 metadata rows per user → **2.5T rows total**. Sharded relational
  DB (MySQL, Vitess) or Spanner-style.
- **Sync API QPS:** 500M clients × 1 sync call every ~30s = **~17M
  req/sec** worldwide. Most return "nothing changed."

These numbers force chunking (storage), aggressive deduplication
(economics), and a tiny, indexable metadata representation (sync hot path).

### Data model

```
-- Sharded by user_id
files(
    file_id PK,
    user_id,
    path,                 -- "/Documents/notes.md"
    size_bytes,
    chunk_list,           -- ordered list of chunk_hashes
    version,
    parent_version,
    created_at,
    updated_at,
    deleted_at NULL
)

-- Append-only history (versioning)
file_versions(file_id, version, chunk_list, created_at, author_id, …)

-- Global table; sharded by chunk_hash
chunks(
    chunk_hash PK,        -- SHA-256 of content
    s3_key,
    size_bytes,
    ref_count
)

-- Permissions
shares(file_id, principal_id, role)   -- role: viewer/editor/owner

-- Per-user event log (used by clients for incremental sync)
sync_events(user_id, cursor, file_id, op, version, ts)
```

### Chunking + deduplication

```
[Local file: 50 MB]
       │
       │ split into 4 MB chunks → 13 chunks
       ▼
       chunk_1.sha256 = abc…
       chunk_2.sha256 = def…
       …
       chunk_13.sha256 = xyz…
       │
       │ ask server: which of these hashes do you already have?
       ▼
       server: [abc, def already exist; upload chunk_3..13]
       │
       ▼
client uploads only missing chunks via pre-signed S3 PUTs
       │
       ▼
client commits file metadata: file_id v2, chunk_list = [...13 hashes]
```

Benefits stack:

- **Resume:** partial uploads pick up at the next missing chunk.
- **Delta sync:** editing 1 byte in chunk 7 only re-uploads 4 MB.
- **Cross-user dedup:** if your boss already uploaded "Q4 plan.pdf" and
  you do too, only metadata + a ref-count bump happen on your side.

The chunker can be **fixed-size** (4 MB) or **content-defined**
(rolling Rabin fingerprint). Content-defined chunking survives
insertions/deletions in large files (rsync-style); fixed-size is simpler
and good enough for most files.

!!! tip
    Production Dropbox uses 4 MB fixed-size chunks because typical user
    files (photos, videos, docs) rarely have small insertions in the
    middle. Fixed-size wins on simplicity.

### Sync algorithm

Each client tracks an opaque `cursor` = "last server event I've processed."

```
client periodically:
    GET /sync/changes?cursor=<last>
    server: { events: [...], next_cursor: <new> }

For each event:
    op=ADD      → download missing chunks, write to disk
    op=DELETE   → remove local file
    op=MODIFY   → diff chunk_list locally, fetch new chunks, atomically swap
```

Server-side, every metadata mutation appends to `sync_events` (per-user
log). A client's "what changed?" question is a partition scan from its
cursor. This works for 500M users because each user's event log is small
and isolated.

For **push** instead of poll, a thin long-poll / WebSocket layer notifies
clients "new events available, sync now." Falls back to polling on
disconnect.

### Conflict resolution

Two devices both edit `/notes.md` while offline. On reconnection both try
to commit a new version with `parent_version = v3`.

Detection: server keeps `parent_version`. First commit wins (`v3 → v4`).
Second commit attempts `parent_version=v3` but current is `v4` → conflict.

Resolution options:

- **Last-write-wins:** simple, loses data. Bad default.
- **Create conflict copy:** server keeps the winner's version, saves the
  loser as `notes (Alice's conflicted copy).md`. **Dropbox's actual
  policy.**
- **Merge:** only for known structured formats (text — three-way merge).

Conflict copies are ugly but preserve every byte the user wrote. Users
prefer "two versions of my file" over "your version disappeared."

### Sharing and permissions

```
ALICE shares /Documents/Q4 with BOB role=editor

shares row: (file_id=Q4_folder, principal=BOB, role=editor)

For BOB's view:
   the sync event stream includes all files he has access to,
   under a virtual path "Shared with me / Q4".
```

Permission check happens on every metadata mutation; bytes are protected
by short-lived signed S3 URLs.

### Architecture diagram

```
   [Desktop client]               [Mobile client]            [Web]
        │ sync                          │                    │
        ▼                                ▼                    ▼
   ┌────────────────────────────────────────────────────────────┐
   │                       Load Balancer                       │
   └──────────┬───────────────────────────────────┬────────────┘
              │                                    │
   ┌──────────▼────────┐                ┌──────────▼─────────┐
   │  Metadata Service │                │  Block Service     │
   │  (file/chunk DB,  │                │  (upload/download  │
   │   sync events)    │                │   via pre-signed   │
   └──┬────────────────┘                │   S3 URLs)         │
      │                                  └─────────┬──────────┘
      ▼                                            ▼
   ┌─────────────────────────┐              ┌──────────────────┐
   │ MySQL/Vitess (sharded   │              │   S3 / blob      │
   │ by user_id)             │              │   (chunks)       │
   │  - files                │              └──────────────────┘
   │  - file_versions        │
   │  - sync_events          │
   │  - shares               │
   └─────────────────────────┘
                                          ┌────────────────────┐
              ┌─────────────────────────▶ │ Notification svc   │
              │ "file changed" events     │ (WebSocket / push) │
              │                            └────────────────────┘
              │
   ┌──────────▼────────┐
   │ Kafka (events)    │
   └───────────────────┘
              │
              ▼
   ┌─────────────────────┐
   │ Search / preview    │
   │ workers, antivirus  │
   └─────────────────────┘
```

## Trade-offs

- **Fixed-size vs. content-defined chunking.** Fixed is simple, fast,
  good for typical files. Content-defined wins on large files with edits
  in the middle but is more CPU. Pick fixed unless you have evidence
  otherwise.
- **Strong vs. eventual metadata consistency.** Inside a single user's
  account: strong consistency (you must see your own write immediately
  on another device or it feels broken). Across shared folders: eventual
  (a collaborator might lag a few seconds).
- **Poll vs. push for sync.** Polling is simple and survives bad
  networks. Push (long-poll or WebSocket) is more responsive but harder
  to operate at 500M-client scale. Hybrid: push if connected, poll
  otherwise.
- **Client-side encryption?** Adds privacy but breaks server-side
  features (search, preview, OCR). Dropbox chose server-side encryption
  + audit trail. iCloud Drive offers optional E2E ("Advanced Data
  Protection").
- **Version retention.** Keep every version forever (storage explodes)
  vs. 30/180 days (loses long history). Tiered policy: recent versions
  on hot storage, older on cold (Glacier).

## Edge cases & gotchas

- **Empty / huge files.** Files smaller than a chunk are stored inline
  in metadata; files larger than RAM must stream in/out chunk by chunk.
- **Resumable uploads.** Track upload session in metadata; client
  re-sends missing chunks only.
- **Garbage collection.** When `ref_count` drops to 0, the chunk is
  unreferenced. Run a delayed GC sweep (e.g., 7 days) to handle
  concurrent ref bumps without races.
- **Deduplication privacy leak.** If user A uploads a unique chunk and
  user B can detect they didn't have to re-upload it, B can probe whether
  A has a given file. Mitigation: per-user salting or "convergent
  encryption" trade-offs.
- **Rename detection.** Naive "delete old, create new" loses
  share/version history. Track rename as a metadata-only operation;
  preserve `file_id`.
- **File path conflicts.** Case-insensitive (mac/win) vs. case-sensitive
  (linux) → "Notes.md" and "notes.md" collide. Normalize on write.
- **Big-fanout shared folder.** A 5,000-person team folder where every
  member's `sync_events` log gets every change is hot. Either shard the
  shared folder's events or coalesce changes server-side.
- **Antivirus / malware.** Async scan on upload, quarantine on match,
  notify owner and revoke shares.
- **Trash / recycle bin.** Soft-delete with `deleted_at`; chunks
  unreferenced only after retention expiry.

## Linked concepts

- [SQL databases](../fundamentals/databases/sql.md) — metadata DB, ACID on commits
- [Database scaling](../fundamentals/databases/scaling.md) — sharding files & chunks
- [Caching strategies](../fundamentals/caching.md) — hot metadata, chunk presence checks
- [Message queues](../fundamentals/message-queues.md) — Kafka for event-driven sync + notifications
- [API design](../fundamentals/api-design.md) — REST + WebSocket / long-poll
- [Microservices](../fundamentals/microservices.md) — metadata vs. block services
- [Distributed systems](../fundamentals/distributed-systems.md) — conflict resolution, consistency models
- [Security](../fundamentals/security.md) — pre-signed URLs, encryption-at-rest
