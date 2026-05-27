# Design a Social Network (Facebook-style)

> **Time:** 60+ minutes

## Problem

Design the object model behind a Facebook-style social network. The system must:

- Manage **users** with profiles and **friend** relationships (with friend requests).
- Allow users to **post status updates** (text, image, link).
- Allow **liking** and **commenting** on posts, with nested replies.
- Generate a **news feed** showing posts from friends, ranked by some criterion.
- Support **privacy settings** (public, friends-only, custom audiences).
- **Notify** users on relevant events (new friend request, like on your post, mention).

## Real-world intuition

Facebook, LinkedIn, Twitter, Reddit all share the same modeling backbone: a graph of users, an append-only stream of posts, a fan-out problem (whose feed should this post appear in?), and a notification fan-out. The HLD question is *how do we scale this to billions*; the LLD question is *how do we structure the classes so the feed algorithm can change tomorrow without rewriting everything*.

## Key classes

- `User` — profile (name, bio, avatar), friend list, blocked list, privacy settings.
- `Post` — author, content, timestamp, audience, reactions, comments.
- `Comment` — author, content, timestamp, replies (nested comments).
- `FriendRequest` — sender, recipient, status (`PENDING`, `ACCEPTED`, `DECLINED`).
- `NewsFeed` — algorithmic ranking of candidate posts for a user.
- `FeedStrategy` (interface) — pluggable ranking: `ChronologicalFeed`, `EngagementRankedFeed`, `MlRankedFeed`.
- `PrivacySettings` — controls who can see profile fields, posts, and friend list.
- `NotificationService` — observer of platform events; routes to email/push/in-app.
- `SocialNetwork` — top-level orchestrator (the "Facebook" object).

## Design patterns used

- **Observer (notifications)** — likes, comments, friend requests, mentions all fan out to interested parties. Observer is the natural fit; notification channels (push, email, in-app) are observers themselves.
- **Strategy (FeedStrategy)** — feed ranking is the most volatile algorithm in the company; it must be swappable for experimentation. Strategy lets product run A/B tests without changing the rest of the system.
- **Composite (nested comments)** — a `Comment` contains a list of reply `Comment`s; reply threads can nest arbitrarily. Composite lets you walk the tree uniformly (e.g., to count total replies).
- **Factory (PostFactory)** — posts come in multiple types (text, image, link, shared). A factory picks the right concrete `Post` from the request.

## Class diagram (sketch)

```text
   ┌──────┐   friends   ┌──────┐
   │ User │◇───────────▶│ User │
   └───┬──┘  (n:n)      └──────┘
       │ authors
       ▼
   ┌────────┐ 1   * ┌──────────┐
   │  Post  │◇─────│ Comment  │◇──┐ replies (Composite)
   └────────┘       └──────────┘  │
       │                 ▲────────┘
       │ has
       ▼
   ┌────────────────┐
   │ PrivacySettings│
   └────────────────┘

   SocialNetwork ──▶ FeedStrategy «interface»  ─△─  Chrono / Engagement / ML
   SocialNetwork ──▶ NotificationService «observer-hub»
```

## Code sketch

