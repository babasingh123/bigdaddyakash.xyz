# Design a Music Streaming Service (Spotify-style)

> **Time:** 50–60 minutes

## Problem

Design the object model for a music streaming service. The system must:

- Let users **play songs**, **pause/resume**, **skip**, **shuffle**, **repeat**.
- Let users **create playlists** and add/remove songs.
- **Search** songs, artists, albums.
- Provide **recommendations** (personalized: "discover weekly", "daily mix").
- Support **subscriptions** (Free with ads, Premium).
- Track **listening history** for analytics and recommendations.

## Real-world intuition

Spotify's core LLD challenge is keeping the **player state** (what's playing, what's next) decoupled from the **catalog** (what exists) and from **personalization** (what should you hear next). Done badly, the player class ends up knowing about recommendations and ads. Done well, the player just consumes a queue that any source (playlist, radio, recommendation engine) can fill.

## Key classes

- `User` — profile, subscription, playlists, listening history.
- `Song` — id, title, duration, artist, album, audio URL.
- `Album` — id, title, artist, list of songs.
- `Artist` — id, name, albums.
- `Playlist` — id, name, owner, ordered list of songs, visibility (private/public).
- `Player` — owns the current `Queue` and player state (`PLAYING`/`PAUSED`/`STOPPED`); exposes `play/pause/next/previous/seek`.
- `Queue` — ordered list of upcoming songs, plus a "currently playing" pointer; abstraction over playlist/radio/recommendation feeds.
- `Subscription` — type (`FREE`/`PREMIUM`), expiry, feature flags.
- `RecommendationEngine` — produces personalized song lists based on history.
- `RecommendationStrategy` (interface) — `DiscoverWeekly`, `DailyMix`, `MoreLikeThis`.
- `SearchService` — search index over songs/artists/albums.

## Design patterns used

- **Strategy (RecommendationStrategy, ShuffleStrategy)** — recommendations come in many flavors; shuffle algorithms differ (truly random vs anti-clustering). Strategy makes each independently swappable.
- **Observer (PlaybackObserver)** — listening history, ad-injection, analytics, and "now playing" status all need to react to playback events. Observer keeps the player code from drowning in side effects.
- **State (Player status)** — `PLAYING` → `PAUSED` → `STOPPED`. Each state handles `play()`, `pause()`, `next()` differently.
- **Composite (Playlist of playlists / "folder")** — Spotify lets users organize playlists into folders. A `Folder` containing `Playlist`s and other `Folder`s is Composite.

## Class diagram (sketch)

```text
   ┌────────┐ owns  ┌──────────┐ * ┌────────┐
   │  User  │◇─────│ Playlist │◇──│  Song  │
   └────┬───┘       └──────────┘   └────┬───┘
        │                               │ belongs to
        │ uses                          ▼
        ▼                          ┌────────┐
   ┌─────────┐ has-a ┌────────┐    │ Album  │◇──▶ Artist
   │ Player  │──────│ Queue  │    └────────┘
   ├─────────┤      └────────┘
   │ -state  │──▶ PlayerState «interface»  ─△─  Playing / Paused / Stopped
   │ -songs  │
   └─────────┘

   Player ──▶ PlaybackObserver «interface» (history, ads, analytics)

   RecommendationEngine ──▶ RecommendationStrategy «interface»
                                △
                                ┊
                DiscoverWeekly  DailyMix  MoreLikeThis
```

## Code sketch

