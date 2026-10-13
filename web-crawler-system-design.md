# Production-Scale Web Crawler

## 1. Requirements

### Functional requirements

1. Discover and fetch public HTTP/HTTPS pages from seed URLs and links.
2. Respect `robots.txt`, crawl-delay policy, and per-origin rate limits.
3. Canonicalize URLs and avoid duplicate fetches and duplicate content.
4. Prioritize fresh, important, and frequently changing pages.
5. Parse HTML, extract links and metadata, and store content for downstream indexing.
6. Retry transient failures without creating retry storms.
7. Recrawl pages according to observed change rate.
8. Support crawler operators adding seeds, exclusions, and priority overrides.

### Non-functional requirements

- Crawl billions of pages while never overwhelming a host.
- Sustain high aggregate throughput with bounded per-origin concurrency.
- At-least-once scheduling with idempotent processing.
- Recover from worker, queue, region, and storage failures.
- Make crawl decisions explainable and auditable.
- Isolate hostile content and protect internal networks.

Out of scope: ranking search results, rendering every JavaScript application, and crawling authenticated/private content.

## 2. Scale assumptions

- 10 billion known canonical URLs; 3 billion retained pages.
- 1 billion fetches/day: about 11,600/second average and 50,000/second peak.
- Average compressed response: 500 KB; roughly 500 TB/day ingress.
- Average page exposes 30 links before filtering.
- 100 million origins, with a highly skewed popularity distribution.
- 30-day average revisit interval, shortened to minutes for news and lengthened to months for static pages.
- Maximum response retained: 20 MB; maximum redirect chain: 10.

Bandwidth, parsing, and storage dominate cost. The global rate is less difficult than safely scheduling millions of independent origin-specific streams.

## 3. High-level architecture

```mermaid
flowchart LR
    OPS[Seeds / Operator Rules] --> ING[URL Ingestion]
    ING --> CAN[Canonicalizer]
    CAN --> DEDUP[(URL Seen Store)]
    DEDUP --> FRONT[Distributed Crawl Frontier]
    FRONT --> POL[Politeness Scheduler]
    POL --> FETCH[Sandboxed Fetch Workers]
    FETCH --> DNS[DNS / Egress Proxy]
    FETCH --> RAW[(Content Object Store)]
    FETCH --> META[(Crawl Metadata DB)]
    FETCH --> PARSE[Parser / Link Extractor]
    PARSE --> CAN
    PARSE --> INDEX[Downstream Index Pipeline]
    META --> REC[Recrawl Planner]
    REC --> FRONT
    ROB[Robots Cache] <--> POL
    ROB <--> FETCH
    FRONT --> DLQ[Dead-letter Queue]
```

The frontier is not a single FIFO. It combines a priority queue of origins with a per-origin queue so global importance never bypasses host politeness.

## 4. Data model

```text
UrlRecord
- url_id, canonical_url, origin_id, url_hash
- discovery_time, discovered_from_url_id
- priority, next_fetch_at, state, retry_count

Origin
- origin_id, scheme, host, port
- next_allowed_at, max_concurrency, crawl_delay_ms
- robots_version, robots_expires_at, health_state

FetchAttempt
- attempt_id, url_id, scheduled_at, started_at, completed_at
- worker_id, status_code, error_class, redirect_target
- bytes, latency_ms, content_hash, lease_token

PageVersion
- url_id, fetched_at, content_hash, object_key
- mime_type, language, etag, last_modified
- parse_status, canonical_target

RobotsPolicy
- origin_id, user_agent, allowed_rules, disallowed_rules
- crawl_delay_ms, fetched_at, expires_at
```

Use normalized canonical URLs for identity, but retain discovered and final URLs for diagnostics. Content hashes identify byte-level duplicates; similarity fingerprints such as SimHash identify near duplicates.

## 5. Core APIs

