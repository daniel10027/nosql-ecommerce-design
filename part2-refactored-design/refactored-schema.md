# Part 2: Refactored Schema for Analytics & High Availability

## 1. New Requirements

- **Analytics:** run large-scale analytical queries on product trends and
  sales data (e.g., revenue by category by day, top sellers, regional
  demand).
- **High Availability:** the system must tolerate node/region failure and
  keep serving traffic as the user base grows (availability + partition
  tolerance, per the CAP theorem).

## 2. Strategy Selected

All three suggested techniques are applied together, each solving a
different part of the problem:

| Technique | Solves |
|---|---|
| **Sharding** | Horizontal scale for both operational writes (orders) and analytical scans (events, sales) |
| **Replication** | High availability and fault tolerance across nodes/regions |
| **Denormalization (pre-aggregation)** | Fast analytical reads without scanning raw order history |

This pushes the overall system toward an **AP-leaning** design (per CAP):
favor availability and partition tolerance for catalog browsing and
analytics, while keeping **strong consistency narrowly scoped** to the
write path of an individual order (you don't want two warehouses both
thinking they shipped the last unit).

## 3. Sharding Strategy

| Collection | Shard Key | Rationale |
|---|---|---|
| `orders` | `customer.user_id` (hashed) | Even write distribution; most queries filter by customer, so they stay shard-local |
| `products` | `category_path` (range) or hashed `_id` | Hashed `_id` avoids hotspotting on popular categories during flash sales |
| `events` (new, see §5) | `product_id` (hashed) + time bucket | Spreads high-velocity clickstream/order-event writes; time bucket keeps recent data together for analytics scans |
| `order_summary_daily` (new) | `date` + `region` | Analytical queries are typically date-range scans; co-locating by date avoids scatter-gather across all shards |

Each shard is itself replicated (see §4), so sharding handles
**horizontal scale**, while replication handles **fault tolerance** within
each shard.

## 4. Replication Strategy

- Each shard runs as a **replica set** (e.g., 1 primary + 2+ secondaries),
  with secondaries distributed across **availability zones / regions**
  (e.g., one replica in Abidjan/West Africa region, one in Europe, one in
  a second AZ) to survive a full data-center loss.
- **Write path (orders):** writes go to the primary with a **majority
  write concern** — acknowledged once a majority of replicas confirm.
  This keeps order-status consistency strong (no lost or conflicting
  order states) while still tolerating a single node failure without
  downtime.
- **Read path (catalog, analytics):** reads can target secondaries with
  **eventual consistency**, which is acceptable — a product page being a
  few hundred milliseconds stale, or an analytics dashboard being a few
  minutes behind, does not harm the business the way a lost order would.
- **Automatic failover:** if a primary goes down, the replica set elects
  a new primary within seconds, satisfying the availability requirement
  without manual intervention.

This is the core trade-off being made explicitly: **strong consistency
is kept only where correctness truly requires it (order writes);
everything else is relaxed to eventual consistency in exchange for
availability and read scalability**, consistent with the CAP theorem
(a partitioned, highly available system cannot also guarantee strong
consistency everywhere).

## 5. Denormalization for Analytics

Querying raw `orders` documents for trend analysis (e.g., "total revenue
per category per day across 50 million orders") is expensive and
competes with operational traffic. Instead, the schema introduces
**pre-aggregated, denormalized read models**, updated asynchronously
(via change streams / event pipeline / scheduled batch jobs — a classic
CQRS pattern: writes go to the operational store, reads for analytics
go to derived, denormalized stores).

### 5.1 `order_summary_daily` (wide-column style aggregate)

```json
{
  "_id": "2026-06-20|footwear|CI",
  "date": "2026-06-20",
  "category": "footwear",
  "region": "CI",
  "order_count": 1342,
  "units_sold": 1890,
  "gross_revenue": 168_402.50,
  "avg_order_value": 125.48,
  "top_products": [
    { "product_id": "prd_1a92c", "units_sold": 210, "revenue": 18897.90 }
  ]
}
```

- Partition key: `date` + `region` (sometimes modeled as a wide-column
  table with `date#region` as the row key and category metrics as
  columns — this maps naturally onto Cassandra/Bigtable-style wide-column
  stores, which excel at this kind of time-bucketed write-heavy,
  scan-friendly aggregate).
- Built by a streaming aggregation job (e.g., Kafka + a stream processor)
  consuming order-completed events, so it stays close to real-time
  without touching the operational `orders` collection.

### 5.2 `product_sales_summary` (per-product rollup)

```json
{
  "_id": "prd_1a92c|2026-06",
  "product_id": "prd_1a92c",
  "month": "2026-06",
  "units_sold": 4920,
  "revenue": 442_750.80,
  "return_rate": 0.018,
  "rank_in_category": 3
}
```

- Supports "product trends over time" queries directly, without
  recomputing from raw orders.
- Recomputed incrementally (e.g., nightly batch + intraday deltas).

### 5.3 `events` (high-velocity raw stream, optional but recommended)

To support both real-time personalization and future analytics needs
beyond what's pre-aggregated, raw clickstream/order events are captured
in an **append-only, time-series-oriented store** (e.g., a wide-column
store like Cassandra, or a managed stream/log like Kafka feeding a data
lake):

```json
{
  "event_id": "evt_77a1",
  "event_type": "order_completed",
  "product_id": "prd_1a92c",
  "user_id": "usr_8f3a2",
  "region": "CI",
  "amount": 89.99,
  "ts": "2026-06-20T10:16:30Z"
}
```

- This raw layer is the **source of truth for re-deriving** aggregates if
  business logic changes (e.g., "we now want revenue net of returns") —
  pre-aggregated tables can always be rebuilt by replaying this stream.

## 6. Updated Relationship / Data Flow Diagram (textual)

```
                 ┌───────────────┐
 write           │   orders      │  (sharded by user_id, replicated,
 (strong ──────> │ (operational) │   majority write concern)
 consistency)    └───────┬───────┘
                         │ change stream / CDC
                         ▼
                 ┌───────────────┐
                 │ event stream  │  (wide-column / log, sharded by
                 │ (Kafka-like)  │   product_id + time bucket)
                 └───────┬───────┘
                         │ stream aggregation
            ┌────────────┴────────────┐
            ▼                         ▼
 ┌─────────────────────┐   ┌─────────────────────────┐
 │ order_summary_daily  │   │ product_sales_summary   │
 │ (analytics, eventual │   │ (analytics, eventual    │
 │ consistency, sharded │   │ consistency)            │
 │ by date+region)       │   └─────────────────────────┘
 └─────────────────────┘
```

Catalog reads (`products`) and analytics reads hit replicas / derived
stores and tolerate eventual consistency; only the order write path
keeps strong, majority-acknowledged consistency.

## 7. Trade-offs Summary

| Dimension | Before (Part 1) | After (Part 2) | Trade-off accepted |
|---|---|---|---|
| **Consistency** | Strong everywhere within a document | Strong only on order writes; eventual elsewhere | Analytics/catalog can be briefly stale in exchange for availability and speed |
| **Availability** | Single-region, single point of failure on primary loss | Multi-region replica sets, automatic failover | Slightly higher write latency due to majority write concern |
| **Query performance (analytics)** | Would require full collection scans over raw orders | Pre-aggregated reads, millisecond dashboard queries | Extra storage for aggregates; added pipeline complexity and small lag |
| **Scalability** | Single shard key per collection, fine at moderate scale | Multiple shard keys tuned per access pattern, horizontal scale-out | More operational complexity (rebalancing, cross-shard query planning) |
