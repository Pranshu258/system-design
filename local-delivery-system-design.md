# Local Delivery System: Gopuff-like Instant Commerce

## 1. Requirements

### Functional requirements

1. Show a serviceable catalog and accurate price/availability for a delivery address.
2. Support cart, promotions, checkout, payment, substitution preferences, and cancellation.
3. Reserve inventory from a specific micro-fulfillment center (MFC).
4. Route orders to picking and packing workflows.
5. Assign a courier and provide live ETA and tracking.
6. Handle out-of-stock items, substitutions, refunds, tips, and age-restricted goods.
7. Let operators manage inventory, batches, hours, delivery zones, and courier capacity.
8. Notify customers throughout the order lifecycle.

Out of scope: marketplace restaurants, scheduled parcel shipping, and warehouse procurement optimization.

### Non-functional requirements

- Checkout availability of `99.99%`.
- Catalog reads below 200 ms at `p95`; order creation below 1 second excluding payment challenge.
- Prevent overselling while keeping browse availability responsive.
- Typical order delivered in 15-30 minutes.
- Durable financial and order state with complete audit history.
- Live tracking updates every few seconds without overloading transactional storage.

## 2. Scale assumptions and capacity

- 20 million registered users; 3 million daily active users.
- 1,000 MFCs across several countries.
- 2 million orders/day; 100,000 orders at peak hour.
- Peak checkout rate: about 30-50 orders/second globally, with strong local spikes.
- 25 items/order; 50 million order lines/day.
- 50,000 active couriers at peak, emitting location every 5 seconds: 10,000 updates/second.
- 5,000 SKUs per MFC; about 5 million location-SKU inventory rows.
- Browse traffic is roughly 100 times order-write traffic.

The global throughput is manageable, but locality creates hot partitions: a single MFC can receive hundreds of simultaneous checkouts during a promotion or weather event.

## 3. High-level architecture

```mermaid
flowchart LR
    U[Customer Apps] --> EDGE[API Gateway]
    D[Courier App] --> EDGE
    W[Warehouse App] --> EDGE
    EDGE --> GEO[Serviceability / Location]
    EDGE --> CAT[Catalog Service]
    EDGE --> CART[Cart / Pricing]
    EDGE --> ORDER[Order Orchestrator]
    EDGE --> TRACK[Tracking Service]

    CAT --> INVREAD[(Inventory Read Cache)]
    CAT --> CATALOG[(Catalog DB)]
    ORDER --> INV[Inventory Service]
    ORDER --> PAY[Payment Service]
    ORDER --> FULFILL[Fulfillment Service]
    INV --> INVDB[(Inventory DB)]
    ORDER --> ORDERDB[(Order DB)]
    FULFILL --> DISPATCH[Dispatch Service]
    DISPATCH --> GEOIDX[(Courier Geo Index)]
    D --> LOC[Location Ingestion] --> GEOIDX
    ORDER --> OUTBOX[(Outbox)]
    OUTBOX --> BUS[Event Stream]
    BUS --> NOTIFY[Notification Service]
    BUS --> TRACK
    BUS --> ANALYTICS[Operations / Analytics]
```

The ordering path is an orchestrated saga across inventory, payment, fulfillment, and dispatch. Each domain owns its database; changes propagate through durable events rather than distributed transactions.

## 4. Data model

```text
FulfillmentCenter
- mfc_id
- region_id
- location
- service_polygon
- operating_hours
- status

SKU
- sku_id
- product_id
- title
- unit_size
- age_restriction
- substitution_group_id

Inventory
- mfc_id
- sku_id
- on_hand
- reserved
- damaged
- safety_stock
- version
- updated_at

InventoryReservation
- reservation_id
- order_id
- mfc_id
- sku_id
- quantity
- expires_at
- state: HELD | CONSUMED | RELEASED | EXPIRED

Order
- order_id
- customer_id
- mfc_id
- delivery_address_snapshot
- status
- pricing_snapshot
- payment_intent_id
- idempotency_key
- version
- created_at

OrderItem
- order_id
- line_id
- requested_sku_id
- fulfilled_sku_id
- requested_quantity
- fulfilled_quantity
- unit_price_snapshot
- substitution_policy

FulfillmentTask
- task_id
- order_id
- state: QUEUED | PICKING | PACKED | HANDED_OFF
- picker_id
- promised_ready_at

Delivery
- delivery_id
- order_id
- courier_id
- state
- pickup_at
- delivered_at
- proof_of_delivery
```

