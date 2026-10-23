# Production System Design Collection

Detailed, production-scale system design walkthroughs. Each design is built up as an argument: every component exists because a simpler design failed in a specific way.

## How each design is structured

1. **Requirements**, ending with *the binding constraint*: the one property that actually shapes the architecture.
2. **Back-of-the-envelope estimation**: only the numbers that drive design decisions.
3. **API design** and **data model**, each built up from a failure of the simpler version.
4. **High-level design**: start naive, then apply one fix per concrete failure.
5. **Deep dives** (3–4): each opens with a probing question and closes with strong-answer criteria and a Bad / Good / Great rubric.
6. **Variants**, **the transferable pattern**, and **review questions** with collapsible answers.
7. **Sources and further reading**: official engineering blogs, papers, documentation, and standards whose lessons are woven into the design. See [Curated Engineering Blogs](./engineering-blogs.md) for the full index.

## Collaboration and storage

- [Dropbox-like File Storage and Sync](./dropbox-system-design.md)
- [Google Docs](./google-docs-system-design.md)
- [Live Comments](./live-comments-system-design.md)
- [Chat App](./chat-app-system-design.md)

## Commerce and transactions

- [Ticketmaster](./ticketmaster-system-design.md)
- [Local Delivery (Gopuff)](./local-delivery-system-design.md)
- [Online Auction](./online-auction-system-design.md)
- [Payment System](./payment-system-design.md)

## Search, discovery, and content

- [Post Search (Full-Text Search)](./post-search-system-design.md)
- [News Aggregator](./news-aggregator-system-design.md)
- [News Feed](./news-feed-system-design.md)
- [Instagram](./instagram-system-design.md)
- [YouTube](./youtube-system-design.md)

## Data and infrastructure

- [Metrics Monitoring System](./metrics-monitoring-system-design.md)
- [Distributed Job Scheduler](./distributed-job-scheduler-system-design.md)
- [Pub-Sub / Message Queue](./pub-sub-message-queue-system-design.md)
- [Distributed Cache](./distributed-cache-system-design.md)
- [Distributed Key-Value Store](./distributed-key-value-store-system-design.md)

## Location and large-scale ingestion

- [Uber / Nearby Drivers](./nearby-drivers-system-design.md)
- [Google Maps](./google-maps-system-design.md)
- [Google Street View](./google-street-view-system-design.md)
- [Web Crawler](./web-crawler-system-design.md)
- [Ad Click Aggregator](./ad-click-aggregator-system-design.md)

## Suggested study order

1. Distributed Key-Value Store
2. Distributed Cache
3. Pub-Sub / Message Queue
4. Dropbox-like File Storage and Sync
5. Chat App
6. News Feed
7. Post Search
8. Metrics Monitoring System
9. Ticketmaster
10. Payment System

The infrastructure designs introduce primitives reused throughout the product-oriented designs.
