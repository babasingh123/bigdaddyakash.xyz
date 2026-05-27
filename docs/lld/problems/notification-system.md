# Design a Notification System

> **Time:** 40–50 minutes

## Problem

Design a multi-channel notification service. The system must:

- Send notifications via **email**, **SMS**, and **push**.
- Respect **user preferences** (which channels are enabled, quiet hours).
- Handle **priority** (CRITICAL > HIGH > NORMAL > LOW).
- **Retry** on transient failures (network blip, throttled by provider).
- Be **extensible** — a fourth channel (Slack, WhatsApp) should be a new class, not edits everywhere.

## Real-world intuition

Notification systems are deceptively common — every app has one. The classic anti-pattern is a `NotificationService.send(type, message)` method that grows into a 600-line `switch` with retry logic, provider-specific quirks, and per-user preferences interleaved. The clean design separates each concern into a small class connected by a few patterns.

## Key classes

- `Notification` — id, recipient, content (subject, body), priority, channel(s), attempt count.
- `NotificationSender` (abstract) — base type for each channel; `send(Notification)`.
    - `EmailSender`, `SmsSender`, `PushSender` — concrete senders.
- `User` — has `NotificationPreferences`.
- `NotificationPreferences` — per-channel opt-in, quiet hours, frequency caps.
- `NotificationQueue` — priority queue of pending notifications.
- `NotificationDispatcher` — pulls from the queue, asks each `Sender` to deliver; coordinates retries.
- `RetryStrategy` (interface) — `ExponentialBackoff`, `FixedDelay`, `NoRetry`.
- `NotificationFilter` (Chain of Responsibility) — chains of checks: opt-in, quiet hours, rate limit, content moderation.

## Design patterns used

- **Strategy (sending mechanism, RetryStrategy)** — each channel is a strategy; retry policies are also strategies. Both are obvious candidates and the system gains both at once.
- **Observer (delivery status)** — analytics, audit log, customer-support tooling all want to know "did this notification reach the user?" Observer pattern decouples them from the sender.
- **Chain of Responsibility (filters + retry)** — pre-send filters (opted in? not in quiet hours? under rate limit?) form a chain. Retry on transient failures is a related chain pattern.
- **Factory (SenderFactory)** — picks the right sender for a channel code. Centralized creation.

## Class diagram (sketch)

```text
   ┌─────────────────────┐
   │NotificationDispatcher│
   ├─────────────────────┤
   │ -queue              │──▶ NotificationQueue (priority)
   │ -senders            │──▶ NotificationSender «abstract»
   │ -filters            │──▶ NotificationFilter «chain»
   │ -retryStrategy      │──▶ RetryStrategy «interface»
   └─────────────────────┘

   ┌────────────────────┐    ┌──────────────────┐
   │NotificationSender  │ ┌──┤Notification      │
   │ «abstract»         │ │  ├──────────────────┤
   ├────────────────────┤ │  │ -id, -recipient  │
   │ +send(n)           │ │  │ -content,prio    │
   └─────△──────────────┘ │  │ -channel,attempts│
        │                 │  └──────────────────┘
   ┌────┼────┬─────────┐
 Email  SMS  Push  Slack(future)
```

## Code sketch

