# Search & Indexing

Search is what you reach for when "look up by primary key" isn't enough — when users want to find content by typing partial words, filter on many dimensions at once, find places near a location, or have results ranked by relevance. The core data structure is the **inverted index**; the canonical tools are Elasticsearch and Solr; and the geospatial cousins are QuadTrees, R-trees, and geohashes.

## Why it exists

A relational database with a B-tree index on a `name` column can answer "name equals 'alice'" or "name between 'a' and 'b'" in milliseconds. It cannot efficiently answer:

- "Find documents that *contain* the words `quick` and `fox` in any order."
- "Find users whose name *sounds like* 'jhonson'."
- "Find restaurants within 2 km of (37.77, -122.42)."
- "Find products matching 'red running shoes' ranked by relevance."

For each of these, a B-tree gives you no useful structure. You'd full-scan and pay milliseconds-per-row across millions of rows. Specialized indexes — **inverted indexes**, **tries**, **R-trees**, **geohashes** — turn each of these into sub-millisecond lookups by encoding *the right structure* for the query.

A separate search engine (Elasticsearch, etc.) also keeps that workload off your OLTP database. Heavy text search on Postgres is possible but it competes with transactional traffic. A dedicated search cluster scales independently.

## How it works

### Full-text search

The data structure that makes search work is the **inverted index**.

#### Inverted index

```
Document 1: "The quick brown fox"
Document 2: "The lazy brown dog"

Inverted Index:
"the"   → [1, 2]
"quick" → [1]
"brown" → [1, 2]
"fox"   → [1]
"lazy"  → [2]
"dog"   → [2]
```

For each unique token, the index records which documents contain it. A query "brown dog" intersects the postings lists for `brown` ([1, 2]) and `dog` ([2]) → result: [2].

Real inverted indexes are richer:

- Each posting includes **term positions** (so you can answer phrase queries: "brown fox" must have those words adjacent).
- Each posting tracks **term frequency** for scoring.
- The lexicon is compressed (front-coding, FST), postings are compressed (delta-encoding, varint).

#### Text analysis pipeline

Before indexing, text goes through an analyzer:

| Step | Purpose |
|------|---------|
| **Tokenization** | Split text into tokens. "Hello, world!" → ["hello", "world"]. Handles punctuation, case folding, Unicode. |
| **Stop words** | Drop very common, low-signal words ("the", "a", "is"). Sometimes; modern engines often keep them. |
| **Stemming / Lemmatization** | Reduce variants to a root form. "running" → "run", "ran" → "run". Aggressive (Porter) or precise (lemmatizer). |
| **Synonyms** | Map "auto" ↔ "automobile", "nyc" ↔ "new york". |
| **N-grams / Edge n-grams** | Break tokens into character substrings for typo tolerance and autocomplete. |

#### Ranking

Once you've found matching documents, you rank them by relevance.

- **TF-IDF** — `term frequency × inverse document frequency`. Rare words across the corpus that appear often in this document score high.
- **BM25** — refinement of TF-IDF; the default in Elasticsearch and Solr. Less sensitive to term frequency saturation.
- **Vector / neural ranking** — embed documents and queries into vectors; rank by cosine similarity. Increasingly common (semantic search).
- **Custom scoring functions** — combine relevance with business signals (recency, popularity, paid placement).

#### Other features you'll likely need

- **Fuzzy matching** — Levenshtein-distance-tolerant lookups for typos. "jhonson" → "johnson".
- **Phrase queries** — terms must appear adjacent in this order.
- **Boolean operators** — `AND`, `OR`, `NOT`.
- **Filters & facets** — narrow results by category, range, etc. Often implemented as bitset intersections.
- **Highlighting** — return snippets with matched terms highlighted.
- **Autocomplete / suggestions** — see [Search Autocomplete](../problems/search-autocomplete.md).

### Tools

- **Elasticsearch** — distributed search and analytics engine on top of Lucene. The de facto choice. Rich query DSL, aggregations, geospatial, clustering. Powers logs (ELK), product search, full-text, observability.
- **Apache Solr** — also Lucene-based. Older, mature, slightly more "configurable XML" oriented. Strong choice when you want fine control.
- **OpenSearch** — Elasticsearch fork after the licensing dispute; API-compatible with older versions.
- **Algolia / Typesense** — managed search, easy integration, very fast for catalog/product search. Pay-as-you-go.
- **Postgres `tsvector` / MySQL FULLTEXT** — built-in full-text search inside the SQL database. Fine for small-scale needs; doesn't scale like a dedicated engine.

### Geospatial indexing

The "find nearby" class of queries needs a different shape of index entirely.

#### The problem

Given coordinates `(lat, lng)` for millions of objects, find all objects within radius `r` of a query point. Naive: full scan, compute distance for each → unacceptable at any scale.

#### Data structures

##### QuadTree

Recursively divide 2D space into four quadrants. A node holds points until it overflows a threshold, then splits.