```java
class Song {
    private final String id;
    private final String title;
    private final Duration length;
    private final Artist artist;
    private final Album album;
    private final URI audioUrl;
}

class Playlist {
    private final String id;
    private final String name;
    private final User owner;
    private final List<Song> songs = new ArrayList<>();
    private boolean isPublic;

    public void add(Song s)    { songs.add(s); }
    public void remove(Song s) { songs.remove(s); }
    public void reorder(int from, int to) { /* ... */ }
}

interface PlayerState {
    void play(Player p);
    void pause(Player p);
    void next(Player p);
}

class PlayingState implements PlayerState {
    public void play(Player p)  { /* already playing */ }
    public void pause(Player p) { p.audioPause(); p.setState(new PausedState()); }
    public void next(Player p)  { p.queue().advance(); p.audioStart(p.queue().current()); }
}

class Player {
    private PlayerState state = new StoppedState();
    private Queue queue = new Queue();
    private final List<PlaybackObserver> observers;

    public void play()  { state.play(this);  notifyObservers(); }
    public void pause() { state.pause(this); notifyObservers(); }
    public void next()  { state.next(this);  notifyObservers(); }
    public void setSource(List<Song> songs) { queue.set(songs); }

    void setState(PlayerState s) { this.state = s; }
    Queue queue()                { return queue; }

    private void notifyObservers() {
        observers.forEach(o -> o.onStateChange(this));
    }
}

interface RecommendationStrategy {
    List<Song> recommend(User u, int n);
}

class DiscoverWeeklyStrategy implements RecommendationStrategy {
    public List<Song> recommend(User u, int n) {
        // ML-driven: similar users, novel genres, anti-repeat from history.
        return List.of();
    }
}

class RecommendationEngine {
    private final Map<String, RecommendationStrategy> strategies = new HashMap<>();

    public List<Song> recommend(User u, String type, int n) {
        return strategies.get(type).recommend(u, n);
    }
}

class HistoryObserver implements PlaybackObserver {
    public void onStateChange(Player p) {
        if (p.state() == PlayerState.PLAYING) {
            // record current song in user's history
        }
    }
}

class AdInjectionObserver implements PlaybackObserver {
    public void onStateChange(Player p) {
        if (p.user().subscription().isFree() && shouldShowAd()) {
            p.queue().insertNext(adService.nextAd());
        }
    }
    private boolean shouldShowAd() { /* every N songs */ return false; }
}
```

## Key design decisions

1. **Queue abstraction in the player** — the player doesn't care if songs came from a playlist, radio, or recommendation; it just consumes a queue. This means new sources (album, station, ML-mix) don't require player changes.
2. **Recommendation strategies are interchangeable** — Discover Weekly today, a neural-net model tomorrow. Strategy pattern keeps the engine generic and the strategies independently testable.
3. **Observer for cross-cutting concerns** — listening history, ad insertion, analytics, social ("friend is listening to") all subscribe to playback events. None of them lives in the player's core logic.
4. **State-driven player** — `PLAYING.pause()` and `STOPPED.pause()` mean different things. State pattern eliminates a tangle of `if (status == PLAYING)` checks.
5. **Subscription as a separate entity** — feature gating (ad-free, offline downloads, hi-fi audio) reads from the `Subscription` flags. Player consults but doesn't store policy.
6. **Search service is its own class** — keeps the catalog (songs/artists/albums) separate from the search index. Today an in-memory map; tomorrow Elasticsearch.

## Extensibility

**Add podcasts:**

1. `Playable` interface; both `Song` and `Podcast` implement it.
2. `Queue` holds `Playable`s, not `Song`s. Player code unchanged.
3. Podcasts gain extras (chapters, speed control) via `PodcastPlayer extends Player` or via specialized observers.

**Add collaborative playlists:**

1. `Playlist` gains a `Set<User> collaborators`.
2. Permission check before `add`/`remove` consults the collaborator set.

**Add offline mode:**

1. `Song.audioUrl` becomes a resolver — local cache first, network second.
2. New `DownloadObserver` pre-fetches songs from the user's playlists.

**Add a new recommendation strategy (ML-based, "Daily Mix"):**

1. `class DailyMixStrategy implements RecommendationStrategy`.
2. Register in the engine. Frontend can request it by name.

**Add social ("friends are listening to..."):**

1. `SocialObserver` subscribes to playback; aggregates per-user "now playing".
2. Privacy: respect user setting in `Subscription` or `User.privacy`.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composition (player → queue → songs), state.
- [SOLID](../fundamentals/solid.md) — OCP via strategies; SRP across player/queue/recommendations/search.
- [Design Patterns](../fundamentals/design-patterns.md) — Strategy, Observer, State, Composite.
