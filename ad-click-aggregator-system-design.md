# Ad Click Aggregation and Reporting System

## 1. Requirements

### Functional requirements

1. Ingest ad impressions, clicks, conversions, and billing adjustments.
2. Produce near-real-time counters by advertiser, campaign, creative, geography, and device.
3. Provide finalized historical reports and downloadable exports.
4. Deduplicate retries and reject malformed or unauthorized events.
5. Attribute conversions to eligible clicks under configurable windows.
6. Detect invalid traffic and preserve raw evidence for reprocessing.
7. Separate estimated live metrics from billing-grade finalized totals.

### Non-functional requirements

- Very high write throughput and burst tolerance.
- Live dashboards within one minute.
- Financially correct, reproducible daily totals.
- No acknowledged event loss; bounded double counting.
- Multi-tenant isolation, privacy controls, and auditable adjustments.

Ad serving itself and auction ranking are out of scope.

## 2. Scale assumptions

- 10 billion impressions/day and 500 million clicks/day.
- 100 million conversions/day.
- Peak ingest: 1 million events/second.
- Average encoded event: 500 bytes: about 61 MB/second at daily average and 500 MB/second at peak, before replication.
- Raw event storage is about 5.3 TB/day uncompressed, or about 2.1 PB for 13 months before compression, indexes, and replication.
- 10 million campaigns, 100 million creatives, 1 million advertisers.
- Interactive reports cover 90 days; raw events retained 13 months.
- Cardinality can exceed trillions of dimension combinations, so arbitrary pre-aggregation is impossible.

The hot path favors availability and durable capture. Billing correctness is achieved through deduplication, watermarks, reconciliation, and immutable recomputation rather than trusting one streaming counter.

## 3. High-level architecture

```mermaid
flowchart LR
    SDK[Ad Servers / SDKs] --> EDGE[Regional Collectors]
    EDGE --> LOG[(Replicated Event Log)]
    LOG --> VAL[Validation / Enrichment]
    VAL --> RAW[(Immutable Raw Lake)]
    VAL --> DEDUP[Deduplication]
    DEDUP --> STREAM[Windowed Aggregation]
    STREAM --> HOT[(Real-time OLAP)]
    DEDUP --> ATTR[Attribution Engine]
    ATTR --> STREAM
    RAW --> BATCH[Daily Recompute / Reconciliation]
    BATCH --> FINAL[(Finalized OLAP Tables)]
    HOT --> QUERY[Reporting API]
    FINAL --> QUERY
    QUERY --> UI[Advertiser Dashboard]
    BATCH --> BILL[Billing Ledger]
    VAL --> FRAUD[Invalid Traffic Detection]
```

Raw immutable events are the source of truth. Live OLAP views are replaceable materializations with upserts/revisions; the billing ledger is append-only and finalized from reconciled data.

## 4. Event and data model

```text
AdEvent
- event_id, event_type: IMPRESSION | CLICK | CONVERSION
- event_time, received_time, producer_id, producer_sequence
- advertiser_id, campaign_id, ad_group_id, creative_id
- placement_id, publisher_id, country, device_type
- user_token, click_id, impression_id
- cost_micros, currency, consent_state, schema_version

AggregateKey
- window_start, window_size
- advertiser_id, campaign_id, selected_dimensions
- impressions, clicks, conversions, spend_micros
- invalid_events, unique_estimate, version, finalization_state

Attribution
- conversion_id, touch_event_id, model, attributed_value
- decision_version, created_at, reversed_at

BillingEntry
- ledger_entry_id, advertiser_id, service_date
- amount_micros, currency, source_batch_id
- entry_type: CHARGE | CREDIT | ADJUSTMENT
```

Money is integer micros, never floating point. Dimension IDs are immutable references; slowly changing names live in versioned metadata.

## 5. Ingestion APIs

```http
POST /v1/events:batch
Content-Encoding: gzip

{"producer_id":"adserver-us-7","events":[{"event_id":"...","type":"CLICK","event_time":"..."}]}
```

Response acknowledges the highest durably committed producer sequence, not downstream aggregation:

```json
{"accepted":999,"rejected":1,"committed_sequence":8127331}
```

Reporting:

```http
GET /v1/reports?advertiser_id=42&from=...&to=...&group_by=campaign,country
POST /v1/report-exports
GET /v1/report-exports/{export_id}
```

Report responses declare `data_fresh_through`, `is_final`, timezone, currency, and metric definitions.

## 6. Ingestion and validation flow

