# DynamoDB modeling reference

Companion to `SKILL.md`. Distilled from the AWS DynamoDB Developer Guide "Data modeling" module. Use it as a pattern library: match the problem in front of you to the closest building block or worked package, then adapt.

## Access-pattern worksheet (fill this first)

| # | Access pattern (verb) | Entity | Read/Write | Frequency / peak | Key inputs | Ordering / filter | Serving table or index |
|---|---|---|---|---|---|---|---|
| 1 | e.g. getOrdersByCustomerForDateRange | Order | Read | high | customerId, dateRange | newest-first | GSI OrderByCustomerDate |

A design is "done" only when every row has a single-operation entry in the last column (GetItem or Query — never Scan, never a client-side join).

## Limits & cost facts to design against

- **Per-physical-partition throughput:** 1000 WCU, 3000 RCU strongly-consistent, ~6000 RCU eventually-consistent per second. Exceed it → `ThroughputExceededException` (throttling). Fix writes with **write sharding**; fix reads with a cache.
- **Item size:** 400 KB max. Writes bill in **1 KB** increments; reads in **4 KB** (4 KB strong = 1 RCU, eventual = 0.5 RCU).
- **`UpdateItem`** bills the **larger** of the pre/post item size, even for a one-attribute change → keep volatile attributes in small, separate items.
- **Transactions:** `TransactWriteItems` / `TransactGetItems`, all-or-nothing, **≤ 100 items**, single API call.
- **TTL:** async, runs ~every 6 h, can lag **up to 48 h**; deletes are free; attribute must be a **Number** epoch timestamp. Not for time-critical cleanup (locks/state).
- **GSI:** eventually consistent only; independent throughput; create any time; **sparse by default** (only items with the index key attrs appear).
- **LSI:** strong consistency possible; shares base capacity; **must be created with the table**; caps the item collection at 10 GB per PK value.

## Building blocks — detail & when to use

### Composite sort key
Encode hierarchy in the SK with a delimiter: `PARENT#CHILD#GRANDCHILD`. Query left-to-right with `begins_with`. E.g. shopping cart `CART#active` vs `CART#saved`; device logs `WARNING#2020-04-27`. Related items stay local; prefixes let different app areas write without collisions; subsets retrieved with zero wasted RCUs.

### Vertical partitioning
Break a nested document into individual items in one item collection keyed by SK prefixes. Enables independent 1-WCU updates, keeps items under the 1 KB write unit and 400 KB hard cap, and lets `begins_with` re-aggregate the document. This is single-table design in action (also usable across tables).

### Sparse index
GSI holds only items that possess its key attribute(s). Use for rare/flagged/"needs action" lookups: escalated complaints (`escalated_to`), due payments (`NextPaymentDate`), open orders. The GSI is small and cheap; you can even `Scan` it performantly. Add a composite SK (e.g. timestamp) to narrow further.

### GSI overloading
Store different entity meanings in one GSI's generic `GSI-PK`/`GSI-SK`, so a single index serves many patterns (invoice-by-id, payment-by-invoice, shipment-by-id all off one GSI). Fewer indexes to pay for and manage.

### Adjacency list (many-to-many)
Represent each relationship as an item in a shared collection; add an **inverted GSI** (swap PK/SK) to read the relationship from the other side. The canonical many-to-many pattern.

### Write sharding
For an unavoidable hot partition key beyond 1000 WCU: append `rand(0, N-1)` to the PK, with `N = ceil(peakWritesPerSec / 1000)` (20k writes → `rand(0,19)`). Reads must fan out across shards and merge; a scheduled job can pre-aggregate (e.g. vote totals). Apply to low-cardinality GSI PKs too (GSI throttling back-pressures the base table).

### TTL & TTL-for-archival
TTL: set a future epoch attribute; DynamoDB deletes for free when it passes. Archival: TTL delete emits a Stream record tagged `principal: dynamodb`; a Lambda stream filter routes those to S3/Glacier for cheap history + Athena/Redshift Spectrum analytics.

## Worked schema packages (adapt these)

### Social network — single table
Access: user info, follower list, following list, posts, post-likers, like-count, timeline.
Design: base table, generic `PK`. One item collection per user with SK-differentiated entities — `PK=<userId>` (info/count), `<userId>#follower`, `<userId>#following`, `<userId>#post`, `<postId>#likelist`, `<postId>#likecount`, `<userId>#timeline`. Posts use **ULID** postIds for chronological sort. Fan-out-on-write into each follower's `#timeline`, aged out with **TTL**. Celebrity `#likecount` throttling → buffer via queue, eventual accuracy. Transactions for follow (+count). *Blocks: composite sort key, vertical partitioning, TTL, transactions, atomic counters.*

### Gaming player profile — single table
Access: friends, all profile, all items, specific item, update character, update item count.
Design: `PK=PlayerID`, overloadable `SK` with `Type` attribute — `#METADATA#playerID`, `FRIENDS#playerID`, `ITEMS#itemId`. Small friend lists as a list attribute (1:1); large/bidirectional → many-to-many. `Query begins_with "ITEMS#"` for all items; if filtering by item type is common, promote it into the SK (`ITEMS#ItemType#ItemId`) instead of a FilterExpression. Updates via `UpdateItem` + **condition expression** (`currency >= :min`) and **atomic counters**; buy-item via **transaction**. On-demand capacity for unknown launch load. *Blocks: GSI overloading, composite sort key, transactions, condition expressions.*