```
   ┌────┬────┐
   │ NW │ NE │
   ├────┼────┤
   │ SW │ SE │
   └────┴────┘
```

Each quadrant subdivides further as more points are added. Searching "find points within radius r of (lat, lng)" traverses only the quadrants that intersect the query circle.

- **Pros**: Adapts to data density. Easy to understand.
- **Cons**: Unbalanced trees for skewed data. Tricky to manage updates.

##### R-tree

Like a B-tree, but for rectangles ("minimum bounding rectangles" of subtrees). Used to index polygons, lines, or points. PostGIS, Mapbox, and Oracle Spatial use R-tree variants.

##### Geohash

Encode `(lat, lng)` as a string by interleaving bits of lat and lng and base32-encoding the result.

```
(37.7749, -122.4194)  → "9q8yyk8yu"
```

Two points that share a prefix are geographically close. "9q8yy*" covers a small bounding box around San Francisco.

- **Pros**: Indexes nicely in any string-keyed store (Redis sorted sets, Cassandra, key-value).
- **Cons**: Boundary effects — two adjacent points can have very different geohashes if they straddle a grid line. Mitigation: query the 8 neighboring geohash cells.

##### H3 (Uber's hexagonal indexing)

Uber's open-source alternative: tile the Earth with hexagons at multiple resolutions. Hexagons have nicer adjacency properties than rectangles. Used internally at Uber for matching, surge pricing, and ETA.

#### Tools

- **PostGIS** — Postgres extension with R-tree indexes and full geospatial query support.
- **MongoDB geospatial indexes** — `2dsphere` indexes for points on a sphere.
- **Elasticsearch geo queries** — `geo_point`, `geo_shape`, `geo_distance` aggregations.
- **Redis GEO commands** — `GEOADD`, `GEORADIUS` on top of sorted sets with geohash scores.
- **Uber's H3** — library for hex-based indexing.

### Indexing pipeline

Search indexes are typically *near* the source of truth, not the source of truth itself.

```
       App writes
           │
     ┌─────▼─────┐
     │ Primary DB│  ← source of truth (Postgres, MySQL)
     └─────┬─────┘
           │ change feed (CDC) / dual-write
     ┌─────▼─────┐
     │  Kafka    │
     └─────┬─────┘
           │
   ┌───────▼───────┐
   │ Elasticsearch │  ← search index
   └───────────────┘
```

Two common patterns:

- **Dual-write** — the application writes to the DB and the search engine in the same transaction. Simple, but cross-system consistency is awkward.
- **Change Data Capture (CDC)** — a connector reads the DB's binlog/WAL and emits change events to Kafka, which feeds the search index. Cleaner separation; eventually consistent.

## When to use

Reach for a dedicated search engine when:

- **Full-text search** with relevance, fuzzy matching, faceting.
- **Filter + sort over many dimensions** that no B-tree composite index can cover cleanly.
- **Log search / observability** (ELK stack).
- **Catalog search** (e-commerce).
- **Geospatial queries** at scale.
- The DB's search features no longer keep up.

Skip and stick with your DB when:

- You only need exact lookups by indexed columns.
- The data set is small enough that Postgres `tsvector` is plenty.
- The team can't operate another stateful system. (Be honest about operational cost.)

## Trade-offs

- **Eventually consistent** vs. the source of truth. New documents take seconds to appear in the index. Design UI around it ("new posts may take a moment to appear in search").
- **Operational complexity.** A search cluster is one more thing to monitor, secure, back up, and tune. JVM-heavy stacks (Elasticsearch) need careful capacity planning.
- **Relevance tuning is its own discipline.** Out-of-the-box BM25 is okay; tuning analyzers, synonyms, scoring, and field boosts is ongoing product work.
- **Storage cost.** Inverted indexes are dense — often 50–150% of the source data size, depending on what you index and how aggressively you store positions and term vectors.
- **Mapping changes can be expensive.** Reindexing a TB-scale Elasticsearch index is hours-to-days of background work. Plan schema carefully.

!!! warning "Search isn't a source of truth"
    Don't put data into Elasticsearch *only*. It's an index, not a database. Re-indexable from the source whenever needed. The day a node loses its disk, you'll want that boring rule already in place.

## Linked problems

- [Search Autocomplete](../problems/search-autocomplete.md) — trie + popularity scoring + caching; the most concentrated search problem.
- [Twitter](../problems/twitter.md) — Elasticsearch on tweet content for full-text search.
- [YouTube](../problems/youtube.md) — Elasticsearch on video metadata + autocomplete.
- [Instagram](../problems/instagram.md) — search on usernames, hashtags, locations.
- [Uber](../problems/uber.md) — geospatial indexing (geohash / QuadTree / H3) is the heart of driver matching.
- [Dropbox](../problems/dropbox.md) — file content search via Elasticsearch.
- [Web Crawler](../problems/web-crawler.md) — Elasticsearch as the destination index for crawled pages.
