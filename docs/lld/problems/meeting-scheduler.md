# Design a Meeting Scheduler

> **Time:** 45–60 minutes

## Problem

Design the object model behind a meeting scheduler (Google Calendar, Outlook). The system must:

- Let users **create meetings** with a title, time slot, location/link, and **invited participants**.
- Show each user's **calendar** with their meetings.
- Check participant **availability** and warn about conflicts.
- Find a **common free slot** across multiple participants (within a date range and duration).
- Send **notifications** on invite, update, cancellation, and reminders.
- Support **recurring meetings** (daily, weekly, monthly, custom).

## Real-world intuition

Calendars look simple but are subtle. The two genuinely interesting subproblems are: (a) **find a common free slot** across N people's calendars (classic interval-merging) and (b) **model recurring meetings** without exploding into 10,000 individual events. Get these right and the rest is plumbing.

## Key classes

- `User` — identity; has a `Calendar`.
- `Calendar` — owns the user's meetings; primary API for availability and slot finding.
- `Meeting` — title, time slot, organizer, participants, location, status.
- `TimeSlot` — start + end `Instant` (or `LocalDateTime`); utility methods for overlap, contains, duration.
- `RecurrenceRule` — generates time slots from a base meeting (weekly, monthly, etc.); inspired by RFC 5545 (iCalendar's RRULE).
- `RecurringMeeting` — meeting + recurrence rule + exception dates (when an instance is moved or cancelled).
- `Invitation` — links a `Meeting` to a participant with a response (`PENDING`, `ACCEPTED`, `DECLINED`, `TENTATIVE`).
- `NotificationService` — observer of meeting events; routes to email, push, in-app.
- `ConflictResolver` — interface; strategies for what to do when a new meeting conflicts (warn, auto-decline, suggest alternatives).

## Design patterns used

- **Observer (NotificationService)** — invites, updates, cancellations, and reminders all fan out. Observer keeps `Meeting.update()` short.
- **Strategy (ConflictResolver, SlotFindingStrategy)** — conflict policy and slot-finding algorithms vary (greedy first-fit, prefer mornings, respect working hours). Strategy makes them swappable.
- **Composite (recurring meetings)** — a `RecurringMeeting` is effectively a "container" that yields concrete `Meeting` instances. Treating both as `Meeting`-like keeps the calendar view uniform.

## Class diagram (sketch)

```text
   ┌──────────┐ owns ┌──────────┐ many ┌─────────┐
   │   User   │◇────│ Calendar │◇────│ Meeting │
   └──────────┘     └──────────┘     └────┬────┘
                                          ◇ participants (many users via Invitation)
                                          │
                                          ▼
                                    ┌────────────┐
                                    │ Invitation │ (status per user)
                                    └────────────┘

   ┌────────────────┐
   │RecurringMeeting│◇──▶ Meeting (template) + RecurrenceRule (RRULE)
   └────────────────┘     yields concrete instances on demand

   NotificationService «observer-hub»
   ConflictResolver «interface», SlotFindingStrategy «interface»
```

## Code sketch

```java
class TimeSlot {
    private final Instant start, end;
    public TimeSlot(Instant s, Instant e) { this.start = s; this.end = e; }
    public boolean overlaps(TimeSlot other) {
        return this.start.isBefore(other.end) && other.start.isBefore(this.end);
    }
    public Duration duration() { return Duration.between(start, end); }
}

class Meeting {
    private final String id;
    private String title;
    private TimeSlot slot;
    private User organizer;
    private final Map<User, InvitationStatus> participants = new HashMap<>();
}

class Calendar {
    private final User owner;
    private final List<Meeting> meetings = new ArrayList<>();    // for big calendars, use an interval tree

    public boolean isFree(TimeSlot slot) {
        return meetings.stream().noneMatch(m -> m.slot().overlaps(slot));
    }

    public List<TimeSlot> freeSlots(TimeSlot window, Duration min) {
        // Merge busy intervals within window, then return gaps >= min.
        return List.of();
    }
}

class SlotFinder {
    public Optional<TimeSlot> findCommon(List<Calendar> cals, TimeSlot window, Duration min) {
        // Merge all busy intervals from each calendar (within window),
        // walk the unified timeline, return the first gap >= min.
        return Optional.empty();
    }
}

class RecurrenceRule {
    enum Frequency { DAILY, WEEKLY, MONTHLY }
    private final Frequency freq;
    private final int interval;          // e.g., every 2 weeks
    private final Instant until;
    private final Set<DayOfWeek> byDays;

    public Stream<Instant> occurrencesAfter(Instant base) {
        return Stream.iterate(base, t -> t.plus(periodFor(freq, interval)))
                     .takeWhile(t -> t.isBefore(until));
    }
    private Duration periodFor(Frequency f, int interval) { /* ... */ return Duration.ZERO; }
}

class RecurringMeeting {
    private final Meeting template;
    private final RecurrenceRule rule;
    private final Set<Instant> exceptions = new HashSet<>();   // cancelled instances
    private final Map<Instant, Meeting> overrides = new HashMap<>(); // rescheduled instances

    public Stream<Meeting> instancesBetween(TimeSlot window) {
        return rule.occurrencesAfter(window.start())
                   .filter(t -> !exceptions.contains(t))
                   .map(t -> overrides.getOrDefault(t, instantiate(t)));
    }
    private Meeting instantiate(Instant t) { /* clone template at time t */ return null; }
}

interface MeetingObserver {
    void onCreated(Meeting m);
    void onUpdated(Meeting m);
    void onCancelled(Meeting m);
}

class NotificationService implements MeetingObserver {
    public void onCreated(Meeting m)   { /* notify all invitees */ }
    public void onUpdated(Meeting m)   { /* notify invitees of change */ }
    public void onCancelled(Meeting m) { /* notify invitees */ }
}
```

## Key design decisions

1. **`TimeSlot.overlaps`** — the single most important utility. Most calendar operations reduce to overlap checks; getting this primitive right (with correct half-open intervals to avoid double-counting boundaries) prevents a class of bugs.
2. **Recurrence as a *rule*, not a list of expanded meetings** — storing 10 years of weekly standups as 520 rows is wasteful and brittle. A rule that *yields* instances on demand is the right model. iCalendar's RRULE format is the proven design.
3. **Exceptions and overrides as separate sets** — when you cancel one instance of a recurring meeting, the underlying series isn't modified. When you reschedule one, the override entry replaces the auto-generated instance at that timestamp. Both must be supported.
4. **Slot finding via interval merging** — list every participant's busy intervals in `[window.start, window.end]`, merge them, walk the timeline, return gaps. `O(n log n)` per call.
5. **Conflict policy is pluggable** — some orgs auto-decline conflicts, some warn, some suggest alternatives. Strategy interface keeps the meeting-creation flow generic.
6. **Observer for all notifications** — invitee notifications, reminders, cross-calendar sync, analytics. Adding a Slack reminder integration is one new observer.
7. **Invitation status is per-participant** — meetings track who has accepted/declined/tentative. Calendar view filters on this for "show declined meetings".

## Slot-finding algorithm

```text
Given calendars C1..Cn, window [W_start, W_end], minimum duration D:

  1. Collect busy intervals from each Ci ∩ window.
  2. Sort all intervals by start.
  3. Merge overlapping intervals.
  4. Walk the merged list and the window:
       - Start at W_start.
       - For each busy interval B:
           - If gap [cursor, B.start) ≥ D, return it.
           - cursor = max(cursor, B.end)
       - If [cursor, W_end) ≥ D, return it.
  5. No common slot found.
```

This is `O(n log n)` in total busy intervals, which is the right complexity. For very large calendars, swap the flat list for an interval tree.

## Extensibility

**Add time zones:**

1. `TimeSlot` stores `Instant`s (UTC). User-facing display converts to the user's time zone.
2. Recurrence rules need careful handling around DST — define them in local time with a TZ, evaluate to UTC.

**Add resource booking (meeting rooms, projectors):**

1. `Resource` has its own `Calendar`. Booking a meeting room is just adding an event to its calendar.
2. Slot-finding extends to participant + resource calendars together.

**Add a new conflict resolver (auto-suggest alternative time):**

1. `class AutoRescheduleResolver implements ConflictResolver` — on conflict, calls `SlotFinder` for a nearby slot and proposes it.
2. Inject; existing flow unchanged.

**Add working-hours awareness:**

1. `User.workingHours` filters available slots.
2. `SlotFinder` intersects with each user's working hours before searching.

**Add a notification preference (email vs push only):**

1. `User.notificationPrefs` consumed by `NotificationService` when routing.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition of user/calendar/meeting.
- [SOLID](../fundamentals/solid.md) — SRP (slot-finder vs notifications vs calendar storage); OCP via strategies.
- [Design Patterns](../fundamentals/design-patterns.md) — Observer, Strategy, Composite (recurring meetings).
