# Autocomplete — Backend System Design

Autocomplete is one of the most frequently asked system design questions because it touches on data structures, caching, ranking, and scalability — all in a single problem. This document covers the backend design at the same depth as the frontend counterpart.

## Question
Design the backend service that powers an autocomplete/typeahead feature. Given a partial query string, the service should return a ranked list of suggestions with low latency (< 100ms).

Some real-life examples:
- Google Search suggestions (~10 results, text-based, ranked by popularity and personalization)
- Amazon product search (rich results with images, prices, categories)
- Slack's user/channel search (scoped to a workspace, real-time membership data)

## Requirements

### Functional Requirements
- Given a prefix string, return top-N matching suggestions
- Results should be ranked by relevance (exact match > prefix match > popularity)
- Support different types of searchable entities (text, products, users, etc.)
- Pagination for extended results

### Non-Functional Requirements
- **Low latency**: P99 < 100ms — users expect instant feedback as they type
- **High availability**: Search is a critical path; downtime = unusable product
- **Scalability**: Handle 10K+ queries per second (each keystroke is a query)
- **Consistency**: Eventual consistency is acceptable — new items can take seconds to appear in suggestions

### Requirements exploration
These are questions you should be asking your interviewer to dive deeper into the problem.

#### How large is the dataset?
This fundamentally changes the architecture. 1K items → in-memory array is fine. 1M items → need a Trie or inverted index. 1B items → need distributed search (Elasticsearch cluster).

#### How often does the dataset change?
- Rarely (city names, country codes): Build index once, serve from memory
- Moderately (products, users): Rebuild index periodically or use incremental updates
- Continuously (trending topics, stock prices): Need real-time indexing pipeline

#### Do we need personalization?
Personalized results (e.g., Google showing your recent searches) require per-user data and add significant complexity. For the initial design, we'll focus on global popularity-based ranking.

#### What is the expected query volume?
A search box on a high-traffic site with debouncing at 300ms means roughly 3-4 API calls per search session. At 1M concurrent users, that's 3-4M QPS — which requires careful caching and load balancing.

## Architecture

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │ GET /api/search?q=foo&limit=10
                           ▼
                    ┌─────────────┐
                    │  API Layer  │  ← Rate limiting, validation, CORS
                    │   (Hono)    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Cache Layer│  ← LRU Cache (in-memory)
                    │             │     Check cache before searching
                    └──────┬──────┘
                           │ cache miss
                    ┌──────▼──────┐
                    │Search Service│ ← Trie traversal + ranking
                    │             │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Data Store │  ← In-memory dataset (or DB at scale)
                    │  (Trie)     │
                    └─────────────┘
```

### Component Responsibilities

- **API Layer**: HTTP routing, input validation, rate limiting, CORS, response formatting. This is the thin "shell" — no business logic here.
- **Cache Layer**: Stores recent query→results mappings. Implements LRU eviction with TTL. Cache-aside pattern: check cache first, on miss → search → store result.
- **Search Service**: The "brain" of the backend. Accepts a prefix query, traverses the Trie, collects matching items, ranks them, and returns top-N.
- **Data Store (Trie)**: The primary data structure holding all searchable items. Built once on startup from the dataset. Supports efficient prefix lookups in O(L + K) where L = prefix length, K = number of matches.

## Data Model

### Searchable Item
```typescript
interface SearchItem {
  id: number;
  name: string;            // Primary searchable text
  category: string;        // "city", "company", "technology", etc.
  popularity: number;      // 0-100, used for ranking
  metadata?: {             // Optional extra data for rich results
    subtitle?: string;
    imageUrl?: string;
  };
}
```

### Trie Node
```typescript
interface TrieNode {
  children: Map<string, TrieNode>;  // character → child node
  itemIds: Set<number>;             // IDs of items that pass through this node
  isEndOfWord: boolean;             // marks complete word boundary
}
```

### Cache Entry
```typescript
interface CacheEntry<T> {
  value: T;
  createdAt: number;       // timestamp for TTL checks
  accessedAt: number;      // timestamp for LRU ordering
}
```

## Interface Definition (API)

### Search Endpoint

```
GET /api/search?q={query}&limit={limit}&delay={delay}
```

| Parameter | Type   | Required | Default | Description                                      |
|-----------|--------|----------|---------|--------------------------------------------------|
| `q`       | string | Yes      | —       | The search prefix query                          |
| `limit`   | number | No       | 10      | Maximum number of results to return              |
| `delay`   | number | No       | 0       | Artificial delay in ms (for testing loading UX)  |

#### Response
```typescript
interface SearchResponse {
  query: string;
  results: SearchResult[];
  meta: {
    total: number;          // Total matches found
    took: number;           // Time taken in ms
    cached: boolean;        // Whether result was served from cache
  };
}

