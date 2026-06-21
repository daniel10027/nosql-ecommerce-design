# Architecture Diagrams — Description

This folder documents two architecture views referenced from the schema
docs. (Rendered diagram images can be regenerated from these
descriptions using any diagramming tool — e.g., draw.io, Mermaid, or
Lucidchart — and committed here as `.png`/`.svg` files if a visual asset
is required for submission.)

## Diagram 1 — Part 1: Initial Architecture

```
                ┌────────────────────┐
   Browse /     │   products          │   Document store
   Search ───>  │   (document store)  │   - text index
                └────────────────────┘   - compound index (category, price)

                ┌────────────────────┐
   Checkout ──> │   orders            │   Document store
                │   (document store)  │   - sharded by customer.user_id
                └────────────────────┘   - single-document atomic writes

                ┌────────────────────┐
   Account ───> │   users             │   Document store
                │   (document store)  │   - unique index on email
                └────────────────────┘
```

Single primary region, document store, optimized for operational
read/write paths.

## Diagram 2 — Part 2: Refactored Architecture

```
 Client traffic
       │
       ▼
 ┌───────────────────────────────────────────────────────────┐
 │  Load Balancer / Multi-region entry point                  │
 └───────────────────────────────────────────────────────────┘
       │                                  │
       ▼                                  ▼
 ┌───────────────┐                 ┌───────────────┐
 │ orders         │  primary +     │ products       │ replicated,
 │ (sharded by    │  2 replicas,   │ (sharded,      │ read from
 │  user_id)      │  majority      │  cached via    │ nearest
 │                │  write concern │  Redis/CDN)    │ region
 └───────┬────────┘                 └───────────────┘
         │ change stream (CDC)
         ▼
 ┌───────────────┐
 │ event stream   │  wide-column / log store,
 │ (Kafka-like)   │  sharded by product_id + time bucket
 └───────┬────────┘
         │ stream aggregation
   ┌─────┴──────────────┐
   ▼                    ▼
┌────────────────────┐ ┌─────────────────────────┐
│ order_summary_daily │ │ product_sales_summary    │
│ (sharded by         │ │ (sharded by product_id)  │
│  date + region)      │ │                          │
└────────────────────┘ └─────────────────────────┘
   Analytics dashboards / BI tools read here
   (eventual consistency, fast aggregate reads)
```

Multi-region replica sets behind each shard provide high availability;
the analytics layer is fully decoupled from the operational write path.
