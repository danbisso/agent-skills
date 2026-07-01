---
name: dynamodb-modeler
description: Design, review, and find gaps in Amazon DynamoDB data models — access-pattern-first schema design, single- vs multi-table foundations, partition/sort key and GSI/LSI design, the building-block patterns (composite sort keys, sparse indexes, GSI overloading, adjacency lists, write sharding, vertical partitioning, TTL), and relationship modeling. Use when designing a DynamoDB table for a new app or idea, reviewing/auditing an existing DynamoDB schema, mapping access patterns to keys and indexes, deciding single-table vs multi-table, diagnosing hot partitions / Scans / filter-heavy queries, or when someone asks "will this model scale / what am I missing."
---

# dynamodb-modeler

Design and critique DynamoDB schemas the way the AWS DynamoDB Developer Guide's data-modeling module teaches: **start from access patterns, not entities.** This skill covers the foundations, the reusable building blocks, relationship modeling, and — most importantly — a repeatable workflow for finding the gaps in a model someone already has or is proposing.

Cross-links: whether DynamoDB is even the right store (vs Aurora/RDS/S3) → `aws-architect`; provisioning the table, GSIs, TTL, and streams in IaC → `aws-cdk`. This skill is about the *shape of the data*, not the infrastructure.

## The one rule that drives everything

**Do not design a DynamoDB schema until you know the questions it must answer.** Unlike an RDBMS — where you normalize first and query later — DynamoDB has no JOINs, and the runtime cost of an operation is fixed by the key design, not the query. NoSQL modeling writes data in a *pre-joined* shape so each access pattern is satisfiable by a single `Query` or `GetItem`. If you don't know the access patterns, you cannot design the keys, full stop.

So the first move — for a new idea OR an audit — is always to reconstruct the access pattern list.

## Step 0 — Gather the prerequisites

Before proposing or judging any schema, assemble these four things. If the user hasn't given them, ask for them (or infer from the app and state your assumptions):

1. **Entities + relationships (an ER diagram).** What are the nouns and how do they relate (1:1, 1:many, many:many)?
2. **Access patterns** — the full list of reads and writes, phrased as verbs: `getOrdersByCustomerForDateRange`, `getLatestCommentByComplaintId`. For a new app, mine user stories; for an existing app, mine query logs. This list is the spec.
3. **Volumes & throughput per entity** — item counts, item sizes, reads/writes per second, peak vs steady, seasonality. This drives partition-key cardinality and sharding decisions.
4. **Data retention** — what ages out, and after how long (drives TTL / archival).

Write the access patterns into a table with columns: **Pattern · Entity · read/write · frequency · key inputs · ordering/filter needs**. This is the worksheet every later decision is checked against (template in `reference.md`).

## Foundations — single-table vs multi-table

The foundation choice is the first design decision. Modern AWS guidance is nuanced — it is **not** "always single-table."

| Choose **single-table** when… | Choose **multi-table** when… |
|---|---|
| Access patterns frequently need multiple entity types **together** (data locality wins) | Entities have **independent** operational characteristics / low access correlation |
| Entities share operational characteristics (backup, encryption, capacity, streams) | Different backup, encryption-key, table-class, or stream-processing needs per entity |
| You want fewer tables to manage and smoother aggregate traffic | You use GraphQL resolvers (one resolver → one table) or higher-level ORMs |
| You can absorb the steeper design learning curve | Simpler mental model matters more than cross-entity query locality |

Single-table works because items with the same partition key (an **item collection**) live on the same partition, so one `Query` returns several entity types pre-joined. Its costs: steep learning curve, all-or-nothing backups, shared encryption, all changes hit one stream, harder with GraphQL/ORMs.

A pragmatic middle path the guide now endorses: **aggregate-oriented multi-table** — separate tables per entity, but embed data that's always read together (Order + OrderItems), use item collections for identifying relationships (Product + Inventory), and add strategic GSIs (sometimes denormalizing a foreign key like `account_rep_id`) for cross-entity queries. Don't treat single-table as dogma.

## The primary-key toolkit