```http
POST /v1/seeds
{"urls":["https://example.com/"],"priority":"HIGH","max_depth":5}

POST /internal/v1/frontier/lease
{"worker_id":"fetch-42","capacity":100}

POST /internal/v1/fetches/{attempt_id}/complete
{"lease_token":"...","status":200,"content_hash":"sha256:...","links_object_key":"..."}

GET /v1/urls/{url_id}/history
POST /v1/origins/{origin_id}/pause
PUT /v1/origins/{origin_id}/policy
```

Lease and completion calls are idempotent. Completion validates that the attempt still owns the lease; a late worker cannot overwrite a newer result.

## 6. URL ingestion and canonicalization

1. Resolve relative links against the document base URL.
2. Accept only configured schemes and valid IDNA hostnames.
3. Lowercase scheme/host, remove default ports and fragments, normalize dot segments, and normalize percent encoding.
4. Apply conservative query-parameter rules; never globally remove parameters because they may change content.
5. Check allow/deny policy and depth limits.
6. Insert into the seen store using an atomic put-if-absent.
7. If new, enqueue with source-derived priority; if known, update discovery signals without duplicating work.

Canonical tags and redirects are hints, not blindly trusted identity changes. They can be malicious or cyclic.

## 7. Crawl frontier and politeness

Maintain:

- **Per-origin ready queues** ordered by URL priority and discovery time.
- **Origin eligibility heap** ordered by `next_allowed_at`.
- **Leases** for origins and URLs with expiry.

The scheduler pops an eligible origin, verifies robots policy and concurrency, leases one or more URLs, then advances `next_allowed_at`. An origin is owned by one scheduler shard at a time, preventing two workers from independently violating delay limits.

Priority can combine PageRank-like importance, seed distance, content type, staleness, change probability, and operator overrides. Aging prevents low-priority starvation.

## 8. Fetch and parse flow

1. Worker receives a leased URL plus resolved policy.
2. It resolves DNS through a guarded resolver and validates every redirect target.
3. It fetches with strict connect, header, body, and total deadlines.
4. Streaming limits stop oversized or decompression-bomb responses.
5. Raw bytes are written under a content hash; metadata commit references only durable objects.
6. Parsers selected by MIME type extract links, canonical hints, language, and indexing fields.
7. New URLs return through canonicalization.
8. Completion updates origin health and schedules recrawl.

HTML parsing runs in resource-limited sandboxes. JavaScript rendering is a separate scarce queue used only for selected pages.

## 9. Deduplication

Three layers serve different purposes:

- URL seen set prevents repeated scheduling of the same canonical URL.
- SHA-256 content hash avoids storing exact duplicate bodies.
- SimHash or MinHash groups mirrors and templated near duplicates for indexing.

A Bloom filter can reject most known URLs cheaply, but false positives must not permanently suppress discovery. Use it only as a cache before an authoritative LSM/KV seen store, or periodically sample positives against the store.

## 10. Consistency and concurrency

- Queue delivery is at least once; fetching and completion are idempotent.
- Seen-set insertion is atomic by canonical URL hash.
- Lease fencing tokens prevent expired workers from committing stale ownership.
- Origin scheduler state is strongly consistent within an origin shard.
- Page bodies are immutable and eventually replicated.
- Metadata publication uses a database transaction plus outbox.

Exactly-once network fetching is impossible after ambiguous failures. Correctness means duplicate attempts do not corrupt frontier state or downstream indexes.

## 11. Partitioning and storage

- Partition origin scheduling by consistent hash of `origin_id`; all URLs for an origin share a shard.
- Subpartition large origin queues by priority bucket while preserving one politeness governor.
- Partition the seen store by URL hash for even distribution.
- Store immutable bodies in object storage with lifecycle tiers.
- Store recent crawl metadata in an LSM database; compact old attempts to analytical storage.

Hot origins are intentionally capped, not spread to gain unconstrained throughput. Scheduler shard splitting moves groups of origins, never halves one origin's policy state.

## 12. Recrawl algorithm

Estimate each URL's change interval from recent fetches, `ETag`/`Last-Modified`, sitemap hints, and page class. A simple policy is:

```text
next_interval = clamp(min_interval, max_interval,
                      previous_interval * adjustment(change_detected))
```

