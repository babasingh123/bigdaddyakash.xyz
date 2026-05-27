# Data Processing

Data processing is everything you do with data *after* it lands in your system: aggregate it, enrich it, transform it, feed it into ML, serve it to a dashboard. The two flavors are **batch** (process large datasets on a schedule) and **stream** (process events as they happen). Real production systems usually combine both.

## Why it exists

Application databases are optimized for **OLTP**: read or write a few rows by ID, fast. They're not designed for "count the unique users who watched a video over the last 30 days, segmented by country". For analytical workloads — reporting, ML, anomaly detection, recommendation, dashboards — you need a different pipeline.

Two further pressures pushed the field beyond just "ETL into a warehouse":

1. **Volume**. Modern systems produce billions of events per day. Even a few hours of accumulated work can drown a single machine. Hadoop and Spark exist to parallelize this.
2. **Latency**. Some questions want answers in seconds, not hours. Fraud detection, anomaly alerts, live dashboards, trending topics — none of these wait for the nightly batch. Stream processing exists to keep up.

The choice between batch and stream is a trade between **completeness/accuracy** and **freshness**. The choice between Lambda and Kappa architectures is about how you reconcile when you need both.

## How it works

### Batch processing

Process **large, bounded datasets** offline, typically on a schedule.

#### Characteristics