1. Authenticate producer and enforce tenant/schema quotas.
2. Validate required IDs, timestamps, enums, consent, and numeric bounds.
3. Add trusted receive time, region, and collector identity.
4. Append to a quorum-replicated log.
5. Return success only after durable append.
6. Enrichment joins campaign metadata using an event-time-valid version.
7. Persist canonical events to partitioned immutable storage.
8. Deduplicate and route to aggregation, attribution, and fraud streams.

Invalid records go to a reason-coded quarantine, not a silent drop. Backpressure returns explicit overload responses before collectors accept data they cannot durably retain.

## 7. Deduplication and delivery semantics

Producers generate stable, globally unique `event_id` values and retry with the same ID. Stream processors use:

- Log offsets/checkpoints for replay position.
- A partition-local dedupe store keyed by `event_id` with TTL longer than the retry horizon.
- Transactional writes from checkpoint to aggregate changelog where the sink supports it; otherwise deterministic idempotency keys on every emitted record.
- Batch-wide exact dedupe from immutable raw events for finalization.

The live path is effectively once for normal retries, not mathematically exactly once across every external boundary. As in production Flink/Kafka/OLAP pipelines, transactions cover log-to-log hops, while external stores still need idempotent keys, upserts, and reconciliation. Final reports correct drift by recomputation.

## 8. Event-time windowing

Partition the stream by a stable aggregation owner such as `advertiser_id`. Compute minute and hour tumbling windows from `event_time`, while tracking a watermark:

```text
watermark = max_observed_event_time - allowed_lateness
```

Arrivals after the watermark update a closed-but-correctable window and increment a late-event revision until the correction horizon expires. Live dashboards may accept events up to 24 hours late; daily billing waits longer, for example 72 hours, then handles later arrivals as explicit adjustments.

Use local advertiser timezones only at report/finalization boundaries. Store windows in UTC to avoid daylight-saving ambiguity.

## 9. Aggregation strategy

Pre-aggregate common dimensions at minute/hour/day granularity. Avoid a full data cube: its dimensional explosion is untenable. Store base event-derived rollups in columnar OLAP and let vectorized queries combine them. Use primary-key upserts or versioned aggregate tables so corrected minutes replace older values instead of adding duplicate rows.

For unique users, HyperLogLog gives mergeable estimates with bounded error. Financial metrics, clicks, and conversions use exact integer counters. Stream processors emit changelog upserts with monotonically increasing aggregate versions, so retries cannot apply an older count over a newer one.

## 10. Conversion attribution

1. A click stores a privacy-safe touch record keyed by `click_id` and optionally user token.
2. A conversion names `click_id` when available; otherwise query eligible touches within the campaign window.
3. Apply the versioned attribution model: last-click, first-click, or fractional.
4. Write an immutable attribution decision and corresponding aggregate deltas.
5. Fraud reclassification or model correction writes reversals and replacements, never mutates history invisibly.

Cross-device attribution requires consent and an identity service; without it, report conversions as unattributed rather than guessing.

## 11. Consistency and concurrency

- Raw log order is guaranteed only within a partition.
- Aggregate state has one active owner per partition epoch.
- Checkpoint and state updates are atomic within the stream framework.
- OLAP reads may lag; response freshness makes that visible.
- Campaign budget enforcement cannot depend on delayed analytics; ad serving uses a separate strongly consistent pacing counter.
- Billing entries are append-only and idempotent by `(source_batch_id, ledger_line_id)`; advertiser, service date, currency, and entry type are attributes, not the only uniqueness boundary.

Corrections use compensating deltas. Concurrent report queries select a consistent snapshot/version rather than mixing partially published partitions.

## 12. Partitioning and storage

- Event log: hash by advertiser or campaign to preserve local aggregation.
- Raw lake: partition by event date/hour and region; cluster by advertiser.
- Hot OLAP: shard by advertiser hash and time, replicate across zones.
- Large advertisers receive salted subpartitions with a second-stage merge.
- Metadata: strongly consistent relational/KV store with versioned snapshots.

Time partitioning enables retention and pruning, but partitioning only by time creates hot writes. Combine time with hash buckets.

## 13. Batch reconciliation and finalization

For each service day:

1. Read all raw partitions plus late-event manifests.
2. Validate checksums and exact-deduplicate event IDs.
3. Reapply versioned enrichment, invalid-traffic rules, and attribution.
4. Produce deterministic aggregates under a `batch_id`.
5. Compare live and batch totals; alert beyond tolerance.
6. Atomically publish a manifest pointing reports to the new snapshot.
7. Create idempotent billing ledger entries.