### Complaint management — single table + 3 GSIs
Access (13): CRUD complaint, add/read comments, latest comment, by-customer, escalate, all-escalated, escalated-by-agent (newest-first), comments-by-agent (date range).
Design: `PK=complaint_id`, `SK=metadata` for the complaint; comments as `SK=comm#<date>#<id>` (**vertical partitioning + composite sort key**), `begins_with "comm#"` (+ `ScanIndexForward=False, Limit 1` for latest). `Customer_Complaint_GSI` (`customer_id` / `complaint_id`) for by-customer. `Escalations_GSI` (`escalated_to` / `escalation_time`) is **sparse** — only escalated items. `Agents_Comments_GSI` (`agent_id` / `comm_date`) for date-range. Non-OLTP needs handled off-table: Streams+Lambda notifications, Export-to-S3 + Athena analytics, TTL+Streams archival. Attachments in S3, URLs in the item. `TransactWriteItems` to add comment + update state atomically. *Blocks: vertical partitioning, composite sort key, sparse index, transactions.*

### Recurring payments — single table + 2 sparse GSIs
Access (7): create/update subscription, create receipt, due reminders by date, due payments by date, subscriptions-by-account, receipts-by-account.
Design: `PK=ACC#account_id`; `SK=SUB#<subId>#SKU<id>` and `SK=REC#<date>#SKU<id>` (item collection, 1:many). `GSI-1` PK=`NextReminderDate`, `GSI-2` PK=`NextPaymentDate` — both **sparse** (receipts not replicated) with **minimal projections**, so daily batch jobs query just the due items. `begins_with "SUB#"` / `"REC#"` for per-account reads. Receipts expire via **TTL**. *Blocks: sparse index, composite sort key, TTL, minimal projection.*

### Device-status monitoring (IoT) — single table + 2 GSIs
Access (7): create log, logs by device (recent-first), warning logs by device, logs by operator between dates, escalated logs by supervisor, +status, +date.
Design: `PK=DeviceID`, `SK=State#Date` (**composite sort key** — turns "warning logs" from a FilterExpression into `begins_with "WARNING"`). `GSI-1` (`Operator` / `Date`) for operator+date-range. `GSI-2` (`EscalatedTo` / `State#Date`) is **sparse** — only escalated logs appear. `ScanIndexForward=False` for recent-first. *Blocks: composite sort key, sparse index; contrast with the inefficient filter-expression starting point.*

### Online shop — single table + 2 overloaded GSIs
Access (16): entities by id (customer/product/warehouse/order/invoice/shipment), products/invoice/shipments by order, product inventory by product & by warehouse, orders by product+date, invoice/payment by invoice, shipments by warehouse, invoices/products by customer+date.
Design: generic `PK`/`SK` with `EntityType` + prefixes (`c#`,`p#`,`w#`,`i#`,`sh#`). Standalone entities use `PK=SK=id`. Order item collection holds order header + `orderItem`/`invoice`/`shipment`/`shipmentItem` (vertical partitioning). `GSI1` **overloaded** (product+date, invoice-by-id, shipment-by-id). `GSI2` **overloaded** (warehouse→shipments/products, customer+date→invoices/products) using `between` on date-prefixed SKs. many:many product↔warehouse modeled both directions. *Blocks: vertical partitioning, GSI overloading, composite sort key, adjacency list.*

### Relational order-entry — aggregate-oriented multi-table
Access (15): employee by id/name/warehouse/title/recent-hire, customer by id/rep, orders by customer+date / by rep / open-orders+date, items-on-order-by-product, inventory by product / by product+warehouse / total.
Design (multi-table, the guide's modern example): **Employee** table (PK `employee_id`) with 4 GSIs incl. `EmployeeByHireDate` using a **static PK** `"EMPLOYEE"` + `hire_date` SK for date-range scans. **Customer** table with `CustomerByAccountRep` GSI (**denormalized** `account_rep_id`). **Order** table **vertically partitioned** (header `SK=order_id`, items `SK=product_id`); `OpenOrdersByDate` GSI is **sparse + write-sharded** (`status`+`shard`, 5 shards for 5000 WPS, parallel query + merge); `ProductInOrders` GSI from order-item records. **Product** table uses the **item-collection pattern** (`SK=product_id` metadata, `SK=warehouse_id` inventory) to fold inventory in and drop a whole table (~50% cost saving), with denormalized `total_inventory`. *Shows: single-table isn't mandatory — separate tables for low-correlation entities + strategic GSIs, denormalization, sparse+sharded indexes, item collections.*

## NoSQL Workbench

Validate and share a model in **NoSQL Workbench** (visual data modeling, sample data, query preview). **Facets** represent each access pattern as a view of the table (a modeling aid only; not a real DynamoDB construct). Every AWS package above ships as an importable JSON model in the `aws-samples/aws-dynamodb-examples` and `amazon-dynamodb-design-patterns` GitHub repos.
