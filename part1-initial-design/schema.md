# Part 1: Initial Schema Design

## 1. Requirements Recap

- Users browse and search for products.
- Orders must store customer info, purchased items, and delivery status.
- The system must support thousands of transactions per second (TPS).

## 2. Key Entities

| Entity | Description |
|---|---|
| **User** | Account holder: profile, shipping addresses, auth info |
| **Product** | Catalog item: name, description, price, category, stock, search tags |
| **Order** | A purchase event: customer reference, line items, status, payment, delivery |
| **Category** (sub-entity) | Used to organize/filter products, embedded as tags/path on Product |

## 3. Chosen NoSQL Model: Document Store

A **document database** (e.g., MongoDB, Amazon DocumentDB, Couchbase) is the
best fit for the initial design because:

- **Products** have a semi-structured, evolving shape (different categories
  need different attributes — a t-shirt has "size" and "color", a laptop has
  "CPU" and "RAM"). Documents handle this naturally without a rigid schema
  or sparse columns.
- **Orders** are naturally hierarchical (one order → many line items, one
  shipping address, one payment record). Storing an order as a single
  document avoids JOINs and makes a single order fully readable/writable
  in one operation — important for write throughput and atomicity at the
  document level.
- Document stores scale horizontally via **sharding**, which directly
  supports the "thousands of TPS" requirement.

A pure key-value store was considered for the product catalog (fast
lookups by product ID), but it would not support the catalog's secondary
query needs (search by category, price range, text search) without
maintaining separate manual indexes — a document store with built-in
secondary and text indexes is more practical.

## 4. Schema Design

### 4.1 `users` collection

```json
{
  "_id": "usr_8f3a2",
  "email": "amara.kone@example.com",
  "name": "Amara Koné",
  "phone": "+225-07-12-34-56-78",
  "addresses": [
    {
      "label": "home",
      "line1": "12 Rue des Jardins",
      "city": "Abidjan",
      "country": "CI",
      "is_default": true
    }
  ],
  "created_at": "2025-03-11T09:20:00Z",
  "loyalty_tier": "silver"
}
```

- **Indexed fields:** `email` (unique index, for login/lookup).
- Addresses are **embedded** (small, bounded, rarely queried independently
  of the user) — a classic 1-to-few embedding decision.

### 4.2 `products` collection

```json
{
  "_id": "prd_1a92c",
  "sku": "SHOE-NK-AIR-42",
  "name": "Nike Air Runner",
  "description": "Lightweight running shoe with breathable mesh upper.",
  "category_path": ["footwear", "running", "men"],
  "tags": ["nike", "running", "lightweight", "mesh"],
  "price": {
    "amount": 89.99,
    "currency": "USD"
  },
  "stock_qty": 482,
  "attributes": {
    "size": "42",
    "color": "black/white",
    "brand": "Nike"
  },
  "rating_avg": 4.6,
  "rating_count": 1290,
  "updated_at": "2026-06-10T14:02:00Z"
}
```

- **Indexed fields:**
  - Single-field index on `sku` (unique, used at checkout/inventory sync).
  - Compound index on `category_path` + `price.amount` (supports
    "browse by category, sort by price" queries).
  - **Text index** on `name`, `description`, `tags` for full-text search
    (or delegate to a dedicated search engine like Elasticsearch/Atlas
    Search fed via change streams — see note below).
- This collection is **read-heavy**; it is the primary candidate for
  caching (e.g., Redis in front of it) and read replicas.

> **Note on search:** for true full-text relevance ranking, fuzzy
> matching, and faceted filters, a dedicated search index (Elasticsearch
> or a managed equivalent) sitting alongside the document store is the
> realistic production answer. The document store remains the system of
> record; the search engine is a derived, eventually-consistent index.

### 4.3 `orders` collection

```json
{
  "_id": "ord_5c0e7",
  "order_number": "CI-2026-000451",
  "customer": {
    "user_id": "usr_8f3a2",
    "name": "Amara Koné",
    "email": "amara.kone@example.com",
    "shipping_address": {
      "line1": "12 Rue des Jardins",
      "city": "Abidjan",
      "country": "CI"
    }
  },
  "items": [
    {
      "product_id": "prd_1a92c",
      "sku": "SHOE-NK-AIR-42",
      "name": "Nike Air Runner",
      "unit_price": 89.99,
      "quantity": 1,
      "line_total": 89.99
    }
  ],
  "totals": {
    "subtotal": 89.99,
    "shipping": 5.00,
    "tax": 4.50,
    "grand_total": 99.49,
    "currency": "USD"
  },
  "status": "processing",
  "status_history": [
    { "status": "created", "at": "2026-06-20T10:15:00Z" },
    { "status": "processing", "at": "2026-06-20T10:16:30Z" }
  ],
  "payment": {
    "method": "card",
    "provider_ref": "pay_9f12ab",
    "paid": true
  },
  "created_at": "2026-06-20T10:15:00Z"
}
```

- **Indexed fields:**
  - `customer.user_id` (compound with `created_at` descending) — "show me
    my orders, most recent first."
  - `status` (for operational dashboards: "all orders in `processing`").
  - `order_number` (unique).
- The customer snapshot and item details are **embedded** (denormalized)
  at order-creation time. This is deliberate: an order is a legal/financial
  record of what was true *at the time of purchase* (price, name,
  address). It must not silently change if the product price or the
  user's address changes later. Embedding gives both correctness and
  single-document write/read performance.
- Only `product_id` and `user_id` are kept as **references** back to the
  catalog/user collections, for traceability and analytics joins.

## 5. Relationships

```
User (1) ───── places ─────> (many) Order
Order (1) ───── contains ──── (many) OrderItem [embedded]
OrderItem ───── references ── Product (by product_id, denormalized snapshot)
Product (many) ── belongs to ── Category [embedded as category_path]
```

No foreign-key constraints exist (NoSQL stores don't enforce them);
referential integrity is handled at the application layer.

## 6. Scalability & Consistency Considerations

| Concern | Decision |
|---|---|
| **High write throughput for orders** | Orders are sharded by `customer.user_id` (or a hashed order ID) so writes spread evenly across nodes; each order write is a single-document operation, which is atomic by default in document stores. |
| **Consistency for order status** | Each order is a single document, so status transitions are atomic within that document — no risk of partial writes leaving an order half-updated. The system uses **strong/read-your-write consistency on the primary** for the order being written, which matters for "did my order go through." |
| **Catalog read scaling** | Products are read far more than written. Add read replicas and a caching layer (Redis/CDN) in front of the `products` collection; eventual consistency is acceptable here (a few hundred ms staleness on a stock count is fine). |
| **Search performance** | Text/secondary indexes on `products`; optionally offload to a search-specific engine fed by change streams. |

This initial design optimizes for **operational** workloads: fast catalog
reads and reliable, high-throughput order writes. It does **not** yet
address analytical queries (e.g., "total revenue by category last
quarter") or multi-region availability — that is the focus of Part 2.