Maintain an append-only `OrderEvent` timeline for support and audit. Prices, taxes, address, and item descriptions are snapshots so historical orders do not change when catalog data changes.

The inventory invariant is:

```text
sellable = on_hand - reserved - damaged - safety_stock
sellable >= 0
```

## 5. Core APIs

### Browse a serviceable catalog

```http
GET /v1/catalog?latitude=37.78&longitude=-122.41
```

The response includes an `mfc_id`, catalog version, estimated availability, prices, and delivery estimate. Browse availability is advisory; checkout performs authoritative reservation.

### Create an order

```http
POST /v1/orders
Idempotency-Key: 6d944...

{
  "mfc_id": "mfc-17",
  "cart_version": 12,
  "address_id": "addr-3",
  "payment_method_id": "pm-8",
  "items": [
    {"sku_id": "sku-42", "quantity": 2, "substitution": "BEST_MATCH"}
  ],
  "tip": 5.00
}
```

```json
{
  "order_id": "ord-91",
  "status": "CONFIRMED",
  "amount_authorized": 34.18,
  "promised_delivery_at": "2026-10-13T08:05:00Z"
}
```

### Tracking and courier APIs

```http
GET  /v1/orders/{order_id}
GET  /v1/orders/{order_id}/tracking
POST /v1/orders/{order_id}/cancel
POST /v1/couriers/{courier_id}/locations
POST /v1/deliveries/{delivery_id}/accept
POST /v1/deliveries/{delivery_id}/complete
```

All mutation APIs require an idempotency key or a version precondition. Mobile clients retry frequently across poor networks.

## 6. Serviceability, catalog, and ETA

1. Geocode the address and validate deliverability.
2. Query a spatial index for MFC service polygons containing the point.
3. Exclude closed, unhealthy, overloaded, or capacity-constrained centers.
4. Select an MFC using distance, inventory coverage, queue depth, courier supply, and promised SLA.
5. Return that MFC's assortment and prices.

ETA combines:

```text
ETA = order_queue_delay
    + pick_and_pack_time
    + courier_assignment_delay
    + pickup_wait
    + route_travel_time
```

Use percentile estimates rather than averages and recalibrate continuously. The displayed ETA should include capacity and uncertainty; optimistic straight-line distance alone causes missed promises.

## 7. Checkout and order saga

One practical ordering sequence is:

1. Validate cart, MFC, address, age restrictions, price version, and limits.
2. Create `PENDING` order idempotently.
3. Atomically reserve all inventory lines at the selected MFC.
4. Authorize the estimated payment amount.
5. Commit the order as `CONFIRMED` and create the fulfillment task.
6. Publish the order event through a transactional outbox.
7. Begin picking; later capture the final amount after substitutions and weighed items.

Compensations:

- Inventory reservation fails: reject with unavailable items; no payment authorization.
- Payment authorization fails: release reservations.
- Fulfillment creation fails after confirmation: retry from outbox; operator queue if prolonged.
- Order is cancelled before picking: release inventory and void authorization.
- Partial fulfillment: adjust inventory, total, tax, and captured amount.

There is no cross-service exactly-once transaction. The goal is exactly-once business effect through stable operation IDs, idempotent handlers, state-machine guards, and reconciliation.

## 8. Inventory consistency and concurrency

Inventory reservation is the central correctness problem. Execute all lines for one MFC in a database transaction:

```sql
UPDATE inventory
SET reserved = reserved + :quantity,
    version = version + 1
WHERE mfc_id = :mfc_id
  AND sku_id = :sku_id
  AND on_hand - reserved - damaged - safety_stock >= :quantity;
```