- **Partition key (PK / HASH)** — hashed to place data. Must be **high-cardinality and evenly accessed** or you get hot partitions. Always require the exact value at query time.
- **Sort key (SK / RANGE)** — optional; enables item collections, one-to-many, ordering, and range ops (`begins_with`, `between`, `>`, `ScanIndexForward` for asc/desc). Apply it **left-to-right** like traversing a tree.
- **Generic key names + type attribute** — name keys `PK`/`SK` (not `UserId`) and add an `EntityType`/`Type` attribute so one table/index can hold many entity types and stay future-proof. Use readable prefixes (`c#`, `p#`, `ORDER#`).
- **GSI** — different PK and/or SK than the base table, **independent throughput**, **eventually consistent only**, creatable any time. The workhorse for "query by another dimension."
- **LSI** — same PK, alternate SK, **shares base-table capacity**, supports strong consistency, but must be created **at table creation** and caps that partition's item collection at 10 GB. Prefer GSIs unless you specifically need strong consistency on an alternate sort.
- **Projections** — project only the attributes an index's patterns need (`KEYS_ONLY` / `INCLUDE`) to cut storage and write cost.

## Building blocks (the pattern catalog)

Pattern-match every access pattern to one of these. Full detail + when-to-use in `reference.md`.

- **Composite sort key** — encode a hierarchy `PARENT#CHILD#GRANDCHILD` in the SK; query prefixes with `begins_with` to retrieve exactly the subtree you need (e.g. `CART#` vs `WISHLIST#`, `WARNING#<date>`). Zero wasted RCUs.
- **Vertical partitioning** — split a big document/entity into multiple items in one item collection (SK prefixes) instead of one nested JSON blob. Keeps items <1 KB (write billing unit), allows independent 1-WCU updates, works around the 400 KB item limit, and lets `begins_with` re-aggregate cheaply.
- **Sparse index** — a GSI only holds items that *have* the index's key attribute. Perfect for "find the rare/flagged item" (escalations, `NextPaymentDate`, open orders): tiny, cheap GSI you can even `Scan` performantly.
- **GSI overloading** — reuse one GSI's generic `GSI-PK`/`GSI-SK` to serve *multiple* entity access patterns by storing different meanings per entity type. Fewer indexes, more patterns.
- **Adjacency list** — model **many-to-many** by making both sides items in a shared collection; a GSI that flips PK/SK reads the relationship from the other direction.
- **Write sharding** — for a single hot PK exceeding **1000 WCU/partition**, append a random suffix (`rand(0, N-1)`, `N = ceil(peakWrites/1000)`) to spread writes; re-aggregate at read time (often via a scheduled job). Also apply to low-cardinality GSI PKs.
- **Time to live (TTL)** — free, async deletes of items whose numeric epoch attribute is in the past (runs ~every 6h, can lag up to 48h — don't use for locks). Strongly recommended to age out data and shrink cost.
- **TTL for archival** — pair TTL with Streams: filter stream records where `principal: dynamodb` and push the expiring items to S3/Glacier for cheap long-term storage + Athena analytics.

## Relationship modeling

- **1:1** — same PK & SK, or an embedded attribute/map.
- **1:many** — the core pattern: an **item collection** (one PK, many SKs). `Query PK` gets all children; `begins_with` on SK narrows to one child type. This replaces foreign keys.
- **many:many** — adjacency list + an inverted GSI (swap PK/SK) to traverse both directions.
- **Eliminate JOINs by denormalizing** — store related data pre-joined / duplicate a foreign key into a GSI when it buys a single-query read. Accept controlled duplication; that's the trade for constant-time reads at scale.
- **Transactions** — `TransactWriteItems` / `TransactGetItems`, all-or-nothing, **max 100 items**, single API call (no long-lived locks). Use for "add follower + increment count," "deduct currency + grant item." Use **condition expressions** (`currency >= :min`) and **atomic counters** for safe concurrent updates.
- **Aggregations/counters that can throttle** (celebrity like-count) — don't force real-time; buffer through a queue and update periodically. A post's like count rarely needs to be 100% instantaneous.

## Finding gaps in a model (the review workflow)

This is the primary job: given an existing or proposed model, produce ranked, concrete findings. Run every access pattern through this gauntlet:

1. **One access pattern → one operation?** Each pattern must resolve to a single `GetItem` or `Query` on the base table or one index. If a pattern needs a **`Scan`** or a **cross-item client-side join**, that's a gap — add/adjust a key or GSI.
2. **Filter-expression abuse.** A `FilterExpression` runs *after* the read and bills the same RCUs. If a query reads many items and filters most away, promote the filtered attribute into the SK (composite key) or a GSI. (A filter is fine only when the excluded ratio is low or the query is rare.)
3. **Partition-key cardinality / hot partitions.** Is the PK high-cardinality and evenly accessed? Flag low-cardinality PKs, monotonic keys, and any single PK expected to exceed **1000 WCU / 3000 RCU (strong) / ~6000 RCU (eventual)** per partition → recommend write sharding or a read cache.
4. **Item size & write cost.** Items approaching **400 KB**, or large items updated often (writes bill per 1 KB, and `UpdateItem` bills the *larger* of before/after size) → recommend vertical partitioning; separate volatile "count" items from stable "info" items.
5. **Sparse-index opportunities.** "Find the flagged/rare/due item" patterns satisfied today by scanning the whole set → recommend a sparse GSI on the rare attribute.
6. **Index sprawl vs overloading.** Multiple single-purpose GSIs that could collapse into one overloaded GSI → recommend consolidation. Also check projections aren't copying unnecessary attributes.
7. **Ordering & pagination.** Patterns needing sorted/most-recent results need a sortable SK (timestamp or **ULID**) and `ScanIndexForward`/`Limit`; flag reliance on client-side sorting.
8. **Relationship modeling.** many:many stored without an adjacency list / inverted GSI; 1:many stored as multiple tables requiring client joins → flag.
9. **Retention & lifecycle.** Unbounded item collections (timelines, logs, receipts) with no TTL → recommend TTL (+ archival to S3 if compliance needs history).
10. **Consistency & transactions.** Multi-item invariants written without a transaction (risk of partial writes); >100 items in one transaction (illegal); strong-consistency needs on a GSI (impossible — GSIs are eventually consistent) → flag.
11. **Analytics leaking into OLTP.** Weekly/reporting/aggregate queries run against the live table → recommend Export to S3 + Athena rather than contorting the key schema.

For each gap, state it as: **access pattern → what breaks at scale (hot partition / Scan / wasted RCU / partial write) → the specific fix (which key or index, which building block).** Rank by blast radius (throttling/correctness first, cost second, polish last). Never hand back "looks fine" without having walked all 11 checks.

## Worked schema packages (pattern library)

`reference.md` contains six complete, real designs distilled from the AWS guide — **social network, gaming profile, complaint management, recurring payments, device-status monitoring, online shop**, plus the relational order-entry example. Each lists its access patterns and the final PK/SK + GSI design and which building blocks it used. When modeling or auditing, find the closest package and adapt it rather than starting from a blank page — most business domains rhyme with one of these.

## Delivering a design

- Produce the **access-pattern table** (pattern → table/index → operation → PK value → SK value → filters), the **base table** key schema, and each **GSI/LSI** with its keys and projection — the same summary format the AWS guide uses. This mapping IS the deliverable; it proves every pattern is served.
- Suggest importing the final model into **NoSQL Workbench** (visual modeling, facets per access pattern, query preview) to validate and share it.
- Note throughput mode: start new/unknown workloads on **on-demand**; move to provisioned once traffic is predictable.

## Anti-patterns to call out

- Designing keys before listing access patterns (relational-brain modeling — a table per entity "because that's how it's done").
- `Scan` in any hot path; `FilterExpression` doing the work a key should do.
- Low-cardinality or monotonically increasing partition keys.
- One giant nested-JSON item you must read/rewrite wholly for a small change.
- Unbounded item collections with no TTL.
- Real-time global counters on a single hot key without sharding/buffering.
- Adding a GSI per query instead of overloading; over-projecting attributes.