interface SearchResult {
  id: number;
  name: string;
  category: string;
  highlight: {              // Which part of the name matched
    start: number;          // Start index of match
    end: number;            // End index of match
  };
  score: number;            // Relevance score (for transparency/debugging)
}
```

#### Error Response
```typescript
interface ErrorResponse {
  error: string;
  code: string;             // Machine-readable error code
}
```

| Status Code | When                                    | Error Code           |
|-------------|----------------------------------------|----------------------|
| 400         | Missing `q` parameter or invalid input | `INVALID_QUERY`      |
| 429         | Rate limit exceeded                    | `RATE_LIMITED`        |
| 500         | Internal server error                  | `INTERNAL_ERROR`     |

## Deep Dive: Data Structures

### Option 1: Linear Scan (Array)
The simplest approach — iterate through all items and filter by prefix match.

```typescript
items.filter(item => item.name.toLowerCase().startsWith(query.toLowerCase()));
```

- **Time**: O(N) per query where N = dataset size
- **Space**: O(N)
- **Pros**: Dead simple, no preprocessing needed
- **Cons**: Too slow for large datasets. At 1M items, every keystroke scans 1M strings
- **When to use**: Dataset < 1,000 items, or as a quick prototype

### Option 2: [Trie](../../concepts/backend/trie.md) (Prefix Tree)
A tree data structure optimized for prefix lookups. Each node represents a character, and paths from root to nodes represent prefixes.

```
          root
         / | \
        a  s  t
       /   |
      p    a
     / \   |
    p   i  n
    |       \
    l        (San Francisco, San Jose, ...)
    |
    e  (Apple)
```

- **Time**: O(L + K) per query where L = prefix length, K = number of matches
- **Space**: O(N × M) where N = items, M = average name length
- **Pros**: Very fast prefix lookups, naturally suited for autocomplete
- **Cons**: High memory usage due to pointer overhead, not cache-friendly (CPU cache, not our app cache)
- **When to use**: Dataset up to ~10M items, single-server deployment

### Option 3: [Inverted Index](../../concepts/backend/database-indexing.md)
Map each possible prefix of every item to a list of item IDs. Pre-compute all prefixes at index time.

```typescript
// For "Apple": index "a", "ap", "app", "appl", "apple"
const index = {
  'a': [1, 5, 12],     // Apple, Amazon, Airbnb
  'ap': [1],            // Apple
  'app': [1],           // Apple
  'am': [5],            // Amazon
  // ...
};
```

- **Time**: O(1) lookup (hash map)
- **Space**: O(N × M²) — much higher than Trie due to all prefix combinations
- **Pros**: Constant-time lookup, simple implementation
- **Cons**: Very high memory for large datasets, expensive to update
- **When to use**: Small-to-medium datasets where lookup speed is paramount

### Which structure to use?

| Factor              | Linear Scan | Trie          | Inverted Index |
|---------------------|-------------|---------------|----------------|
| Dataset size        | < 1K        | < 10M         | < 100K         |
| Lookup speed        | O(N)        | O(L + K)      | O(1)           |
| Memory usage        | Low         | Medium-High   | Very High      |
| Update cost         | O(1)        | O(L)          | O(M)           |
| Implementation      | Trivial     | Moderate      | Simple         |
| Best for            | Prototypes  | General use   | Read-heavy     |

For our implementation, we'll use a **Trie** — it's the most common interview answer, offers the best balance of performance and memory, and naturally fits the autocomplete use case.

## Deep Dive: [Caching](../../concepts/caching.md)

### Why cache on the backend?
Even with an efficient Trie, there are benefits to server-side caching:
1. **Avoid repeated Trie traversal**: Popular queries like "fac" (Facebook) are searched thousands of times
2. **Reduce CPU usage**: Ranking and sorting are expensive; cache the final sorted result
3. **Consistent response times**: Cache hits return in ~0ms vs ~1-5ms for Trie traversal

### [Cache-Aside Pattern](../../concepts/caching.md)
This is the standard caching pattern for read-heavy workloads:

```
1. Client sends query "san"
2. Server checks cache for "san"
   → HIT: return cached results immediately
   → MISS: search Trie → rank results → store in cache → return results