```java
enum Audience { PUBLIC, FRIENDS, CUSTOM }

class User {
    private final String id;
    private final String name;
    private final Set<String> friendIds = new HashSet<>();
    private PrivacySettings privacy;

    public boolean canSee(Post p) {
        return switch (p.audience()) {
            case PUBLIC  -> true;
            case FRIENDS -> friendIds.contains(p.author().id()) || this.id.equals(p.author().id());
            case CUSTOM  -> p.customAudience().contains(this.id);
        };
    }
}

class Post {
    private final User author;
    private final String content;
    private final Instant createdAt;
    private final Audience audience;
    private final List<Reaction> reactions = new ArrayList<>();
    private final List<Comment> comments = new ArrayList<>();
}

class Comment {                         // Composite — comments can have child comments
    private final User author;
    private final String text;
    private final Instant createdAt;
    private final List<Comment> replies = new ArrayList<>();

    public int totalRepliesDeep() {
        return replies.size() + replies.stream().mapToInt(Comment::totalRepliesDeep).sum();
    }
}

interface FeedStrategy {
    List<Post> generate(User viewer, List<Post> candidates);
}

class ChronologicalFeed implements FeedStrategy {
    public List<Post> generate(User viewer, List<Post> candidates) {
        return candidates.stream()
            .filter(viewer::canSee)
            .sorted(Comparator.comparing(Post::createdAt).reversed())
            .toList();
    }
}

class EngagementRankedFeed implements FeedStrategy {
    public List<Post> generate(User viewer, List<Post> candidates) {
        return candidates.stream()
            .filter(viewer::canSee)
            .sorted(Comparator.comparingDouble(this::score).reversed())
            .toList();
    }
    private double score(Post p) {
        double recency = 1.0 / (1 + Duration.between(p.createdAt(), Instant.now()).toHours());
        return p.reactions().size() * 2 + p.comments().size() * 3 + recency * 10;
    }
}

interface PlatformObserver {
    void onPosted(Post p);
    void onCommented(Post p, Comment c);
    void onFriendRequest(FriendRequest fr);
}

class NotificationService implements PlatformObserver {
    public void onPosted(Post p) { /* notify friends if audience allows */ }
    public void onCommented(Post p, Comment c) { /* notify post author + thread participants */ }
    public void onFriendRequest(FriendRequest fr) { /* notify recipient */ }
}

class SocialNetwork {
    private final Map<String, User> users = new HashMap<>();
    private final List<Post> posts = new ArrayList<>();
    private FeedStrategy feedStrategy = new EngagementRankedFeed();
    private final List<PlatformObserver> observers = new ArrayList<>();

    public List<Post> feedFor(User u) {
        List<Post> candidates = posts.stream()
            .filter(p -> u.friends().contains(p.author().id()) || p.audience() == Audience.PUBLIC)
            .toList();
        return feedStrategy.generate(u, candidates);
    }

    public void post(User u, String content, Audience a) {
        Post p = new Post(u, content, Instant.now(), a);
        posts.add(p);
        observers.forEach(o -> o.onPosted(p));
    }
}
```

## Key design decisions

1. **Feed as Strategy** — feed ranking changes constantly (experiments, ML models, holiday hacks). It must be a swappable strategy, not embedded in `SocialNetwork`. This is the single highest-value abstraction in the design.
2. **Comments as Composite** — replies are themselves comments with their own replies. Uniform treatment makes recursive operations (count, render, mark-as-read) one method.
3. **Privacy enforced at read time** — `canSee(Post)` lives on the user (or a separate `PrivacyChecker`). Filtering at read time is simpler and safer than maintaining materialized per-user feed denormalized at write time (which has consistency problems).
4. **Observer for notifications** — the surface area of "things that should notify someone" grows constantly. Centralizing the observer hub keeps post/comment/friend code clean.
5. **Friendship as a symmetric set** — both users get the other's id in their `friendIds`. A `FriendRequest` is a *separate* entity with its own state; once accepted, it's discarded and the symmetric friendship persists.
6. **Audience modeled as enum + custom list** — `PUBLIC` / `FRIENDS` / `CUSTOM`. Custom carries an explicit user-id list. Keeps the privacy check small.

## Extensibility

**Add a new feed algorithm (ML-based):**

1. `class MlRankedFeed implements FeedStrategy` — uses a remote model service.
2. Inject (or feature-flag between strategies). Rest of the system unaware.

**Add a new content type (video):**

1. `Post` becomes abstract or generic, `VideoPost extends Post` with extra fields.
2. Feed strategies just consume `Post`; they don't care about subtype.

**Add reactions beyond "Like" (Love, Haha, Wow):**

1. `Reaction` becomes an enum or its own class with `type`.
2. Engagement scoring weights different reactions differently.

**Add groups / pages:**

1. New `Audience` value (`GROUP`), new `Group` entity owning a member list and its own posts.
2. Feed strategy gains group posts as candidates; privacy check extends to group membership.

**Add muting / blocking:**

1. `User.blocked: Set<String>`; feed strategy filters out blocked authors.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition of user/post/comment; recursive Comment models.
- [SOLID](../fundamentals/solid.md) — OCP via feed strategies, SRP across feed/notification/privacy.
- [Design Patterns](../fundamentals/design-patterns.md) — Observer, Strategy, Composite, Factory.