```java
enum Priority { CRITICAL, HIGH, NORMAL, LOW }
enum Channel  { EMAIL, SMS, PUSH }
enum Status   { PENDING, SENT, FAILED, DROPPED }

class Notification {
    private final String id;
    private final User recipient;
    private final String subject;
    private final String body;
    private final Priority priority;
    private final Channel channel;
    private int attempts = 0;
    private Status status = Status.PENDING;
}

abstract class NotificationSender {
    public abstract Channel channel();
    public abstract void deliver(Notification n) throws DeliveryException;
}

class EmailSender extends NotificationSender {
    public Channel channel() { return Channel.EMAIL; }
    public void deliver(Notification n) throws DeliveryException {
        // SMTP / SES / Sendgrid call
    }
}

class SmsSender extends NotificationSender {
    public Channel channel() { return Channel.SMS; }
    public void deliver(Notification n) throws DeliveryException {
        // Twilio / Nexmo
    }
}

interface NotificationFilter {
    /** Returns true to allow; false to drop. */
    boolean allow(Notification n);
}

class OptInFilter implements NotificationFilter {
    public boolean allow(Notification n) {
        return n.recipient().preferences().isEnabled(n.channel());
    }
}

class QuietHoursFilter implements NotificationFilter {
    public boolean allow(Notification n) {
        if (n.priority() == Priority.CRITICAL) return true;
        return !n.recipient().preferences().isInQuietHours(Instant.now());
    }
}

class RateLimitFilter implements NotificationFilter {
    public boolean allow(Notification n) { /* per-user, per-window count */ return true; }
}

interface RetryStrategy {
    boolean shouldRetry(Notification n, Exception e);
    Duration backoff(Notification n);
}

class ExponentialBackoff implements RetryStrategy {
    private final int maxAttempts;
    public ExponentialBackoff(int max) { this.maxAttempts = max; }
    public boolean shouldRetry(Notification n, Exception e) {
        return n.attempts() < maxAttempts && isTransient(e);
    }
    public Duration backoff(Notification n) {
        return Duration.ofSeconds((long) Math.pow(2, n.attempts()));
    }
    private boolean isTransient(Exception e) { return !(e instanceof PermanentDeliveryException); }
}

class NotificationDispatcher {
    private final NotificationQueue queue;
    private final Map<Channel, NotificationSender> senders;
    private final List<NotificationFilter> filters;
    private final RetryStrategy retry;
    private final List<DeliveryObserver> observers;

    public void enqueue(Notification n) {
        queue.offer(n);
    }

    public void processOne() {
        Notification n = queue.poll();
        if (n == null) return;

        for (NotificationFilter f : filters) {
            if (!f.allow(n)) { n.setStatus(Status.DROPPED); return; }
        }

        NotificationSender sender = senders.get(n.channel());
        try {
            sender.deliver(n);
            n.setStatus(Status.SENT);
            observers.forEach(o -> o.onDelivered(n));
        } catch (Exception e) {
            n.incrementAttempts();
            if (retry.shouldRetry(n, e)) {
                queue.offerDelayed(n, retry.backoff(n));
            } else {
                n.setStatus(Status.FAILED);
                observers.forEach(o -> o.onFailed(n, e));
            }
        }
    }
}
```

## Key design decisions

1. **Channel-per-sender, polymorphic** — each channel has its own `NotificationSender` subclass. Adding Slack is one class, no edits to the dispatcher.
2. **Filters as Chain of Responsibility** — opt-in, quiet hours, rate limit, content moderation. Each is an independent class with a single `allow(n)` method. Composable in any order. Adding "fraud check" is one filter at the front.
3. **Retry strategy is pluggable** — exponential backoff is the default; fixed-delay is useful for testing; `NoRetry` is useful for cleanly opting out. Inject via constructor.
4. **Priority queue, not FIFO** — `CRITICAL` notifications jump the queue. The queue compares by priority then arrival time.
5. **Observer for delivery analytics** — audit logs, customer-support views, A/B test analysis all need delivery events. Observer keeps the dispatcher's `processOne` short.
6. **Notifications carry their channel** — sender lookup is `Map<Channel, NotificationSender>`. Dispatch is `O(1)`. Adding multi-channel ("email AND SMS") is creating two notifications, one per channel.
7. **Transient vs permanent errors** — only transient errors trigger retry. The retry strategy distinguishes them (e.g., 5xx and timeouts are transient; 401 and "user opted out" are permanent).

## Filter chain (Chain of Responsibility)

```text
   ┌──────┐    ┌────────┐    ┌──────────┐    ┌───────────┐
   │OptIn │───▶│ Quiet  │───▶│RateLimit │───▶│ContentMod │───▶ Sender
   └──────┘    └────────┘    └──────────┘    └───────────┘

   Any filter returning false drops the notification.
   CRITICAL priority can short-circuit Quiet hours.
```

## Extensibility

**Add a new channel (Slack):**

1. `class SlackSender extends NotificationSender`.
2. Register in the dispatcher's sender map. Add `SLACK` to the `Channel` enum.
3. Filters and retry unchanged.

**Add a new filter (e.g., "don't send marketing notifications during exam season for student users"):**

1. `class ExamSeasonFilter implements NotificationFilter`.
2. Add to the filter chain. Other filters unchanged.

**Add a new retry policy (constant 5-second retry, max 3 tries):**

1. `class FixedDelayRetry implements RetryStrategy`.
2. Inject. Dispatcher logic unaware.

**Add idempotency / dedupe:**

1. `Notification.idempotencyKey` field.
2. A `DedupeFilter` checks a seen-keys store; drops duplicates.

**Add scheduled / delayed delivery:**

1. `Notification.scheduledFor: Instant`.
2. Queue prefers scheduled notifications whose time has come; older items waiting count as overdue.

## Linked concepts

- [OOP](../fundamentals/oop.md) — abstract sender, composition of dispatcher with strategies and filters.
- [SOLID](../fundamentals/solid.md) — OCP via new senders/filters; DIP (inject senders, retry, queue); SRP (each filter has one job).
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy (sender, retry), Observer (delivery events), Chain of Responsibility (filters + retry), Factory (sender selection).
