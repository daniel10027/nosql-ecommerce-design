# Reflection Report

**Word count: ~228 words**

The biggest challenge in the refactor was reconciling two consistency
models inside one system. The original order schema relied on strong,
single-document consistency, which works well operationally but cannot
scale to large analytical scans without competing with live traffic. I
had to explicitly decide where strong consistency was actually necessary
(order writes and status transitions) versus where it could be relaxed
(catalog reads, analytics dashboards), rather than applying one
consistency level everywhere. Choosing shard keys was the second
challenge: a key that distributes writes evenly for orders is not the
same key that makes analytical date-range queries efficient, so
different collections ended up with different shard keys.

The analytics requirement pushed the design from a single operational
store toward a CQRS-style split: raw operational writes flow through a
change stream into pre-aggregated, denormalized read models
(`order_summary_daily`, `product_sales_summary`). This avoided scanning
millions of raw order documents per dashboard query. The high-
availability requirement pushed the design toward multi-region replica
sets with majority write concern for orders and eventual consistency
for replica reads, accepting the CAP trade-off explicitly rather than
implicitly.

The refactor improved scalability by giving each collection a shard key
suited to its access pattern, improved availability through replicated,
geographically distributed replica sets with automatic failover, and
improved analytical query performance by orders of magnitude, since
dashboards now read small pre-aggregated documents instead of scanning
the operational order history.