Reprocessing creates a new version and audit diff. It never overwrites the evidence used by an issued invoice.

## 14. Reliability and failures

| Failure | Handling |
|---|---|
| Collector loses downstream connection | Buffer within limit, then explicit retryable failure |
| Producer retries a batch | Stable event IDs and sequences deduplicate it |
| Stream worker crashes | Restore checkpoint/state and replay log |
| Event is very late | Correct live revision or create post-finalization adjustment |
| OLAP write partially succeeds | Versioned idempotent upsert and replay |
| Fraud service unavailable | Mark pending; never silently classify as valid |
| Hot advertiser overloads shard | Salt partitions and merge in second stage |
| Batch output incomplete | Do not publish manifest or bill |
| Region outage | Producers fail over; event IDs suppress duplicates |
| Schema is incompatible | Reject/quarantine by version with visible errors |

## 15. Multi-region design

Collectors accept traffic near producers and append to regional replicated logs. Events asynchronously replicate to a home processing region determined by advertiser. During failover, a new processor acquires a fenced epoch; duplicate cross-region delivery is harmless.

Raw storage is copied cross-region with manifests and checksums. Interactive reads use nearby replicas, while finalized reports and billing use the authoritative published snapshot. Versioned update batches and manifests keep geo-replicated snapshots repeatable even during failover. Regional ingestion clocks are not trusted for event order; both event and receive times are retained.

## 16. Security and privacy

- mTLS or signed service credentials for producers; OAuth/RBAC for advertisers.
- Encrypt transit and storage; isolate tenant encryption keys where required.
- Tokenize user identifiers at collection and restrict raw-event access.
- Enforce consent, purpose limitation, deletion, and regional residency.
- Prevent report inference with minimum group thresholds and export controls.
- Sign event batches and audit schema/rule/model changes.
- Rate-limit ingestion and reporting; detect replayed credentials.
- Keep financial ledger permissions separate from analytics administration.

## 17. Observability

Monitor accepted/rejected events, log durability latency, consumer lag, watermark lag, duplicate rate, late-event distribution, state-store size, OLAP publication lag, query latency, live-versus-final variance, attribution rate, invalid-traffic rate, batch completeness, and billing reconciliation.

Key SLOs:

- 99.9% of accepted events visible in live reporting within 60 seconds.
- 100% of published billing batches pass completeness and balance checks.
- No acknowledged-event loss within the retention and recovery model.

## 18. Major trade-offs

- **Streaming versus batch:** streaming is fresh; batch is deterministic. Use both.
- **Exactly once versus idempotency:** end-to-end exactly once is brittle; stable IDs plus replay and reconciliation are auditable.
- **Pre-aggregation versus flexible queries:** common rollups lower latency; columnar base rollups preserve exploration.
- **Low watermark delay versus completeness:** dashboards favor freshness; billing waits and supports adjustments.
- **Advertiser partitioning versus hot keys:** locality simplifies aggregation but requires salting exceptional tenants.

## 19. Lessons from production systems

### Uber: exactly-once ads processing still needs idempotent sinks

Uber's ads event system uses Flink, Kafka, Pinot, and Hive for near-real-time ad event processing. Flink commits Kafka outputs with checkpoint-aligned two-phase commit and consumers read only committed records, but Uber still emits record UUIDs for downstream idempotency, deduplicates in Hive, and relies on Pinot upserts to avoid duplicate analytical rows. This maps directly to this design: claim exactness only per supported boundary, and require stable ids, sink upserts, and batch reconciliation for billing-grade results.

### Google Ubiq: log processing must survive datacenter loss automatically

Google's Ubiq is a geographically distributed log-processing framework used by Google advertising pipelines. Its important lesson is operational: event-log processing should continue through infrastructure degradation or datacenter outage without manual intervention while maintaining low-latency processing of event bundles. For this design, regional collectors should durably accept locally, replicate to a home processing region, and use fenced processor epochs so failover does not create double owners.

### Google Photon: attribution joins need persistent state and eventual exactness

Photon joins unordered or delayed streams such as ad clicks and web-search queries for Google's advertising system. It targets no duplicate joined output at any point, near-exact real-time joins, and exactly-once results eventually. That is a better mental model for conversion attribution than a simple cache lookup: keep durable touch state for the whole attribution window, tolerate delayed counterpart events, emit reversible decisions, and let finalization close the gap.

### Google Mesa: versioned updates make geo-replicated analytics repeatable

