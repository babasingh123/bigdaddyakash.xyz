# Monitoring & Observability

Monitoring is the layer that tells you what your system is doing — and, more importantly, what it's doing *wrong*. The three pillars of observability are **metrics** (numerical time series), **logs** (timestamped events), and **traces** (request paths across services). Used together, they answer the questions "is the system healthy?", "what happened?", and "where did time go?".

## Why it exists

A system without observability is a system you can't operate. You won't notice when a deploy regressed latency, you won't know which service is slow during an incident, and you'll guess at root cause when a customer reports a problem at 3 am.

Three reasons monitoring is now non-negotiable:

1. **Distributed systems hide failure.** In a monolith, a stack trace tells you most of the story. In a microservices system, a single user action traverses ten services and three databases — no one process holds the whole story.
2. **Modern reliability targets demand it.** Four nines (99.99%) is 53 minutes of downtime per year. You can't hit that without seeing problems in seconds, not hours.
3. **Engineering velocity depends on it.** Without metrics, deploys ship blind. With metrics, you can roll back instantly when a graph dips.

The three pillars are complements, not substitutes. Metrics tell you something is wrong; logs and traces help you figure out what.

## How it works

### Metrics

Numeric time series, sampled at regular intervals. Cheap to store, cheap to alert on, expensive to dig into for specific events.

#### The Golden Signals (and RED / USE)

Two well-known frameworks for "what to measure":

**Golden Signals** (SRE Book) — for any user-facing service:

- **Latency** — how long requests take. Report **percentiles** (p50, p95, p99), not averages. Averages hide the long tail where the real pain is.
- **Throughput** — requests per second (QPS).
- **Error rate** — percentage of failed requests. 5xx, timeouts, business errors.
- **Saturation** — how full the system is. CPU, memory, queue depth, connection pool usage.

**RED** (for services) — Rate, Errors, Duration. A subset of Golden Signals, easy to remember.

**USE** (for resources) — Utilization, Saturation, Errors. For hardware/OS-level signals (CPU, disk, network).

#### Percentiles

A common interview-killer: never alert on *average* latency.

```
Average response: 100 ms
p50: 80 ms    (50% of requests)
p95: 250 ms   (5% of requests are slower)
p99: 2000 ms  (1% of requests are *very* slow)
```

That 1% might be your most valuable users hitting your slowest endpoint. The average has hidden them entirely. Always reason in percentiles, especially p95 and p99.

#### Pull vs. push

- **Pull-based** (Prometheus) — the monitoring system scrapes each service's `/metrics` endpoint on a schedule. Each service is a passive endpoint; the system knows who's alive.
- **Push-based** (StatsD, Datadog agent) — services emit metrics to a collector. Better for short-lived jobs; needs explicit registration.

#### Tools

- **Prometheus** — open-source, pull-based, the default for Kubernetes-era systems.
- **Datadog** — managed, push-based, polished UI. Expensive at scale.
- **New Relic / Dynatrace** — managed APM with metrics, traces, logs in one.
- **AWS CloudWatch / GCP Cloud Monitoring** — cloud-native, deeply integrated with vendor infra.
- **Grafana** — visualization layer; works with most backends.

### Logging

Logs are timestamped event records. They're the high-cardinality complement to metrics.

#### Structured logging

Plain text logs are unsearchable at scale. Structure them as JSON:

```json
{
  "timestamp": "2024-05-20T12:00:00Z",
  "level": "ERROR",
  "service": "user-service",
  "trace_id": "abc-123",
  "user_id": "42",
  "message": "Failed to create user",
  "error": "Database connection timeout",
  "duration_ms": 5023
}
```

Why structure matters:

- Searchable by field (`error.type:"timeout"`).
- Aggregatable (count errors by service).
- Joinable with other observability data via trace IDs.

#### Log levels

- `DEBUG` — verbose; off in production unless you turn it on.
- `INFO` — normal operation events.
- `WARN` — recoverable but suspect.
- `ERROR` — failed operation.
- `FATAL` — service-killing.

Pick log levels conservatively. Logging at INFO for every request inflates costs without insight; better to expose request stats via metrics and reserve INFO for state changes.

#### Centralized logging

In any non-trivial system, logs from all services flow to one place.

```
   Services ──→ Log shipper (Fluentd, Filebeat, Vector)
                       │
                       ▼
                ┌──────────────┐
                │ Aggregation  │ Logstash / Vector
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │   Storage    │ Elasticsearch / Loki / S3
                └──────┬───────┘
                       ▼
                ┌──────────────┐
                │ Query / UI   │ Kibana / Grafana
                └──────────────┘
```

- **ELK Stack** — Elasticsearch + Logstash + Kibana. The classic, mature, JVM-heavy.
- **Grafana Loki** — log aggregation designed to be cheap; indexes labels, not content.
- **Splunk** — enterprise-grade, expensive, very capable.
- **Datadog Logs / Cloudwatch Logs** — managed offerings tied to their broader platform.

!!! warning "Logs are not metrics"
    A common pattern that scales badly: counting events by grep'ing logs. For anything you'll graph or alert on, use a metric. Logs are for *unstructured detail* — debugging the specific instance once you know roughly where to look.

### Distributed tracing

A trace follows a single request through every service it touches.