Lock rows in sorted `sku_id` order to reduce deadlocks. If any line fails, roll back the entire reservation unless the product explicitly supports partial checkout.

Browse reads use an eventually consistent cache. Checkout uses the primary inventory store. Reservation records have expirations; a sweeper releases abandoned holds, but it must transition reservation state conditionally so it cannot release a consumed hold.

Physical stock can still differ because of shrinkage, damage, or mis-picks. Cycle counts and picker reports produce adjustments. Safety stock and conservative low-stock thresholds trade sales for fewer substitutions.

## 9. Picking, substitution, and handoff

Fulfillment queues are partitioned by MFC. Within a center, prioritize by promised time, temperature requirements, and route batching.

1. A picker claims a task with a lease.
2. The app gives a route optimized for aisle/bin locations.
3. Each scan validates barcode and quantity.
4. Missing items trigger the customer's substitution policy.
5. Material price changes may request customer approval with a deadline.
6. Packing records bags, temperature zones, restricted items, and final total.
7. A handoff scan transfers custody to the assigned courier.

Picker leases expire and can be reclaimed. Every scan uses a unique event ID; retries must not decrement inventory twice.

## 10. Dispatch and live tracking

Dispatch continuously matches ready-soon orders to couriers using:

- Predicted pick completion.
- Courier position and vehicle type.
- Delivery route and promised SLA.
- Batch compatibility and detour cost.
- Courier capacity, breaks, and fairness.

Start with greedy constrained matching every few seconds; introduce optimization windows or min-cost matching when batching value justifies complexity.

Courier locations enter a high-write, TTL-based geo store, not the order database. Keep the latest point for dispatch and a sampled, retention-limited trail for support. Customers receive throttled updates over WebSocket/SSE; raw courier coordinates should stop after delivery.

Delivery state changes use compare-and-swap:

```text
ASSIGNED -> ACCEPTED -> AT_PICKUP -> PICKED_UP -> AT_CUSTOMER -> DELIVERED
```

Proof can include a photo, PIN, signature, or age verification depending on order policy.

## 11. Payments, refunds, and tips

- Authorize estimated total at checkout, including a bounded substitution buffer where lawful.
- Capture the final fulfilled total after packing or delivery.
- Void unused authorization and issue line-level refunds when needed.
- Keep tips separately adjustable within a policy window.
- Store provider tokens, never raw card data.
- Use an internal payment ledger so provider webhooks and order state can be reconciled.

Payment provider timeouts are ambiguous. Query by the stable payment operation ID before retrying; never create a new charge merely because the response was lost.

## 12. Partitioning and storage

- Partition inventory and fulfillment by `mfc_id`; this preserves local transactions and isolates hot centers.
- Partition orders primarily by `order_id`, with secondary customer and MFC indexes.
- Partition the event stream by `order_id` for ordered lifecycle events; use separate MFC topics for warehouse scheduling.
- Store catalog product data centrally but materialize MFC-specific assortment, price, and availability views.
- Store location points in a geospatial/time-series system with TTL.
- Store receipts and proof images in object storage with signed access.

For a hot MFC, scale reads through caches and replicas, but keep each SKU's reservation authority single-writer. Splitting by `sku_id` increases write capacity but makes multi-line atomic reservation harder; only do so after measuring contention.

## 13. Reliability and important failures

| Failure | Handling |
|---|---|
| Client retries checkout | Return original order by idempotency key and customer |
| Reservation succeeds, payment fails | Saga releases reservation |
| Payment times out | Query provider by operation ID before retry |
| Reservation expires during commit | State transition and transaction prevent release of consumed hold |
| Picker app disconnects | Durable task state and expiring claim lease |
| Item physically missing | Substitute/refund and inventory adjustment |
| Courier rejects or disappears | Lease timeout and redispatch |
| Location updates stop | Show stale indicator and use last ETA |
| Event delivered twice | Idempotent consumer inbox/state guard |
| MFC loses connectivity | Stop new promises; local operational fallback and later reconciliation |
| Region fails | Route unaffected MFCs normally; fenced failover for impacted partitions |