```

### [LRU Cache](../../concepts/backend/lru-cache.md) Design
We implement an LRU (Least Recently Used) cache from scratch — a classic interview question on its own.

**Data structure**: Doubly-linked list + Hash map
- **Hash map**: O(1) lookup by key (query string)
- **Doubly-linked list**: O(1) move-to-front on access, O(1) evict from tail

```
Hash Map                   Doubly Linked List (most recent → least recent)
┌──────────┐
│ "san" ───┼──────→  [san] ←→ [new] ←→ [app] ←→ [goo] ←→ [fac]
│ "new" ───┼──────→    ↑                                      ↑
│ "app" ───┼──────→   HEAD                                   TAIL
│ "goo" ───┼──────→        (evict from tail when full)
│ "fac" ───┼──────→
└──────────┘
```

**Operations**:
- `get(key)`: Look up in hash map → if found, move node to head → return value
- `set(key, value)`: If exists, update and move to head. If full, evict tail. Insert at head.
- **TTL**: Each entry has a timestamp. On `get()`, check if expired → treat as miss if stale.

### Cache Invalidation
The hardest problem in computer science (after naming things). For autocomplete:

- **TTL-based**: Simplest approach. Each cache entry expires after N seconds. Good enough for most cases.
- **Event-driven**: When the dataset changes, invalidate affected cache entries. More complex but ensures freshness.
- **Hybrid**: Short TTL (e.g., 60s) + event-driven invalidation for critical updates.

For our implementation, we use TTL-based invalidation with a configurable duration.

### At Scale: Redis
In a production multi-server deployment, an in-memory per-process cache won't share state across servers. Solutions:
- **Redis**: Centralized cache shared by all API servers. Supports TTL natively. Adds ~1ms network latency.
- **Two-tier caching**: Process-local LRU (L1) + Redis (L2). Check L1 first, then L2, then search.

## Deep Dive: [Ranking](../../concepts/ranking-and-scoring.md)

Not all matches are equal. "san" could match "San Francisco", "Santa Cruz", "Samsung", "Sandwich". How do we rank them?

### Scoring Formula
```
score = (exactMatchBonus × 100) + (prefixMatchBonus × 50) + (popularity × 1)
```

- **Exact match**: Query equals the item name → highest priority
- **Prefix match**: Query matches the start of a word in the item name (e.g., "fran" matches "San **Fran**cisco")
- **Popularity**: A pre-computed score (0-100) based on search volume, click-through rate, etc.

### At Scale: Learning to Rank
Production systems use ML models that consider:
- User's search history and context
- Geographic location
- Time of day (trending topics)
- Click-through rates on previous suggestions
- Query frequency across all users

This is far beyond interview scope but worth mentioning to show awareness.

## Deep Dive: Performance Optimization

### Request Validation
Reject bad queries early to avoid wasting resources:
- Empty queries → return initial/trending results
- Queries that are too long (> 100 chars) → reject with 400
- Queries with special characters → sanitize or reject

### Response Size
Keep responses lean:
- Return only the fields needed for rendering (don't send the entire item object)
- Limit results (default 10, max 50)
- Use gzip/brotli compression for the HTTP response

### Connection Handling
- **Keep-Alive**: Reuse TCP connections (HTTP/2 does this by default)
- **Timeouts**: Set aggressive server-side timeouts (e.g., 500ms) — if the search takes longer, something is wrong

## Deep Dive: Scalability

### [Read Replicas](../../concepts/backend/replication.md)
Autocomplete is overwhelmingly read-heavy. At scale:
- One primary server handles writes (dataset updates, re-indexing)
- Multiple read replicas serve search queries
- Load balancer distributes queries across replicas

### [Sharding](../../concepts/backend/sharding.md)
For very large datasets, shard by prefix:
- Server A handles a-m
- Server B handles n-z
- A routing layer directs queries to the right shard

### [CDN](../../concepts/backend/cdn.md)/Edge Caching
Popular queries (top 1000) can be cached at the CDN edge:
- "weather", "facebook", "amazon" → served from CDN with ~5ms latency
- Long-tail queries → served from origin servers

### Pre-computation
For the top 1000 most popular prefixes, pre-compute and store the results. Update every few minutes. This eliminates Trie traversal for the majority of queries.

## Deep Dive: Reliability

### [Rate Limiting](../../concepts/rate-limiting.md)
Autocomplete endpoints are abuse targets (scraping, DDoS). Implement:
- **Per-IP rate limiting**: e.g., 100 requests/minute
- **Token bucket algorithm**: Allows burst traffic while enforcing average rate
- **Sliding window**: More accurate than fixed windows, prevents burst at window boundaries

### Graceful Degradation
When the system is under extreme load:
1. Return only cached results (skip Trie search)
2. Reduce result count (return 3 instead of 10)
3. Increase debounce hint to clients via response headers
4. Shed traffic from non-critical paths

### [Monitoring](../../concepts/monitoring-and-observability.md)
Key metrics to track:
- **P50/P95/P99 latency**: Are we meeting the <100ms target?
- **Cache hit rate**: Should be >80% in steady state. Low hit rate → cache is too small or TTL too short
- **Query volume**: QPS trends, anomaly detection for DDoS
- **Error rate**: 5xx responses indicate backend issues

## Comparison: Real-World Autocomplete Backends

| Aspect           | Google              | Elasticsearch        | Algolia              | Typesense           |
|------------------|---------------------|----------------------|----------------------|---------------------|
| Data structure   | Custom (undisclosed)| Inverted index + FST | Custom trie variant  | In-memory trie      |
| Latency          | ~30ms               | ~10-50ms             | ~1-20ms              | ~5-50ms             |
| Fuzzy search     | Yes (ML-based)      | Yes (edit distance)  | Yes (typo tolerance) | Yes (edit distance) |
| Ranking          | ML (200+ signals)   | BM25 + custom        | Custom tie-breaking  | Popularity + text   |
| Personalization  | Heavy               | Manual               | Manual               | Manual              |
| Scale            | Billions of queries | Self-hosted clusters | Managed SaaS         | Self-hosted/cloud   |
| Highlighting     | Yes                 | Yes                  | Yes                  | Yes                 |

## References
- [The Life of a Typeahead Query — Facebook Engineering](https://engineering.fb.com/2010/05/17/web/the-life-of-a-typeahead-query/)
- [How We Built Prefixy — A Scalable Prefix Search Service](https://medium.com/@prefixyteam/how-we-built-prefixy-a-scalable-prefix-search-service-for-powering-autocomplete-c20f98e2eff1)
- [System Design: Autocomplete/Typeahead — Karan Pratap Singh](https://www.karanpratapsingh.com/courses/system-design/typeahead)
- [Elasticsearch Completion Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester)
- [Trie Data Structure — Wikipedia](https://en.wikipedia.org/wiki/Trie)