```
Request: place_order
         │
[Trace ID: abc123]
 ├─ [Span A: Gateway]        12ms
 │   └─ [Span B: User Svc]    3ms
 ├─ [Span C: Order Svc]      45ms
 │   ├─ [Span D: DB query]   28ms
 │   └─ [Span E: Payment]    14ms
 └─ [Span F: Notification]    8ms
```

#### Key concepts

- **Trace** — a single logical operation that crosses one or more services.
- **Span** — a unit of work within a trace (an HTTP call, a DB query, a function). Spans have parent/child relationships, forming a tree.
- **Context propagation** — every service forwards the trace ID (and parent span ID) to downstream calls, usually via HTTP headers (`traceparent` per W3C Trace Context, or `X-B3-*` in Zipkin format).
- **Sampling** — tracing every request is expensive; tools sample (e.g. 1%) or use head-based / tail-based sampling to keep representative traces.

#### Why this matters

- "Where did 800 ms of latency go?" → a flame graph of spans.
- "Which downstream call failed?" → see which span errored.
- "Is the new deploy slowing one specific endpoint?" → compare spans over time.

Without distributed tracing, the alternative is grepping logs across N services by trace ID — possible, but slow.

#### Tools

- **Jaeger** — CNCF, popular open-source choice.
- **Zipkin** — Twitter-originated, also open-source.
- **AWS X-Ray** — native to AWS.
- **OpenTelemetry** — the *standard* — instrumentation and protocol that all the above support. Standardize on it.

### Alerting

Alerts are the bridge from "we have data" to "someone wakes up". A few principles, learned the hard way:

- **Alert on symptoms, not causes.** "Error rate > 1%" is a symptom users feel. "Disk at 80%" is a cause that may or may not matter. Symptom alerts wake humans; cause alerts go in dashboards.
- **Alert on SLOs, not arbitrary thresholds.** Define a Service Level Objective ("99.9% of requests under 500 ms over 30 days") and alert when the error budget is burning fast.
- **Every alert is actionable.** If there's nothing the on-call can do, it's a dashboard, not an alert. Alert fatigue is real and corrosive.
- **Alert with context.** The notification should include: what's wrong, how serious, link to the dashboard, link to the runbook.

### SLI / SLO / SLA

- **SLI (Service Level Indicator)** — a measurement: "% of requests under 500 ms".
- **SLO (Service Level Objective)** — a target on that indicator: "99.9% of requests should be under 500 ms".
- **SLA (Service Level Agreement)** — a contractual promise: "if SLO is breached, customer gets credit".

The SRE practice of **error budgets** comes from this: if you're targeting 99.9% (one nine of nines), you have 0.1% — about 43 minutes per month — of allowable downtime. If you've used it all by day 20, freeze risky deploys until the budget recovers.

### Health checks

A `/health` (or `/ready`) endpoint that the load balancer, orchestrator, or monitoring system queries.

- **Liveness** — "is the process alive?". A passing liveness check means "don't restart me."
- **Readiness** — "am I ready to serve traffic?". A passing readiness check means "send me requests."
- **Deep health checks** verify downstream dependencies (DB connectivity, cache availability). Don't go *too* deep — a deep check that fails when an optional downstream is degraded will cascade into unnecessary restarts.

See [Misc Patterns: Heartbeat](misc-patterns.md) for the pattern in distributed systems contexts.

## When to use

Always. Specifically:

- **Metrics** — every service emits the four golden signals plus business KPIs.
- **Logs** — every error and significant state change, structured, with trace IDs.
- **Traces** — every public entry point, propagated across all internal calls.
- **Alerts** — on SLO burn rate, on customer-facing symptoms, on hard limits (out of disk, certificate expiry).
- **Health checks** — on every service, used by LB and orchestrator.

## Trade-offs

- **Cost.** Observability bills scale with traffic. Datadog can rival your AWS bill; Splunk is famously expensive. Sampling, log retention policies, and cardinality limits are real engineering work.
- **Cardinality.** High-cardinality labels (e.g. `user_id`) blow up Prometheus storage. Stick to bounded labels (status code, region, service); use logs and traces for per-entity detail.
- **Instrumentation toil.** Every new endpoint, every new dependency wants instrumentation. OpenTelemetry helps, frameworks help more.
- **Noise.** Too many alerts and the on-call ignores them. Curate ruthlessly.
- **Privacy.** Logs often capture PII. Redact at the source, retain only what you need, follow GDPR/CCPA rules.

!!! tip "Build the dashboard before the outage"
    Before a service goes to production, draft its primary dashboard: golden signals, key business metrics, dependency health. The first time you debug an incident is a bad time to discover you don't have the graph you need.

## Linked problems

Observability is implicit in every problem; here are ones where it's worth calling out explicitly:

- [Uber](../problems/uber.md) — high-cardinality traces across matching, pricing, payment.
- [Netflix](../problems/netflix.md) — pioneered much of modern observability practice; chaos engineering reads from monitoring.
- [Notification System](../problems/notification-system.md) — DLQ depth, third-party API error rate, delivery latency are critical metrics.
- [Web Crawler](../problems/web-crawler.md) — crawl throughput, per-domain error rates, queue depth.
- [Rate Limiter](../problems/rate-limiter.md) — alert on `429` rates as both a security and reliability signal.
- [Distributed Cache](../problems/distributed-cache.md) — hit rate, latency, eviction rate, replication lag.
- [URL Shortener](../problems/url-shortener.md) — redirect QPS, cache hit rate, DB latency.