Run reconcilers for inventory reservations versus order state, payment ledger versus provider settlement, packed items versus captured amount, and delivered orders versus courier payout.

## 14. Multi-region design

MFCs are naturally regional:

- Assign each MFC a home region for inventory, order orchestration, and fulfillment writes.
- Replicate within three availability zones synchronously.
- Route catalog and tracking reads to nearby replicas.
- Replicate order history asynchronously to a disaster-recovery region.
- Keep courier and customer traffic pinned to the active region for the delivery.

During regional failure, do not accept inventory writes in two regions. Fence the former primary, promote a sufficiently current replica, and temporarily mark uncertain centers unavailable. A few minutes of order unavailability is preferable to overselling and duplicate dispatch.

## 15. Security, safety, and compliance

- TLS, encryption at rest, managed secrets, and least-privilege service identities.
- Tokenize payment instruments and minimize PCI scope.
- Encrypt precise addresses and courier locations; use strict retention and access auditing.
- Require step-up checks for account changes, refunds, and high-risk orders.
- Detect promotion abuse, account takeover, chargeback patterns, and courier GPS spoofing.
- Verify age and identity for alcohol or regulated products; record required evidence.
- Separate customer and courier contact details using masked communication.
- Restrict warehouse devices by role and MFC.
- Apply rate limits and signed requests to courier location ingestion.
- Support deletion, residency, tax, receipt, and labor-policy obligations by market.

## 16. Observability and SLOs

Track:

- Catalog latency, error rate, and stale-availability rate.
- Checkout conversion and failure by inventory, payment, or validation reason.
- Reservation conflicts, expirations, and inventory adjustment rate.
- Order acceptance-to-pick, pick duration, pack duration, and handoff wait.
- Courier assignment time, rejection rate, location freshness, and batch utilization.
- Promised versus actual delivery time by MFC and time of day.
- Substitution, cancellation, refund, chargeback, and support-contact rates.
- Saga age, outbox lag, dead-letter events, and reconciliation discrepancies.
- MFC queue depth, picker capacity, courier supply, and system health.

The customer-facing north-star SLI is:

```text
percentage of confirmed orders delivered complete and within promised window
```

Operational dashboards must be scoped per MFC; global averages hide local failures.

## 17. Major trade-offs

### Strong inventory versus available browsing

- Strong checkout reservation prevents overselling.
- Eventually consistent browse availability is faster and cheaper.
- Clearly treat the cart as advisory until checkout.

### Reserve before or after payment

- Reserving first prevents charging for unavailable items but briefly holds stock for failed payments.
- Authorizing first increases charge reversals and can charge when stock disappears.
- Use short inventory holds, then authorize.

### Central versus MFC-local authority

- Central systems simplify operations and analytics.
- MFC-local writes reduce latency and isolate failures.
- Use regional authority per MFC with centralized aggregated views.

### Immediate versus batched dispatch

- Immediate dispatch minimizes assignment delay.
- Short batching windows improve courier utilization and route economics.
- Bound optimization by the delivery promise.

### Accuracy versus sales

- Safety stock reduces substitutions and overselling.
- Aggressive sellable counts improve conversion but worsen customer experience.

## 18. Recommended interview presentation

A clear 45-minute sequence is:

1. Clarify delivery promise, MFC model, substitutions, and regulated goods.
2. Estimate orders, browse QPS, inventory rows, and courier updates.
3. Draw catalog, order saga, inventory, fulfillment, dispatch, and tracking.
4. Deep-dive into atomic inventory reservation and idempotent checkout.
5. Walk through picking, substitution, payment capture, and courier handoff.
6. Explain dispatch/ETA and high-volume location ingestion.
7. Cover MFC partitioning, failure reconciliation, and regional failover.
8. Finish with overselling, batching, and availability trade-offs.

The central design insight is:

> Keep inventory authority and fulfillment local to a specific MFC, coordinate checkout as an idempotent saga, and make every physical or financial transition auditable because real-world stock, payments, and courier state will inevitably diverge and require reconciliation.