Mesa stores critical Google advertising measurement data with near-real-time ingestion, petabyte scale, millions of row updates per second, and consistent repeatable answers across datacenters. The design implication is to publish aggregate updates as versioned batches/manifests rather than mutable partial tables. Report queries should bind to a snapshot version, and finalized billing should use that immutable version.

### Pinterest: budget and pacing cannot wait for dashboards

Pinterest's ads systems separate spend aggregation from budget enforcement, pacing, and bidding. They run hot-hot primary/standby Kafka Streams pipelines, monitor input QPS and event delay every few minutes, and add a grace period before failback to avoid flip-flop. This supports the design choice that ad serving uses separate strongly consistent or highly available pacing counters; reporting aggregates are too delayed to be the enforcement mechanism.

### X/Twitter: validate streaming counts against a trusted baseline

X/Twitter rebuilt high-scale interaction aggregation around Kafka/Pub/Sub/Dataflow/Bigtable, using UUIDs for downstream dedupe and continuous BigQuery comparisons between raw, deduped, and batch-derived counts. They found discrepancies partly because late events were captured by the new stream but discarded by the older batch pipeline. This design should therefore track duplicate-rate and late-event distributions, and compare live and final totals by key rather than assuming the batch path is automatically more correct.

### LinkedIn Pinot: low-latency OLAP needs columnar indexing, not a generic store

LinkedIn built Pinot because generic databases could not serve arbitrary high-cardinality analytics at web scale with real-time ingestion. Pinot powered products including Ads Reporting and supports low-latency slice-and-dice queries through distributed columnar storage and specialized indexes. This reinforces using a real-time OLAP tier for advertiser dashboards while keeping raw lake and finalized snapshots as the source of truth.

## 20. Recommended interview presentation

1. Clarify live reporting versus billing correctness.
2. Estimate peak event and storage rates.
3. Draw durable ingestion, streaming, OLAP, raw lake, and batch paths.
4. Deep-dive into event IDs, watermarks, and late events.
5. Explain aggregation cardinality and attribution.
6. Cover deterministic reconciliation and append-only billing.
7. Discuss hot partitions, region failover, privacy, and SLOs.
8. Mention production lessons: idempotent sinks, versioned snapshots, attribution state, and separate budget enforcement.
9. Finish with the streaming/batch trade-off.

The central design insight is:

> Acknowledge only durable raw events, make live aggregates replayable and explicitly provisional, then derive billing-grade totals through deterministic deduplication, reconciliation, and versioned publication.


## Further reading: curated engineering blogs

| Source | Article | Why it matters |
|---|---|---|
| Uber | [Real-Time Exactly-Once Ad Event Processing with Apache Flink, Kafka, and Pinot](https://www.uber.com/us/en/blog/real-time-exactly-once-ad-event-processing/) | Shows Flink/Kafka transactions, record UUID idempotency, Hive dedupe, and Pinot upserts for ads events. |
| Google | [Ubiq: A Scalable and Fault-tolerant Log Processing Infrastructure](https://research.google/pubs/ubiq-a-scalable-and-fault-tolerant-log-processing-infrastructure/) | Describes geo-distributed exactly-once log processing used by Google advertising pipelines. |
| Google | [Photon: Fault-tolerant and Scalable Joining of Continuous Data Streams](https://research.google/pubs/photon-fault-tolerant-and-scalable-joining-of-continuous-data-streams/) | Explains low-latency, delayed-stream joins for ad clicks/search queries with eventual exactly-once output. |
| Google | [Mesa: Geo-Replicated, Near Real-Time, Scalable Data Warehousing](https://research.google/pubs/mesa-geo-replicated-near-real-time-scalable-data-warehousing/) | Covers versioned, geo-replicated ad measurement tables with consistent repeatable query answers. |
| Pinterest | [Ensuring High Availability of Ads Realtime Streaming Services](https://medium.com/pinterest-engineering/ensuring-high-availability-of-ads-realtime-streaming-services-ea3889420490) | Shows hot-hot budget/pacing/bidding streams, health metrics, and automatic switchover for ads spend control. |
| X/Twitter | [Processing billions of events in real time at Twitter](https://blog.x.com/engineering/en_us/topics/infrastructure/2021/processing-billions-of-events-in-real-time-at-twitter-) | Demonstrates UUID dedupe, late-event handling, Bigtable serving, and validation against batch counts at massive scale. |
| LinkedIn | [Real-time Analytics at Massive Scale with Pinot](https://engineering.linkedin.com/analytics/real-time-analytics-massive-scale-pinot) | Explains why ads reporting needs a dedicated real-time columnar OLAP system with specialized indexes. |