- Operates on **finite** data (a day's worth, a partition).
- Optimized for **throughput**, not latency.
- Runs on a schedule (hourly, daily, weekly).
- Produces **complete, accurate** results — every input is processed exactly once with full context.

#### Tools and patterns

- **MapReduce** — Google's original framework. Map phase parallelizes per record, reduce phase aggregates. Conceptually simple, foundational.
- **Apache Hadoop** — open-source MapReduce + HDFS + YARN. Heavyweight; mostly superseded by Spark for new work.
- **Apache Spark** — in-memory, much faster than Hadoop MapReduce. Rich APIs (SQL, DataFrames, MLlib, GraphX). The de facto general-purpose batch engine.
- **Beam / Flink batch mode** — modern unified APIs that target both batch and stream.
- **dbt / SQL-on-warehouse** — increasingly, "batch processing" just means scheduled SQL on Snowflake/BigQuery/Redshift.

#### Use cases

- **ETL** — extract from sources, transform, load into a warehouse.
- **Data warehousing** — populate fact and dimension tables from event streams.
- **Analytics & reporting** — daily metrics, weekly business reviews.
- **ML model training** — historical data feeds training pipelines.
- **Aggregations across very long time windows** — counts and joins over years of data.

### Stream processing

Process **unbounded sequences of events** in (near) real time.

#### Characteristics

- Operates on **continuous** data — events flow in forever.
- Optimized for **low latency** (seconds or sub-second).
- Often **approximate** — sliding-window aggregates, sketches, probabilistic data structures.
- Handles **late-arriving** data, out-of-order events, watermarks.

#### Tools

- **Apache Kafka Streams** — stream processing as a library on top of Kafka topics. Lightweight; runs in your service.
- **Apache Flink** — full-featured streaming engine with exactly-once semantics, sophisticated windowing, stateful operations. The most powerful option.
- **Apache Storm** — older, lower-level. Mostly displaced by Flink and Kafka Streams.
- **Apache Spark Structured Streaming** — micro-batch streaming on Spark. Familiar API, slightly higher latency than Flink.
- **AWS Kinesis Data Analytics**, **Google Dataflow** — managed services.

#### Concepts you'll meet

- **Windowing** — group events into fixed-size, sliding, or session windows for aggregation.
- **Watermarks** — heuristics for "we've probably seen all events up to time T". Lets a window close even when late events trickle in.
- **State** — running totals, joins, deduplication tables — all must be checkpointed for failure recovery.
- **Exactly-once semantics** — Flink + Kafka can guarantee each event is processed once *with respect to state updates*. The hardest guarantee to deliver; the most expensive.

#### Use cases

- **Real-time analytics** — live dashboards, KPIs that update every second.
- **Fraud detection** — score every transaction as it happens, block if suspicious.
- **Anomaly detection / alerting** — surface threshold breaches in seconds.
- **IoT processing** — sensor streams aggregated per device, region, type.
- **Trending topics / leaderboards** — sliding-window counts over events (see [Twitter](../problems/twitter.md)).
- **Change Data Capture (CDC)** — propagate DB changes to other systems in real time.

### Lambda Architecture

Combines a batch layer (accurate, slow) and a speed layer (fast, approximate) feeding a serving layer that merges results.

```
Data → [Batch Layer (accurate, slow, periodic)]   ┐
                                                   ├──→ Serving Layer ──→ Query
       ↓                                          │
       [Speed Layer (fast, approximate, real-time)]┘
```

- **Batch layer** — re-processes the entire historical dataset on a schedule. Produces the "master" view. Slow but correct.
- **Speed layer** — processes the same incoming stream, but only the recent slice. Produces an "incremental" view that fills in until the next batch.
- **Serving layer** — combines the batch view + speed view to answer queries.

**Why this exists**: streaming systems were once flaky and inaccurate; batch was the source of truth. Lambda gave you both: the batch view eventually corrects anything the stream got wrong.

**Pros**: Best of both worlds. Resilient to streaming bugs (batch eventually fixes them).

**Cons**: **Two code paths** — every transformation must be implemented twice, once in the batch engine and once in the stream engine. Drift between the two is a constant source of bugs.

### Kappa Architecture

Drop the batch layer. Use streaming for *everything*. When you need to reprocess, **replay the stream**.

```
Data → [Stream Processing] → Serving Layer → Query
```

- One code path. Same processing logic for live and historical data.
- "Batch" becomes "stream over historical data" — re-run the stream consumer from offset 0.
- Requires a durable, replayable log (Kafka with long retention) and a streaming engine capable of large-scale catch-up (Flink).

**Pros**: One implementation, no drift. Simpler operationally.

**Cons**: The stream engine must handle *all* the cases the batch engine used to — huge historical scans, complex joins, schema changes. Not every problem fits cleanly.

### When to use Lambda vs. Kappa

| Situation | Choose |
|-----------|--------|
| Mature batch pipeline, need to add real-time features | Lambda (incremental) |
| Greenfield, modern streaming engine (Flink), strong infra | Kappa |
| Heavy historical analytics, ML training, complex aggregates | Lambda (or batch-only) |
| Real-time everything, simple aggregates | Kappa |

In practice, plenty of systems are messier hybrids — a streaming pipeline for the hot path, a batch warehouse for the analytical path, both fed by the same event stream. The neat architectural diagrams are aspirational.

### A concrete pipeline

```
       ┌────────┐
       │  App   │  produces events
       └───┬────┘
           │
       ┌───▼────┐
       │ Kafka  │  durable event log (retention: 30 days)
       └─┬────┬─┘
         │    └──────────────┐
         ▼                   ▼
   ┌──────────┐       ┌──────────────┐
   │ Stream   │       │ Batch (Spark)│
   │ (Flink)  │       │  nightly     │
   └────┬─────┘       └──────┬───────┘
        ▼                    ▼
   ┌──────────┐       ┌──────────────┐
   │ Redis /  │       │  Warehouse   │
   │ Cassandra│       │ (Snowflake)  │
   │ (live)   │       │ (analytics)  │
   └──────────┘       └──────────────┘
        ▲                    ▲
        └──── Serving ───────┘
              (API / BI)
```

Events stream into Kafka, a Flink job aggregates the hot path into Cassandra/Redis for real-time API responses, a nightly Spark job produces historical aggregates into the warehouse for BI. New analytical questions get answered against the warehouse; user-facing real-time features hit the live store.

## When to use

- **Batch**: ETL, ML training, historical analytics, reports, long-window joins.
- **Stream**: live dashboards, fraud detection, trending, real-time aggregates, CDC, near-real-time ML features.
- **Lambda**: when you need both *and* you trust batch more than streaming (or you're migrating).
- **Kappa**: when your streaming engine is robust and you'd rather maintain one pipeline.

A useful question to ask before reaching for stream processing: **what is the SLA on the freshness of this output?** Per-second? Use streaming. Per-hour? Probably fine in batch. Per-day? Definitely batch.

## Trade-offs

- **Stream processing is genuinely harder than batch.** Out-of-order events, watermarks, late data, exactly-once semantics — all add real complexity.
- **Batch is simpler but introduces latency.** You can do nightly. You probably can't do per-second.
- **Lambda's dual codebases are a maintenance hazard.** They drift. Bugs differ between the two. Plan for the operational cost.
- **Stream state is expensive** — Flink state can grow huge; checkpoints to durable storage are mandatory.
- **Cost shifts.** Batch is cheap per record but slow. Streaming is fast but pays a per-record latency tax. Pick based on what you actually need.

!!! tip "Start batch; add streaming when latency demands"
    Premature streaming is a top-five HLD trap. If you can answer your question with a 1-hour-old aggregate, do that. Streaming is for things that fail at hour-scale freshness.

## Linked problems

- [YouTube](../problems/youtube.md) — view counts via stream processing (Kafka Streams / Flink); recommendations via batch (Spark) + real-time refinement.
- [Netflix](../problems/netflix.md) — recommendations heavy on batch; real-time signal pulled in via streaming.
- [Twitter](../problems/twitter.md) — trending topics computed by sliding-window stream processing on tweets.
- [Recommendation System](../problems/recommendation-system.md) — the canonical batch + stream (Lambda) architecture.
- [Uber](../problems/uber.md) — surge pricing as continuous stream processing on supply/demand signals.
- [Instagram](../problems/instagram.md) — counters and aggregates as a mix of stream + batch backfill.
- [Web Crawler](../problems/web-crawler.md) — batch-like crawl scheduling combined with streaming ingestion of fetched pages.