Shorten after change, lengthen after repeated stability, add jitter, and prioritize expected freshness value per fetch cost. `304 Not Modified` still consumes host capacity but saves bandwidth and parsing.

## 13. Reliability and failure handling

| Failure | Handling |
|---|---|
| Worker dies after lease | Lease expires; another worker retries |
| Completion response is lost | Idempotent completion returns prior result |
| DNS or TLS transient failure | Exponential backoff with jitter and retry budget |
| Origin returns 429/503 | Honor `Retry-After`, reduce concurrency, open circuit |
| Robots cache unavailable | Fail closed for unknown policy; use unexpired cached policy |
| Parser crashes | Isolate document, record failure, continue fetch pipeline |
| Queue shard fails | Replicated log elects a leader and replays leases |
| Object write succeeds but metadata fails | Unreferenced object expires through lifecycle GC |
| Poison URL retries forever | Bounded retries then dead-letter with reason |
| Link explosion/trap | Depth, pattern, per-site budget, and calendar/session detection |

## 14. Multi-region design

Assign each origin a home region so only one region controls politeness. Fetch near the origin when legal and useful, but transfer ownership through a fenced epoch. Replicate the seen store and content asynchronously; route new discoveries to the origin's owner. A region outage pauses affected origins until a secondary acquires a higher epoch, preferring temporary staleness over accidental traffic multiplication.

Respect data residency for stored page bodies and regional restrictions on crawling. Global seeds and policy updates use a durable replicated control plane.

## 15. Security and abuse prevention

- Block loopback, link-local, private, metadata-service, and internal ranges before and after DNS resolution.
- Revalidate every redirect to prevent SSRF and DNS rebinding.
- Run fetchers and parsers without ambient credentials in isolated networks.
- Restrict egress ports and protocols; cap response size and decompression ratio.
- Sign worker leases and authenticate internal APIs with mTLS.
- Treat page text, headers, and robots files as untrusted data.
- Maintain takedown, domain exclusion, user-agent identification, and contact processes.
- Log policy overrides and protect operator APIs with least privilege.

## 16. Observability

Track fetch success by error class, pages/second, bytes/second, frontier age, eligible-origin lag, per-origin request rate, robots denials, lease expiry, duplicate ratio, content-change rate, parser failures, 429/503 rates, retry volume, and storage growth.

Key SLOs:

- 99% of high-priority eligible URLs start within their scheduling deadline.
- Zero verified politeness violations attributable to scheduler concurrency.
- Freshness lag by page class stays within target.

## 17. Major trade-offs

### Centralized versus distributed frontier

A central frontier simplifies ordering but becomes a bottleneck. Sharding by origin preserves local politeness and scales horizontally.

### Breadth versus freshness

Discovering more pages competes with recrawling important pages. Allocate explicit budgets by crawl class rather than letting one starve the other.

### Full rendering versus HTTP fetching

Browser rendering increases coverage but costs orders of magnitude more CPU and attack surface. Apply it selectively.

### Strict versus optimistic deduplication

Authoritative checks preserve coverage; probabilistic filters save I/O but can miss pages if used as truth.

## 18. Capacity and cost controls

Enforce global, region, tenant, origin, and content-type budgets. Backpressure propagates from parsers and storage to fetch scheduling. Sample low-value repeated errors, compact attempt history, compress content, and tier unchanged old versions. Admission control should shed speculative discovery before high-priority refresh work.

## 19. Recommended interview presentation

1. Clarify crawl scope, freshness, and robots compliance.
2. Estimate fetch rate, bandwidth, and frontier size.
3. Draw ingestion, frontier, fetch, parse, and storage.
4. Deep-dive on the two-level origin scheduler.
5. Explain canonicalization and layered deduplication.
6. Cover leases, idempotency, traps, and hostile content.
7. Discuss partitioning, recrawl scoring, and multi-region ownership.
8. Finish with observability and cost/freshness trade-offs.

The central design insight is:

> Partition scheduling by origin, enforce politeness before issuing every fenced lease, and treat fetching as an at-least-once operation whose metadata and downstream effects are idempotent.
