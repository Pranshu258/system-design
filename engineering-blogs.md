# Curated Engineering Blogs

Official engineering blog posts and papers from companies that run these systems in production, curated per design. Each design weaves these lessons into its fixes and deep dives and lists the sources under `Sources and further reading`.

## Collaboration and storage

### [Dropbox-like File Storage and Sync](./dropbox-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Dropbox | [Inside the Magic Pocket](https://dropbox.tech/infrastructure/inside-the-magic-pocket) | Explains Dropbox's immutable 4 MB block storage, FileJournal separation, replication, and erasure coding. |
| Dropbox | [Improving storage efficiency in Magic Pocket, our immutable blob store](https://dropbox.tech/infrastructure/improving-storage-efficiency-in-magic-pocket-our-immutable-blob-store) | Shows why immutable stores need GC plus compaction to reclaim deleted blob space at exabyte scale. |
| Dropbox | [Rewriting the heart of our sync engine](https://dropbox.tech/infrastructure/rewriting-the-heart-of-our-sync-engine) | Describes Nucleus and why stable data models are essential for large-scale file sync correctness. |
| Dropbox | [Testing sync at Dropbox](https://dropbox.tech/infrastructure/-testing-our-new-sync-engine) | Details model/randomized testing and protocol invariants for a sync engine with offline clients. |
| Dropbox | [Streaming File Synchronization](https://dropbox.tech/infrastructure/streaming-file-synchronization) | Demonstrates prefetching uncommitted blocklists to pipeline large-file sync and cut latency. |
| Dropbox | [Broccoli: Syncing faster by syncing less](https://dropbox.tech/infrastructure/-broccoli--syncing-faster-by-syncing-less) | Shows block-sync compression reducing median latency and network bytes by more than 30%. |
| Dropbox | [Lepton image compression: saving 22% losslessly from images at 15MB/s](https://dropbox.tech/infrastructure/lepton-image-compression-saving-22-losslessly-from-images-at-15mbs) | Shows specialized deterministic JPEG recompression saving multiple petabytes without changing file bytes. |
| Dropbox | [(Re)Introducing Edgestore](https://dropbox.tech/infrastructure/reintroducing-edgestore) | Presents Dropbox's strongly consistent metadata abstraction over sharded MySQL with caching and geo-replication. |
| Dropbox | [Future-proofing our metadata stack with Panda, a scalable key-value store](https://dropbox.tech/infrastructure/panda-metadata-stack-petabyte-scale-transactional-key-value-store) | Explains rebalancing, ACID transactions, and incremental capacity expansion for petabyte-scale metadata. |
| Dropbox | [That time we unplugged a data center to test our disaster readiness](https://dropbox.tech/infrastructure/disaster-readiness-test-failover-blackhole-sjc) | Separates active-active content storage from harder metadata failover and grounds RTO/RPO planning. |

### [Google Docs](./google-docs-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Google Drive Blog | [What’s different about the new Google Docs: Conflict resolution](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_22.html) | Explains Google Docs' revision log and OT transform model for insert, delete, and style changes. |
| Apache Wave | [Google Wave Operational Transformation](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html) | Details optimistic UI, server acknowledgement, batching, and structured-document OT. |
| Figma | [How Figma’s multiplayer technology works](https://www.figma.com/blog/how-figmas-multiplayer-technology-works/) | Shows when a centralized CRDT-inspired model is simpler than OT for object trees. |
| Figma | [Making multiplayer more reliable](https://www.figma.com/blog/making-multiplayer-more-reliable/) | Describes the write-ahead journal, ownership lock, checkpoints, and replay validation behind reliable multiplayer. |
| Figma | [Realtime editing of ordered sequences](https://www.figma.com/blog/realtime-editing-of-ordered-sequences/) | Introduces fractional indexing for collaborative ordered lists where text-style OT is unnecessary. |
| Notion | [The data model behind Notion’s flexibility](https://www.notion.com/blog/data-model-behind-notion) | Explains block IDs, content arrays, parent pointers, and transactional operations for flexible documents. |
| Notion | [Herding elephants: Lessons learned from sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion) | Covers workspace-local sharding, transitive sharding of related rows, and 480 logical shards over 32 databases. |
| Notion | [How we made Notion available offline](https://www.notion.com/en-gb/blog/how-we-made-notion-available-offline) | Shows complete offline subtrees, reason tracking, push refresh, and conflict-resolution migration. |
| Microsoft | [Fluid Framework: How SharedTree merges changes](https://devblogs.microsoft.com/microsoft365dev/fluid-framework-how-sharedtree-merges-changes/) | Defines sequenced merge semantics for collaborative trees, true moves, removals, and last-writer-wins conflicts. |
| Linear | [Rebuilding Linear’s delta sync read path](https://linear.app/now/rebuilding-delta-sync-read-path) | Demonstrates scalable permission-aware delta sync using immutable actions, inverted indexes, authoritative overlap, and late enrichment. |

### [Live Comments](./live-comments-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Meta | [Live Commenting: Behind the Scenes](https://engineering.fb.com/2011/02/07/core-infra/live-commenting-behind-the-scenes/) | Shows why low-latency comments need push and subscription state rather than frequent polling. |
| Meta / SOSP | [Bladerunner: Stream processing at scale for a live view of backend data mutations at the edge](https://doi.org/10.1145/3477132.3483572) | Describes per-application stream processors that filter, aggregate, and rank live updates before device delivery. |
| Meta | [How we scale Live streaming for millions of viewers simultaneously](https://engineering.fb.com/2020/10/22/video-engineering/live-streaming/) | Details redundancy and failover expectations for high-stakes live events with millions of concurrent viewers. |
| LinkedIn | [Now You See Me, Now You Don’t: LinkedIn’s Real-Time Presence Platform](https://www.linkedin.com/blog/engineering/product-design/now-you-see-me-now-you-dont-linkedins-real-time-presence-platf) | Provides the heartbeat, expiry, delayed-trigger, and sticky-routing pattern for presence and viewer counts. |
| Slack | [Flannel: An Application-Level Edge Cache to Make Slack Scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale-/) | Demonstrates edge caching, slim bootstraps, team affinity, and reconnect-storm reduction for real-time clients. |
| Twitch | [Twitch State of Engineering 2023](https://blog.twitch.tv/en/2023/09/28/twitch-state-of-engineering-2023/) | Explains chat edge, policy evaluation, and Pubsub fan-out separation for hundreds of billions of chat messages/day. |
| Disney+ Hotstar | [Building PubSub for 50M concurrent socket connections](https://blog.hotstar.com/building-pubsub-for-50m-concurrent-socket-connections-5506e3c3dabf) | Shows practical broadcast fan-out lessons: publisher/subscriber split, cluster-size limits, load balancers, and reverse bridges. |
| Discord | [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Shows request coalescing and consistent-hash routing to protect hot message partitions. |

### [Chat App](./chat-app-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Discord | [How Discord Stores Billions of Messages](https://discord.com/blog/how-discord-stores-billions-of-messages) | Cassandra/Scylla-style message bucketing by channel and time to bound partitions and support range reads. |
| Discord | [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Hot-partition mitigation with Rust data services, request coalescing, and consistent-hash routing. |
| Discord | [How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users) | Ordered, distributed real-time fan-out and the CPU/network cost of large pub/sub groups. |
| Discord | [Maxjourney: Pushing Discord's Limits with a Million+ Online Users in a Single Server](https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server) | Large-group responsiveness, membership scans, process handoff, and runtime memory/GC lessons. |
| Slack | [Real-time Messaging](https://slack.engineering/real-time-messaging/) | Channel servers, gateway servers, consistent hashing, transient events, and global WebSocket fan-out. |
| Slack | [Flannel: An Application-Level Edge Cache to Make Slack Scale](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale-/) | Edge metadata caching, lazy loading, reconnect-storm protection, and client pub/sub reduction. |
| Meta Messenger | [Building Mobile-First Infrastructure for Messenger](https://engineering.fb.com/2014/10/09/production-engineering/building-mobile-first-infrastructure-for-messenger/) | Iris snapshot-plus-delta sync, per-user ordered queues, MQTT push, and mobile bandwidth reduction. |
| WhatsApp | [How WhatsApp Enables Multi-Device Capability](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/) | Device identity keys, client fan-out encryption, encrypted history transfer, and secure app-state sync. |
| LinkedIn | [Designing Our New Messaging System](https://www.linkedin.com/blog/engineering/messaging-notifications/designing-our-new-messaging-system) | Centralized message copies, service ownership boundaries, and correctness-first migration principles. |
| LinkedIn | [Unifying Messaging Experiences Across LinkedIn](https://www.linkedin.com/blog/engineering/messaging-notifications/unifying-messaging-experiences-across-linkedin) | Reactive client data layer combining local store, realtime stream, and query APIs across platforms. |

## Commerce and transactions

### [Ticketmaster](./ticketmaster-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Ticketmaster | [Taylor Swift \| The Eras Tour Onsale Explained](https://business.ticketmaster.com/press-release/taylor-swift-the-eras-tour-onsale-explained/) | Real incident numbers for Verified Fan demand shaping, bot/non-code traffic, queue slowdowns, and sale outcomes. |
| Ticketmaster | [Smart Queue: Streamline Sales and Maximize Sell-Through](https://business.ticketmaster.com/smart-queue/) | Shows the production queue contract: account sign-in, assigned positions, controlled admission, and live in-queue messaging. |
| Ticketmaster | [SafeTix: Secure Ticketing Built for Fans](https://business.ticketmaster.com/safetix-encrypted-digital-ticketing/) | Demonstrates why issued tickets should be rotating, account/device-bound credentials rather than reusable screenshots. |
| AWS / SeatGeek | [Build a Virtual Waiting Room with Amazon DynamoDB and AWS Lambda at SeatGeek](https://aws.amazon.com/blogs/architecture/build-a-virtual-waiting-room-with-amazon-dynamodb-and-aws-lambda-at-seatgeek/) | Concrete ticketing waiting-room design with edge gatekeeper, protected-zone tokens, DynamoDB queue state, and operational metrics. |
| SeatGeek | [Browser-Based Load Testing Without Breaking the Bank](https://chairnerd.seatgeek.com/browser-based-load-testing/) | Explains why high-demand onsales need real browser simulations for queues, WebSockets, heartbeats, and token exchange. |
| SeatGeek | [The Transactional Outbox Pattern: Transforming Real-Time Data Distribution at SeatGeek](https://chairnerd.seatgeek.com/transactional-outbox-pattern/) | Production outbox/WAL/Debezium/Kafka approach for avoiding dual-write gaps in inventory, order, and ticket event streams. |
| SeatGeek | [Shielding the Core: Scaling Resilience with a Multi-Layered Approach](https://chairnerd.seatgeek.com/shielding-the-core/) | Shows how CDN shielding, rate limits, circuit breakers, and load tests stop cache-miss spikes from overwhelming origin services. |
| Cloudflare | [Building Waiting Room on Workers and Durable Objects](https://blog.cloudflare.com/building-waiting-room-on-workers-and-durable-objects/) | Details global slot allocation, encrypted queue cookies, bucketed users, and bounded edge coordination for waiting rooms. |
| Cloudflare | [Waiting Room: Random Queueing and Custom Web/Mobile Apps](https://blog.cloudflare.com/waiting-room-random-queueing-and-custom-web-mobile-apps/) | Compares FIFO and random queueing and explains when hype launches need randomized fairness instead of arrival-order dominance. |
| AWS | [Introducing AWS Virtual Waiting Room](https://aws.amazon.com/blogs/compute/introducing-aws-virtual-waiting-room/) | Reference implementation for signed time-limited admission tokens, authorizers, burst buffering, and protected API integration. |

### [Local Delivery (Gopuff)](./local-delivery-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| DoorDash | [Next-Generation Optimization for Dasher Dispatch at DoorDash](https://careersatdoordash.com/blog/next-generation-optimization-for-dasher-dispatch-at-doordash/) | Shows why dispatch evolves from simple matching to MIP-style optimization with predictions, constraints, rollout safety, and multi-delivery routes. |
| DoorDash | [Scaling a routing algorithm using multithreading and ruin-and-recreate](https://careersatdoordash.com/blog/scaling-a-routing-algorithm-using-multithreading-and-ruin-and-recreate/) | Explains bounded route optimization, same-day constraints, concurrency, and greedy fallbacks for last-mile routing. |
| DoorDash | [Improving ETAs with multi-task models, deep learning, and probabilistic forecasts](https://careersatdoordash.com/blog/improving-etas-with-multi-task-models-deep-learning-and-probabilistic-forecasts/) | Details probabilistic, multi-stage ETA modeling and why uncertainty must feed the customer promise. |
| DoorDash | [Building a Source of Truth for an Inventory with Disparate Data Sources](https://careersatdoordash.com/blog/building-a-source-of-truth-for-a-digital-inventory-with-disparate-data-sources/) | Covers inventory-confidence signals, precomputed availability projections, search down-ranking, and substitutions. |
| Instacart | [The story behind an Instacart order, part 3: predicting the shop](https://tech.instacart.com/the-story-behind-an-instacart-order-part-3-predicting-the-shop-d1a3bb9f5d98) | Shows item availability and replacement models for grocery fulfillment with stale partner inventory. |
| Instacart | [The story behind an Instacart order, part 2: browsing and buying](https://tech.instacart.com/the-story-behind-an-instacart-order-part-2-browsing-and-ordering-6b6a6066a706) | Explains searchable grocery catalog scale, contextual pricing, deal application, checkout persistence, and authorization. |
| Uber Eats | [From Restaurants to Retail: Scaling Uber Eats for Everything](https://www.uber.com/us/en/blog/scaling-uber-eats-for-everything/) | Describes INCA catalog/inventory ingestion, enrichment, publishing, indexing, versioning, and rollback at retail scale. |
| Zepto | [Building Search for a 10-Minute World](https://blog.zepto.com/building-search-for-a-10-minute-world-43b1992f5dca) | Explains hyperlocal quick-commerce search where query intent, fulfillment zone, inventory, price, and ranking are resolved live. |
| Zepto / AWS | [How Zepto powers sub-second search using OpenSearch Service OR2 instances](https://aws.amazon.com/blogs/big-data/how-zepto-powers-sub-second-search-using-opensearch-service-or2-instances/) | Shows event-driven hub-level indexing, P0/P1 freshness pipelines, and the trade-off between search refresh latency and real-time inventory validation. |
| Swiggy | [Building Rock-Solid ML Systems](https://medium.com/swiggy-bytes/building-rock-solid-ml-systems-d1d5ab544f0e) | Covers ML operational excellence for ETA and quick-delivery systems, including clipping, shadow mode, error codes, and fallbacks. |

### [Online Auction](./online-auction-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| eBay | [Automatic bidding](https://www.ebay.com/help/buying/bidding/automatic-bidding?id=4014) | Canonical proxy-bidding behavior: hidden maximums, public increments, immediate outbid semantics, and contractual commitment. |
| eBay Developers | [Offer API](https://www.developer.ebay.com/develop/api/buy/offer_api) | Shows proxy bid placement/retraction and buyer bid summaries as explicit marketplace API capabilities. |
| eBay | [eBay's Global Secondary Indexes](https://innovation.ebayinc.com/stories/ebays-global-secondary-indexes/) | Petabyte-scale distributed indexing pattern that avoids scatter-gather reads across listing shards. |
| eBay | [How eBay's Distributed Architecture Surfaces More Item Listings for Buyers](https://innovation.ebayinc.com/stories/how-ebays-distributed-architecture-surfaces-more-item-listings-for-buyers/) | Connects buyer demand signals, listing attributes, and asynchronous data products to search quality. |
| Whatnot | [Scaling Whatnot: Behind the Largest Live Shopping Stream in US History](https://medium.com/whatnot-engineering/scaling-whatnot-behind-the-largest-live-shopping-stream-in-us-history-040a458f538c) | Practical live-auction adjacent lessons on admission control, load testing, connection pooling, and hot-event readiness. |
| Whatnot | [Taking On Shill Bidding](https://blog.teamwhatnot.com/unitedstates/taking-on-shill-bidding) | Trust-and-safety signals for detecting seller-related bids, collusion, and manipulation. |
| AWS | [Building resilient real-time streaming workers with Amazon DynamoDB leases](https://aws.amazon.com/blogs/architecture/building-resilient-real-time-streaming-workers-with-amazon-dynamodb-leases/) | Lease, heartbeat, and reconciliation pattern for exactly-one ownership of stateful WebSocket work. |
| AWS | [AWS AppSync real-time reference architecture](https://aws.amazon.com/blogs/mobile/aws-appsync-real-time-reference-architecture/) | Managed WebSocket subscription fan-out pattern for delivering realtime auction read-model updates. |

### [Payment System](./payment-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Stripe | [Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency) | Explains ambiguous network failures, idempotency keys, replayed results, and backoff for safe payment retries. |
| Airbnb | [Avoiding double payments in a distributed payments system](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb) | Shows a recovery-point idempotency framework for exactly-once effects across payment microservices. |
| Stripe | [Ledger: Stripe's system for tracking and validating money movement](https://stripe.com/blog/ledger-stripe-system-for-tracking-and-validating-money-movement) | Describes immutable fund-flow modeling, clearing/timeliness/completeness checks, and data-quality tooling at payment scale. |
| Square | [Books, an immutable double-entry accounting database service](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/) | Details why Square built a shared double-entry ledger database for settlements, payouts, consistency, and auditability. |
| Modern Treasury | [Behind the Scenes: How We Built Ledgers for High Throughput](https://www.moderntreasury.com/journal/behind-the-scenes-how-we-built-ledgers-for-high-throughput) | Covers hot-account bottlenecks, balance caching, and making double-entry ledger writes fast enough for critical paths. |
| Uber | [Building High Throughput Payment Account Processing](https://www.uber.com/us/en/blog/high-throughput-processing/) | Explains how Uber handles bursty hot financial accounts while preserving strict consistency in Gulfstream. |
| Shopify | [10 Tips for Building Resilient Payment Systems](https://shopify.engineering/building-resilient-payment-systems) | Practical payment resilience advice on low timeouts, queues, retries, observability, and realistic load testing. |
| Stripe | [Online migrations at scale](https://stripe.com/blog/online-migrations) | Gives a dual-write/backfill/read-switch pattern for evolving payment schemas without downtime or data divergence. |
| Airbnb | [Rebuilding Payment Orchestration at Airbnb](https://medium.com/airbnb-engineering/rebuilding-payment-orchestration-at-airbnb-341d194a781b) | Shows how Airbnb decoupled reusable payment orchestration from product-specific booking flows during SOA migration. |

## Search, discovery, and content

### [Post Search (Full-Text Search)](./post-search-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Twitter/X | [Twitter's New Search Architecture](https://blog.x.com/engineering/en_us/a/2010/twitters-new-search-architecture) | Explains Earlybird's real-time Lucene-derived inverted index, lock-free postings, reverse traversal, and early termination. |
| X | [Reducing search indexing latency to one second](https://blog.x.com/engineering/en_us/topics/infrastructure/2020/reducing-search-indexing-latency-to-one-second) | Shows how one-second indexing required timestamp-based doc IDs, skip-list postings, partial updates, dark reads, and careful rollout. |
| Twitter/X | [Omnisearch index formats](https://blog.x.com/engineering/en_us/topics/infrastructure/2016/omnisearch-index-formats) | Details active versus optimized postings formats, delta compression, bit packing, and measured memory/latency wins. |
| Meta | [Under the Hood: Building out the infrastructure for Graph Search](https://engineering.fb.com/2013/03/06/core-infra/under-the-hood-building-out-the-infrastructure-for-graph-search/) | Introduces Unicorn as the unified inverted-index infrastructure for Facebook's social graph search. |
| Meta | [Under the Hood: Indexing and ranking in Graph Search](https://engineering.fb.com/2013/03/14/core-infra/under-the-hood-indexing-and-ranking-in-graph-search/) | Covers vertical fan-out, query rewriting, forward-index scoring, result-set scoring, blending, and A/B-tested ranking. |
| LinkedIn | [Did you mean "Galene"?](https://engineering.linkedin.com/search/did-you-mean-galene) | Describes Galene's federators, brokers, offline rebuilds, live updates, and separation of infrastructure from relevance. |
| Discord | [How Discord Indexes Billions of Messages](https://discord.com/blog/how-discord-indexes-billions-of-messages) | Shows application-layer routing to smaller Elasticsearch clusters, lazy indexing, shard mapping, and refresh control. |
| Discord | [How Discord Indexes Trillions of Messages](https://discord.com/blog/how-discord-indexes-trillions-of-messages) | Explains Elasticsearch cells, destination-aware Pub/Sub bulk indexing, Kubernetes operations, and special handling for huge guilds. |
| Slack | [Search at Slack](https://slack.engineering/search-at-slack/) | Demonstrates two-stage Solr retrieval plus personalized re-ranking with work-graph features and click-bias-aware training. |
| Slack | [How we built enterprise search to be secure and private](https://slack.engineering/how-we-built-enterprise-search-to-be-secure-and-private/) | Shows when to use federated, permission-aware real-time retrieval instead of indexing stale external data. |

### [News Aggregator](./news-aggregator-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Google Research | [Google News Personalization: Scalable Online Collaborative Filtering](https://research.google/pubs/google-news-personalization-scalable-online-collaborative-filtering/) | Shows why online collaborative filtering must be blended with fresh content/profile features for rapidly changing news inventory. |
| Google Blog | [New Google News: AI meets human intelligence](https://blog.google/products-and-platforms/products/news/new-google-news-ai-meets-human-intelligence/) | Explains real-time AI/ML organization of news into evolving storylines and personalized briefings. |
| Google Blog | [Get the full news story with Full Coverage in Search](https://blog.google/products-and-platforms/products/news/get-full-news-story-full-coverage-search/) | Shows how story pages expose multiple sources, local headlines, explainers, and perspectives for complex developing events. |
| Google Search Central | [Best practices for XML sitemaps and RSS/Atom feeds](https://developers.google.com/search/blog/2014/10/best-practices-for-xml-sitemaps-rssatom) | Clarifies why feeds are for recent updates while sitemaps are for complete URL discovery and reconciliation. |
| Google Research | [Detecting Near-Duplicates for Web Crawling](https://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/33026.pdf) | Establishes compact fingerprinting as a crawl-scale near-duplicate detection primitive. |
| Yahoo! JAPAN Tech Blog | [Yahoo! JAPANアプリにおけるニュース記事レコメンドの取り組み](https://techblog.yahoo.co.jp/entry/2023022730415148/) | Describes request-time vector retrieval, new-article cold start handling, and dedup/diversity for news recommendations. |
| SmartNews Engineering | [Feature Engineering Practice in SmartNews](https://medium.com/smartnews-inc/feature-engineering-practice-in-smartnews-e14877e3edc2) | Details real-time/offline feature-store design and sliding-window pitfalls for low-latency recommendation features. |
| LinkedIn Engineering | [Feed relevance](https://engineering.linkedin.com/teams/data/artificial-intelligence/feed) | Describes first-pass rankers, second-pass ranking, re-ranking, and implicit feedback for heterogeneous personalized feeds. |
| Microsoft Learn | [Microsoft News Recommendation Dataset](https://learn.microsoft.com/en-us/azure/open-datasets/dataset-microsoft-news) | Documents impression logs, clicked/non-clicked negatives, entities, and anonymized users for news-recommendation training data. |

### [News Feed](./news-feed-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Meta | [How machine learning powers Facebook’s News Feed ranking algorithm](https://engineering.fb.com/2021/01/26/ml-applications/news-feed-ranking/) | Shows multi-pass News Feed ranking, multi-objective predictions, and diversity rules at billions-user scale. |
| Meta | [TAO: The power of the graph](https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/) | Explains social graph storage, object/association APIs, celebrity hot spots, and cache-aware graph reads. |
| X/Twitter | [Using Deep Learning at Scale in Twitter’s Timelines](https://blog.x.com/engineering/en_us/topics/insights/2017/using-deep-learning-at-scale-in-twitters-timelines) | Shows ranked timeline layering over chronological followed inventory and relevance scoring under tight latency. |
| X/Twitter | [X’s Recommendation Algorithm](https://github.com/twitter/the-algorithm/blob/main/README.md) | Documents candidate sources, real-time user signals, graph embeddings, model serving, product mixing, and visibility filters. |
| X/Twitter | [Manhattan, our real-time, multi-tenant distributed database for Twitter scale](https://blog.x.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale) | Covers low-latency multi-tenant storage, quotas, rate limits, replication, and workload-specific storage for social data. |
| LinkedIn | [FollowFeed: LinkedIn’s Feed Made Faster and Smarter](https://www.linkedin.com/blog/engineering/feed/followfeed-linkedin-s-feed-made-faster-and-smarter) | Explains timeline indexes, fan-out-on-read trade-offs, online relevance, RocksDB storage, and p99/feed-cost wins. |
| LinkedIn | [Community-focused Feed optimization](https://www.linkedin.com/blog/engineering/feed/community-focused-feed-optimization) | Details two-pass feed ranking, local feature storage, Quasar model inference, data-pack deployment, and async feature tracking. |
| Pinterest | [Building a smarter home feed](https://medium.com/pinterest-engineering/building-a-smarter-home-feed-ad1918fdfbe3) | Introduces Smart Feed source mixing, best-first ordering, unseen/presented separation, and deferring lower-value items. |
| Pinterest | [Using PID controllers to diversify content types on home feed](https://medium.com/pinterest-engineering/using-pid-controllers-to-diversify-content-types-on-home-feed-1c7195c89218) | Shows production control loops for media mix, freshness, creator goals, and AB-test-safe post-ranking constraints. |
| Meta/Instagram | [Instagram Feed Ranking System Card](https://ai.meta.com/tools/system-cards/instagram-feed-ranking/) | Summarizes feed sourcing, action prediction, score normalization, integrity demotions, and author/media diversity rules. |

### [Instagram](./instagram-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Instagram Engineering | [Sharding & IDs at Instagram](https://medium.com/instagram-engineering/sharding-ids-at-instagram-1cf5a71e5a5c) | Shows time-sortable shard-aware IDs without a central ID service. |
| Instagram Engineering | [Open-sourcing a 10x reduction in Apache Cassandra tail latency](https://medium.com/instagram-engineering/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589) | Explains Rocksandra and why p99/GC behavior matters for online key-value stores. |
| Meta Engineering | [Needle in a haystack: efficient storage of billions of photos](https://engineering.fb.com/2009/04/30/core-infra/needle-in-a-haystack-efficient-storage-of-billions-of-photos/) | Details append-only photo storage with compact in-memory indexes to avoid metadata I/O. |
| Meta Engineering | [An analysis of Facebook photo caching](https://engineering.fb.com/2014/02/27/web/an-analysis-of-facebook-photo-caching-2/) | Quantifies browser, edge, origin, and backend cache effects for photo serving. |
| Facebook / USENIX | [f4: Facebook's Warm BLOB Storage System](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-muralidhar.pdf) | Shows how to tier cooled immutable media into erasure-coded warm storage. |
| Meta AI | [Powered by AI: Instagram’s Explore recommender system](https://ai.meta.com/blog/powered-by-ai-instagrams-explore-recommender-system/) | Gives concrete Explore mechanisms: IGQL, embeddings, distillation, and staged ranking. |
| Meta Engineering | [Scaling the Instagram Explore recommendations system](https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/) | Updates Explore with Two Tower retrieval, ANN, precomputation, MTML ranking, and reranking. |
| Meta Engineering | [How Instagram suggests new content](https://engineering.fb.com/2020/12/10/web/how-instagram-suggests-new-content/) | Covers suggested-post seed selection, cold start, value models, and “Feels Like Home” constraints. |
| Meta Engineering | [Journey to 1000 models: Scaling Instagram’s recommendation system](https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/) | Shows model registry, launch automation, criticality, and stability SLOs for large ML fleets. |
| Meta Engineering | [Introducing Immortal Objects for Python](https://engineering.fb.com/2023/08/15/developer-tools/immortal-objects-for-python-instagram-meta/) | Explains Instagram's Python/Django serving model and memory-efficiency work at scale. |

### [YouTube](./youtube-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| YouTube | [A new era of video infrastructure](https://blog.youtube/inside-youtube/new-era-video-infrastructure/) | Explains YouTube's VCU/Argos accelerator approach and 20-33x transcoding efficiency improvement. |
| Google Research | [Deep Neural Networks for YouTube Recommendations](https://research.google/pubs/deep-neural-networks-for-youtube-recommendations/) | Canonical two-stage candidate generation plus ranking design for YouTube-scale recommendations. |
| Vitess | [Vitess: The Complete Story](https://vitess.io/blog/2016-03-10-percona-live-featured-talk-with-sugu-sougoumarane-vitess-the-complete-story/) | Describes Vitess originating from YouTube's need to shard and operate MySQL at commodity-hardware scale. |
| Netflix | [Per-Title Encode Optimization](https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2) | Shows why each video needs a content-aware bitrate ladder rather than a fixed ladder. |
| Netflix | [Serving 100 Gbps from an Open Connect Appliance](https://netflixtechblog.com/serving-100-gbps-from-an-open-connect-appliance-cdb51dda3b99) | Demonstrates the appliance, OS, NIC, and storage tuning needed for video CDN edge throughput. |
| Netflix | [Optimized shot-based encodes for 4K: Now streaming!](https://netflixtechblog.com/optimized-shot-based-encodes-for-4k-now-streaming-47b516b10bbb) | Extends ladder optimization to shot-level bit allocation and perceptual quality metrics. |
| Meta | [How Facebook encodes your videos](https://engineering.fb.com/2021/04/05/video-engineering/how-facebook-encodes-your-videos/) | Uses benefit-cost and ML prioritization so expensive codecs are applied where they improve QoE most. |
| Meta | [Reducing Instagram's basic video compute by 94 percent](https://engineering.fb.com/2022/11/04/video-engineering/instagram-video-processing-encoding-reduction/) | Demonstrates DAG/intermediate reuse to remove redundant encoding work at massive upload scale. |
| Meta | [Bringing AV1 to Facebook and Instagram Reels](https://engineering.fb.com/2023/02/21/video-engineering/av1-codec-facebook-instagram-reels/) | Covers AV1's compression benefits and the compute/device trade-offs of production codec rollout. |

## Data and infrastructure

### [Metrics Monitoring System](./metrics-monitoring-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Meta | [Beringei: A high-performance time series storage engine](https://engineering.fb.com/2017/02/03/core-infra/beringei-a-high-performance-time-series-storage-engine/) | Explains the Gorilla-style in-memory hot store, delta-of-delta timestamps, XOR float compression, and why recent metrics need millisecond reads. |
| Prometheus | [Storage](https://prometheus.io/docs/prometheus/latest/storage/) | Canonical reference for WAL-protected head chunks, two-hour immutable blocks, tombstones, compaction, and the 1-2 bytes/sample rule of thumb. |
| Uber | [M3: Uber’s Open Source, Large-scale Metrics Platform for Prometheus](https://www.uber.com/gb/en/blog/m3/) | Shows multi-tenant Prometheus-compatible metrics at billions of series, quorum writes, global queries, and policy-driven rollups/retention. |
| Netflix | [Atlas Overview](https://netflix.github.io/atlas-docs/overview/) | Covers dimensional metrics, regional isolation, global query composition, and why metric-name encoding does not scale. |
| Grafana Labs | [How we scaled our new Prometheus TSDB Grafana Mimir to 1 billion active series](https://grafana.com/blog/how-we-scaled-our-new-prometheus-tsdb-grafana-mimir-to-1-billion-active-series/) | Quantifies active-series scaling, redundancy overhead, load testing, and resource needs for billion-series clusters. |
| Datadog | [Evolving our real-time timeseries storage again: Built in Rust for performance at scale](https://www.datadoghq.com/blog/engineering/rust-timeseries-engine/) | Describes separating real-time point storage from tag indexing and redesigning the hot engine for high-cardinality, bursty workloads. |
| Pinterest | [Improving Efficiency Of Goku Time Series Database at Pinterest (Part — 1)](https://medium.com/pinterest-engineering/improving-efficiency-of-goku-time-series-database-at-pinterest-part-1-7659b94796f4) | Highlights short-term/long-term storage separation, compaction, smart routing, and recovery-time bottlenecks. |
| Cloudflare | [How Cloudflare runs Prometheus at scale](https://blog.cloudflare.com/how-cloudflare-runs-prometheus-at-scale/) | Gives operational lessons from hundreds of Prometheus instances, billions of series, and practical cardinality governance. |

### [Distributed Job Scheduler](./distributed-job-scheduler-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Dropbox | [How we designed Dropbox ATF: an async task framework](https://dropbox.tech/infrastructure/asynchronous-task-scheduling-at-dropbox) | Defines at-least-once async task contracts, no-concurrent execution, isolation, and latency targets at 9k tasks/sec. |
| Slack | [Scaling Slack’s Job Queue](https://slack.engineering/scaling-slacks-job-queue/) | Shows how durable Kafka buffering avoided Redis memory deadlock while preserving existing job queue interfaces. |
| Meta | [FOQS: Scaling a distributed priority queue](https://engineering.fb.com/2021/02/22/production-engineering/foqs-scaling-a-distributed-priority-queue/) | Details sharded persistent priority queues with delay, leases, ack/nack, active-topic discovery, and trillion-item/day scale. |
| Meta | [Async: Driving efficiency and developer productivity at Facebook scale](https://engineering.fb.com/2020/08/17/production-engineering/async/) | Explains delay tolerance, time shifting, batching, and fairness for billions of asynchronous requests. |
| Airbnb | [Dynein: Building an Open-source Distributed Delayed Job Queueing System](https://medium.com/airbnb-engineering/dynein-building-a-distributed-delayed-job-queueing-system-93ab10f05f99) | Separates immediate SQS-backed queues from durable delayed scheduling with buffering, rate limits, and backpressure. |
| Google SRE | [Distributed Periodic Scheduling with Cron](https://sre.google/sre-book/distributed-periodic-scheduling/) | Covers distributed cron state, idempotency, skipped-vs-duplicate launch trade-offs, and consensus-backed scheduler replicas. |
| Netflix | [100X Faster: How We Supercharged Netflix Maestro’s Workflow Engine](https://netflixtechblog.com/100x-faster-how-we-supercharged-netflix-maestros-workflow-engine-028e9637f041) | Shows how single-source-of-truth state, rebuildable in-memory actors, partitioned ownership, and generation IDs remove workflow races. |
| Uber | [Announcing Cadence 1.0: The Powerful Workflow Platform Built for Scale and Reliability](https://www.uber.com/us/en/blog/announcing-cadence/) | Highlights code-driven workflows, signals, schedules, resets, versioning, failover, search, and operational tooling. |
| Temporal | [The definitive guide to Durable Execution](https://temporal.io/blog/what-is-durable-execution) | Clarifies durable execution as crash-proof workflow progress while leaving external side effects to idempotency/fencing. |
| Google Research | [Large-scale cluster management at Google with Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/) | Adds placement lessons: admission control, task packing, overcommitment, isolation, correlated-failure avoidance, and simulation. |

### [Pub-Sub / Message Queue](./pub-sub-message-queue-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| LinkedIn | [How LinkedIn customizes Apache Kafka for 7 trillion messages per day](https://www.linkedin.com/blog/engineering/open-source/apache-kafka-trillion-messages) | Shows real partition, broker, controller, maintenance-mode, and quota pressures at Kafka scale. |
| Confluent | [Why ZooKeeper Was Replaced with KRaft - The Log of All Logs](https://www.confluent.io/blog/why-replace-zookeeper-with-kafka-raft-the-log-of-all-logs/) | Explains why large Kafka clusters need a replicated metadata log and scalable controller path. |
| Confluent | [Exactly-Once Semantics Are Possible: Here’s How Apache Kafka Does It](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/) | Details producer idempotence, transactions, and the limits of exactly-once processing. |
| Confluent | [Infinite Kafka Data Retention in Confluent Platform](https://www.confluent.io/blog/infinite-kafka-storage-in-confluent-platform/) | Motivates tiered storage for retention, broker replacement, and partition rebalancing. |
| Uber | [uReplicator: Uber Engineering’s Robust Apache Kafka Replicator](https://www.uber.com/blog/ureplicator-apache-kafka-replicator/) | Shows operational requirements for reliable cross-datacenter Kafka replication. |
| Pinterest | [MemQ: An efficient, scalable cloud native PubSub system](https://medium.com/pinterest-engineering/memq-an-efficient-scalable-cloud-native-pubsub-system-4402695dd4e7) | Demonstrates object-storage-backed pub/sub with decoupled storage and serving for lower-cost topics. |
| Cloudflare | [Using Apache Kafka to process 1 trillion inter-service messages](https://blog.cloudflare.com/using-apache-kafka-to-process-1-trillion-messages/) | Highlights partition skew, client abstractions, and observability lessons for Kafka-backed message buses. |
| Slack | [Scaling Slack’s Job Queue](https://slack.engineering/scaling-slacks-job-queue/) | Connects Kafka-style logs to practical job-queue retry, dedupe, and migration concerns. |
| Google Cloud | [Architectural overview of Pub/Sub](https://docs.cloud.google.com/pubsub/architecture) | Describes forwarders, routers, quorum persistence, and horizontal sharding in a managed pub/sub service. |
| Google Cloud | [Exactly-once delivery](https://docs.cloud.google.com/pubsub/docs/exactly-once-delivery) | Clarifies ack IDs, lease deadlines, regional scope, and duplicate versus redelivery semantics. |

### [Distributed Cache](./distributed-cache-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Meta / USENIX | [Scaling Memcache at Facebook](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) | Canonical design for leases, regional pools, gutter fallback, and origin protection at billion-QPS cache scale. |
| Meta | [Introducing mcrouter: A memcached protocol router for scaling memcached deployments](https://engineering.fb.com/2014/09/15/web/introducing-mcrouter-a-memcached-protocol-router-for-scaling-memcached-deployments/) | Shows how proxy routing, replicated pools, shadowing, warmup, and online reconfiguration become core cache infrastructure. |
| Meta | [Cache made consistent](https://engineering.fb.com/2022/06/08/core-infra/cache-made-consistent/) | Explains Polaris, client-observable consistency SLOs, and tracing the fill/invalidation race window. |
| Meta | [McDipper: A key-value cache for Flash storage](https://engineering.fb.com/2013/03/05/web/mcdipper-a-key-value-cache-for-flash-storage/) | Demonstrates when flash/NVMe cache tiers work and why tombstones/delete hold-offs prevent stale resurrection. |
| Netflix | [EVCache features](https://netflix.github.io/EVCache/features/) | Documents zone-aware discovery, mirroring, retries, client caches, maintenance modes, and explicit best-effort consistency. |
| Uber | [How Uber Serves Over 40 Million Reads Per Second from Online Storage Using an Integrated Cache](https://www.uber.com/us/en/blog/how-uber-serves-over-40-million-reads-per-second-using-an-integrated-cache/) | Shows why caching belongs in the storage/query layer for transparent adoption, independent scale-out, and centralized invalidation. |
| Uber | [How Uber Serves over 150 Million Reads per Second from Integrated Cache with Stronger Consistency Guarantees](https://www.uber.com/us/en/blog/how-uber-serves-over-150-million-reads/) | Covers CDC invalidation, invalidation markers, timestamp dedupe, bypass policies, and TTL-bound staleness at larger scale. |
| Cloudflare | [Instant Purge: invalidating cached content in under 150ms](https://blog.cloudflare.com/instant-purge/) | Details local per-machine purge indexes with RocksDB sidecars, active purge, and global invalidation latency targets. |
| Twitter / Pelikan | [Why Pelikan](https://pelikan.io/blog/why-pelikan/) | Explains operational cache-server requirements: predictable memory, tail latency isolation, protocol extensibility, and safer admin commands. |
| Twitter / Pelikan | [Segcache: a memory-efficient, scalable cache for small objects with TTL](https://pelikan.io/blog/segcache/) | Shows how TTL-heavy small-object workloads are dominated by metadata, fragmentation, and expiration efficiency. |

### [Distributed Key-Value Store](./distributed-key-value-store-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| AWS | [Scaling DynamoDB: How partitions, hot keys, and split for heat impact performance](https://aws.amazon.com/blogs/database/part-3-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/) | Shows partition splitting, adaptive capacity, burst behavior, and why hot keys still need design work. |
| Amazon / USENIX | [Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service](https://www.usenix.org/conference/atc22/presentation/elhemali) | Explains partitioned Multi-Paxos replication, autoscaling, backups, global tables, and operating at tens of millions of requests per second. |
| Amazon / USENIX | [Distributed Transactions at Scale in Amazon DynamoDB](https://www.usenix.org/system/files/atc23-idziorek.pdf) | Details how DynamoDB adds bounded distributed transactions without making normal key-value operations slow. |
| Discord | [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages) | Demonstrates hot-partition mitigation with data services, request coalescing, consistent routing, and Cassandra-to-ScyllaDB migration results. |
| Meta | [How we built a general purpose key value store for Facebook with ZippyDB](https://engineering.fb.com/2021/08/06/core-infra/zippydb/) | Shows a multi-tenant strongly consistent KV service with tiers, shards, RocksDB, Paxos/async replication, and flexible durability choices. |
| Uber | [The Architecture of Schemaless, Uber Engineering’s Trip Datastore Using MySQL](https://www.uber.com/us/en/blog/schemaless-part-two-architecture/) | Covers immutable cells, sharding, versioned updates, and eventually consistent indexes in a production datastore. |
| LinkedIn | [Open Sourcing Venice: LinkedIn’s Derived Data Platform](https://www.linkedin.com/blog/engineering/open-source/open-sourcing-venice-linkedin-s-derived-data-platform) | Provides production numbers and patterns for read-heavy derived KV data, BatchGet fanout, multi-tenancy, quotas, and multi-region operation. |
| Apple / FoundationDB | [Technical Overview](https://apple.github.io/foundationdb/technical-overview.html) | Shows why a minimal ordered transactional KV core, layers, optimistic concurrency, and simulation-driven testing matter. |
| Netflix | [Introducing Netflix’s Key-Value Data Abstraction Layer](https://netflixtechblog.com/introducing-netflixs-key-value-data-abstraction-layer-1ea8a0a11b30) | Explains how a KV DAL standardizes consistency, idempotency, pagination, and data modeling across multiple backing stores. |

## Location and large-scale ingestion

### [Uber / Nearby Drivers](./nearby-drivers-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Uber | [H3: Uber’s Hexagonal Hierarchical Spatial Index](https://www.uber.com/us/en/blog/h3/) | Hierarchical hex cells for dispatch, pricing, rollups, and hot-cell management. |
| Uber | [Ringpop: Open Source Node.js Library](https://www.uber.com/us/en/blog/ringpop-open-source-nodejs-library/) | Shows why real-time driver locations are ephemeral in-memory state distributed across geo workers. |
| Uber | [Uber’s Fulfillment Platform: Ground-up Re-architecture to Accelerate Uber’s Go/Get Strategy](https://www.uber.com/us/en/blog/fulfillment-platform-rearchitecture/) | Strong multi-entity fulfillment state needs transactions, state models, and a source-of-truth store. |
| Uber | [Uber’s Real-Time Push Platform](https://www.uber.com/us/en/blog/real-time-push-platform/) | Replaces aggressive mobile polling with streaming updates while keeping API reconciliation. |
| Uber | [DeepETA: How Uber Predicts Arrival Times Using Deep Learning](https://www.uber.com/us/en/blog/deepeta-how-uber-predicts-arrival-times/) | ETA ranking needs real-time signals and ML refinement beyond graph shortest paths. |
| Uber | [How We Built Uber Engineering’s Highest Query per Second Service Using Go](https://www.uber.com/us/en/blog/go-geofence-highest-query-per-second-service/) | High-QPS geofence/policy lookups belong in specialized low-latency serving paths. |
| Uber | [Reinforcement Learning for Modeling Marketplace Balance](https://www.uber.com/us/en/blog/reinforcement-learning-for-modeling-marketplace-balance/) | Dispatch ranking should consider future supply-demand balance, not only nearest ETA. |
| Grab | [Pharos - Searching Nearby Drivers on Road Network at Scale](https://engineering.grab.com/pharos-searching-nearby-drivers-on-road-network-at-scale) | Road-network KNN, map matching, in-memory replicas, and ETA-based search for driver allocation. |
| Lyft | [How We Built a Smarter Pickup Experience for Gated Communities](https://eng.lyft.com/how-we-built-a-smarter-pickup-experience-for-gated-communities-47416e9df029) | Pickup normalization, gate-aware routing, and access instructions reduce cancellations and bad ETAs. |

### [Google Maps](./google-maps-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Google Maps | [Google Maps 101: how we map the world](https://blog.google/products-and-platforms/products/maps/google-maps-101-how-we-map-world/) | Shows why production maps require imagery, authoritative sources, ML, human review, and user feedback with provenance. |
| Google Maps | [Google Maps 101: How AI helps predict traffic and determine routes](https://blog.google/products-and-platforms/products/maps/google-maps-101-how-ai-helps-predict-traffic-and-determine-routes/) | Explains blending aggregate live traffic, historical patterns, route selection, incidents, and freshness under changing traffic regimes. |
| Google DeepMind | [Traffic prediction with advanced Graph Neural Networks](https://deepmind.google/blog/traffic-prediction-with-advanced-graph-neural-networks/) | Details GNN supersegments, adjacent-road message passing, and production stability concerns for ETA prediction. |
| Uber | [ETA Phone Home: How Uber Engineers an Efficient Route](https://www.uber.com/us/en/blog/engineering-routing-engine/) | Shows routing-engine evolution, contraction-hierarchy rebuild limits, dynamic traffic trade-offs, and cell-based preprocessing. |
| Microsoft Research | [Faster Customization of Road Networks](https://www.microsoft.com/en-us/research/publication/faster-customization-of-road-networks/) | Provides the CRP rationale for fast metric customization over a mostly stable road topology. |
| Uber | [DeepETA: How Uber Predicts Arrival Times Using Deep Learning](https://www.uber.com/ch/en/blog/deepeta-how-uber-predicts-arrival-times/) | Demonstrates low-latency ML residual correction on top of graph routing and real-time traffic. |
| Uber | [Scaling Real-Time Traffic Forecasting with a Graph-Aware Transformer](https://www.uber.com/us/en/blog/scaling-real-time-traffic/) | Gives concrete scale for map-matched GPS pings, segment traversal forecasts, future horizons, and high-QPS traffic ML serving. |
| Mapbox | [Vector tiles introduction](https://docs.mapbox.com/data/tilesets/guides/vector-tiles-introduction/) | Explains MVT/PBF vector tile data, client styling, caching, interactivity, and raster trade-offs. |
| Google Maps | [How we’re giving everyone, everywhere an address](https://blog.google/products-and-platforms/products/maps/google-maps-101-giving-everyone-everywhere-an-address/) | Introduces Plus Codes for geocoding and navigation where conventional street addresses are incomplete or absent. |

### [Web Crawler](./web-crawler-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Google Search Central | [What crawl budget means for Googlebot](https://developers.google.com/search/blog/2017/01/what-crawl-budget-means-for-googlebot) | Defines crawl budget as crawl rate limit plus crawl demand, grounding origin health and freshness prioritization. |
| Google Search Central | [Formalizing the Robots Exclusion Protocol specification](https://developers.google.com/search/blog/2019/07/rep-id) | Explains REP edge cases: 500 KiB parsing, 24-hour robots caching, and conservative behavior on unavailable robots files. |
| Google Search Central | [Inside Googlebot: demystifying crawling, fetching, and the bytes we process](https://developers.google.com/search/blog/2026/03/crawler-blog-post) | Documents modern Googlebot fetch limits, especially the 2 MB non-PDF per-URL cutoff that motivates parser byte caps. |
| Google Research | [Detecting Near-Duplicates for Web Crawling](https://research.google.com/pubs/archive/33026.pdf) | Introduces SimHash-style near-duplicate detection for mirrors and templated pages at crawler scale. |
| Common Crawl | [September 2026 Crawl Archive Now Available](https://commoncrawl.org/blog/september-2026-crawl-archive-now-available) | Gives real crawl-scale numbers for pages, hosts, domains, new URLs, and uncompressed storage volume. |
| Common Crawl | [Host- and Domain-Level Web Graphs July, August, and September 2026](https://commoncrawl.org/blog/host--and-domain-level-web-graphs-july-august-and-september-2026) | Shows why link graph construction and ranking should be an offline pipeline feeding frontier priority. |
| Microsoft Bing Webmaster | [Bingbot Series: Maximizing Crawl Efficiency](https://blogs.bing.com/webmaster/2018/10/bingbot-Series-Maximizing-Crawl-Efficiency/) | Frames crawl quality as fresh/new content discovered per fetch, not merely pages crawled per second. |
| Microsoft Bing Webmaster | [IndexNow: Announcing Sharing of Submitted URLs](https://blogs.bing.com/webmaster/2022/1/IndexNow-Announcing-Sharing-of-Submitted-URLs/) | Shows how push URL-change notifications can improve freshness and reduce waste while still needing validation. |
| Cloudflare | [From 0 to 20 billion - How We Built Crawler Hints](https://blog.cloudflare.com/from-0-to-20-billion-how-we-built-crawler-hints/) | Details a production freshness-hint pipeline with Kafka, Redis dedupe generations, batching, retries, and observability. |

### [Ad Click Aggregator](./ad-click-aggregator-system-design.md)

| Source | Article | Why it matters |
|---|---|---|
| Uber | [Real-Time Exactly-Once Ad Event Processing with Apache Flink, Kafka, and Pinot](https://www.uber.com/us/en/blog/real-time-exactly-once-ad-event-processing/) | Shows Flink/Kafka transactions, record UUID idempotency, Hive dedupe, and Pinot upserts for ads events. |
| Google | [Ubiq: A Scalable and Fault-tolerant Log Processing Infrastructure](https://research.google/pubs/ubiq-a-scalable-and-fault-tolerant-log-processing-infrastructure/) | Describes geo-distributed exactly-once log processing used by Google advertising pipelines. |
| Google | [Photon: Fault-tolerant and Scalable Joining of Continuous Data Streams](https://research.google/pubs/photon-fault-tolerant-and-scalable-joining-of-continuous-data-streams/) | Explains low-latency, delayed-stream joins for ad clicks/search queries with eventual exactly-once output. |
| Google | [Mesa: Geo-Replicated, Near Real-Time, Scalable Data Warehousing](https://research.google/pubs/mesa-geo-replicated-near-real-time-scalable-data-warehousing/) | Covers versioned, geo-replicated ad measurement tables with consistent repeatable query answers. |
| Pinterest | [Ensuring High Availability of Ads Realtime Streaming Services](https://medium.com/pinterest-engineering/ensuring-high-availability-of-ads-realtime-streaming-services-ea3889420490) | Shows hot-hot budget/pacing/bidding streams, health metrics, and automatic switchover for ads spend control. |
| X/Twitter | [Processing billions of events in real time at Twitter](https://blog.x.com/engineering/en_us/topics/infrastructure/2021/processing-billions-of-events-in-real-time-at-twitter-) | Demonstrates UUID dedupe, late-event handling, Bigtable serving, and validation against batch counts at massive scale. |
| LinkedIn | [Real-time Analytics at Massive Scale with Pinot](https://engineering.linkedin.com/analytics/real-time-analytics-massive-scale-pinot) | Explains why ads reporting needs a dedicated real-time columnar OLAP system with specialized indexes. |
