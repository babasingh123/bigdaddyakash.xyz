# LLD — Low Level Design

Object-oriented design at the **class and method** level: how to model entities, pick the right abstractions, apply SOLID, and reach for the right design pattern instead of an `if/else` chain.

## What is LLD?

**Low Level Design (LLD)** is about designing the internal structure of a system — the classes, interfaces, methods, and their relationships. Where HLD asks *"how do we scale this to 100M users?"*, LLD asks *"how do we structure these 30 classes so that adding a new payment method tomorrow doesn't break checkout today?"*

LLD lives inside a single application or module. You are not picking between Postgres and Cassandra here — you are deciding whether `Payment` should be an abstract class or an interface, whether `ParkingLot` should be a singleton, and whether `Move` belongs in `Piece` or in `Board`.

## LLD vs HLD

| Aspect | High Level Design (HLD) | Low Level Design (LLD) |
|--------|------------------------|------------------------|
| **Focus** | System architecture, components | Classes, interfaces, methods |
| **Scale** | Distributed systems, databases | Single application/module |
| **Questions** | How to scale? Which DB? | Which design pattern? How to structure classes? |
| **Artifacts** | Architecture diagrams, data flow | UML diagrams, code structure |
| **Examples** | Design YouTube, Design Uber | Design Chess, Design Parking Lot |

## What interviewers look for

1. **Object-oriented thinking** — can you model real-world entities as classes with sensible responsibilities?
2. **SOLID principles** — does your code stay open to extension and closed to modification?
3. **Design patterns** — do you reach for Strategy/State/Factory when they fit, instead of stacking `if/else`?
4. **Code quality** — clean, readable, modular code; consistent naming; small focused methods.
5. **Problem solving** — can you break a fuzzy problem ("design Uber") into 6–10 concrete classes?
6. **Communication** — can you explain *why* you picked a pattern, not just *what* pattern you picked?

## How to use this section

Start with the **fundamentals** — every problem in the next section assumes you can recognize Strategy and Singleton on sight. Then walk through problems in order of difficulty (Parking Lot → Vending Machine → Chess → Uber). For each problem, sketch the classes on paper *before* reading the solution.

## Fundamentals

- [Object-Oriented Programming (OOP)](fundamentals/oop.md) — Encapsulation, Abstraction, Inheritance, Polymorphism; composition vs inheritance; interface vs abstract class.
- [SOLID Principles](fundamentals/solid.md) — SRP, OCP, LSP, ISP, DIP with concrete refactors.
- [Design Patterns](fundamentals/design-patterns.md) — 13 patterns across Creational, Structural, Behavioral.
- [UML & Code Quality](fundamentals/uml-code-quality.md) — UML notation cheat sheet + DRY/KISS/YAGNI.

## Problems

Easy (30–40 min)

- [Parking Lot](problems/parking-lot.md)
- [Vending Machine](problems/vending-machine.md)
- [Snake and Ladder](problems/snake-and-ladder.md)
- [URL Shortener](problems/url-shortener.md)
- [LRU Cache](problems/lru-cache.md)
- [Library Management System](problems/library-system.md)

Medium (40–60 min)

- [Hotel Booking](problems/hotel-booking.md)
- [Movie Ticket Booking](problems/movie-booking.md)
- [ATM](problems/atm.md)
- [Restaurant Management](problems/restaurant.md)
- [Car Rental](problems/car-rental.md)
- [Notification System](problems/notification-system.md)
- [Music Streaming (Spotify)](problems/music-streaming.md)
- [Elevator System](problems/elevator.md)
- [File System](problems/file-system.md)
- [Meeting Scheduler](problems/meeting-scheduler.md)

Hard (60+ min)

- [Chess Game](problems/chess.md)
- [Social Network (Facebook)](problems/social-network.md)
- [Ride Sharing (Uber)](problems/ride-sharing.md)
- [Online Shopping (Amazon)](problems/online-shopping.md)
