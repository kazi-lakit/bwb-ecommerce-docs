# Blocks Platform Feature Suggestions

**For:** the SELISE Blocks platform team.
**From:** the E-Commerce/POS/Inventory platform project (`ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md`, `ECOMMERCE_PLATFORM_ON_BLOCKS.md`).
**Why this document exists:** that project builds entirely on Blocks services with **no new backend service of its own** — no worker, no cron daemon, no custom API server. Every item below is something the *project* worked around with a client-only or manual mitigation specifically because standing up infrastructure to solve it wasn't an option. Each entry names the Blocks service it belongs to, what's missing, why it matters for this project, and the interim mitigation in use today.

**Data Gateway and Storage — the two services this project uses most heavily — have a dedicated, deeper companion document:** `DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md`. It goes further than the entries below: it's a source-level audit (with `file:line` references into `blocks-data`) covering the same feature gaps in more technical detail, plus a set of security findings — including two critical ones — that don't belong in a product-shaped gap list like this one. Read that document for anything tagged Data Gateway or Storage below; treat it as authoritative for those two services.

No item here is blocking — the project ships without any of them. They're ranked by how much they'd improve the product if delivered.

---

## P0 — closes a real product gap today

### 1. [New capability — home TBD, likely Data Gateway or a new "Blocks Functions"] Managed inbound trigger / scheduled execution

**What's missing:** any Blocks-native way for (a) an external system (a payment provider's webhook) to push a trusted, authenticated event into a project, or (b) code to run on a schedule, without the customer operating their own server.

**Why it matters:** this single gap is behind two separate weak spots in the project:

- **Payment confirmation.** A PSP confirms payment by calling a webhook on a server you control; a browser can't receive one, and trusting the browser's own claim that payment succeeded is exactly the fraud a webhook prevents. Today the project uses **manual staff-side confirmation** (a permission-gated "Confirm payment" action, checked by a human against the PSP's dashboard) instead. Correct, but slow and human-dependent.
- **Reservation expiry, balance/ledger reconciliation, replenishment suggestions, dashboard summaries, low-stock sweeps.** All of these want "run this periodically, independent of any user request." Today the project uses **lazy, client-triggered checks** (a reservation past its expiry is treated as released the next time *any* client reads it) and **on-demand reports** a staff member triggers. Both degrade gracefully but leave gaps a real scheduler wouldn't have.

**Current mitigation:** manual confirmation + lazy/on-demand checks, documented in `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §6.1–§6.3.

**What would satisfy this:** a Blocks-managed feature — e.g. a project-scoped inbound webhook endpoint that verifies a provider's signature and writes a signed event into the project's own event stream, plus scheduled/triggered execution of project-defined logic — that remains a Blocks service the customer configures, not a server the customer runs. If the workflow engine referenced by `DataChangeEvent` (`blocks_logic_workflow_data_trigger_listener`, consumed by `blocks-utilities-net`) already does part of this, exposing it through the CLI/SDK and documenting its trigger model would let this project adopt it directly — see item 10.

---

### 2. [Data Gateway] Atomic increment (`$inc`) on update mutations

**What's missing:** every generated `updateX`/`updateManyX` mutation issues `$set` only (`blocks-data/server/DataGateway.DomainService/Repositories/MongoCollectionOperations.cs:54`). There is no way to say "increment this field by N," only "set it to this value."

**Why it matters:** every inventory quantity change (reserve, commit, release, receive, adjust, count) becomes read → compute the new value client-side → guarded conditional write → check `totalImpactedData` → retry on conflict. An `$inc`-based mutation with the same `where` guard (e.g. `AvailableToSell: {gte: qty}`) would make the whole operation atomic and unconditional-on-read, eliminating the read, the version field, and the retry loop.

**Current mitigation:** the compare-and-swap pattern in `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1 — a `Version` field plus a `where` guard on the derived `AvailableToSell` bucket, applied in one `updateX(where:, input:)` call, with a bounded retry on `totalImpactedData === 0`.

**Impact if delivered:** the single highest-leverage change for any inventory-shaped workload on Blocks — removes an entire class of retry-loop code from every client that touches stock.

---

### 3. [Data Gateway] Real unique-index enforcement behind `IsUniqueData`

**What's missing:** a field marked `IsUniqueData: true` is enforced by a check-then-insert query (`ValidateUniquenessOrThrowAsync`, `blocks-data/server/DataGateway.DomainService/Services/Implementations/MutationService.cs:309`), not a database constraint. Two concurrent requests with the same value can both pass the check.

**Why it matters:** idempotency keys (on orders, POS transactions, payments, inventory movements, goods receipts) are the project's only defense against double-processing a retried request — including a POS terminal replaying a queued sale after being offline for days. A schema-level `IsUniqueData` flag reads like a guarantee and isn't one under concurrency.

**Current mitigation:** a real MongoDB unique index created out-of-band via `POST /schemas/indexes` (`SchemaIndexController`) on every `IdempotencyKey` field, since that's the only mechanism that's actually atomic.

**What would satisfy this:** either back `IsUniqueData: true` with a real unique index at push time, or clearly document today's behavior as a non-atomic convenience check so nobody else relies on it as a constraint.

---

## P1 — meaningfully improves correctness or reduces client-side complexity

### 4. [Data Gateway] Index declaration in the schema file / `blocks data sync`

**What's missing:** `SchemaIndexController` (single-field or compound, unique or not, up to 15 per schema) is REST-only — no `blocks data index *` CLI command, no SDK method, and no way to declare an index inside `blocks/data/schemas/<Schema>.json`.

**Why it matters:** unique indexes are the only real idempotency mechanism (item 3), yet they're the one part of this schema-as-code workflow that can't be pushed, reviewed, or reloaded alongside the schema itself — they have to be scripted separately and re-applied by hand.

**Current mitigation:** a standalone script (outside `blocks data sync`) that calls the REST endpoint directly for each required index, checked into the repo next to the schema files it corresponds to.

---

### 5. [Data Gateway] Transactional multi-collection batch mutation

**What's missing:** no way to commit writes across more than one collection (or more than one document) as a single MongoDB transaction.

**Why it matters:** an inventory movement always pairs a balance update with an append to the immutable ledger (`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.5); today those are two separate calls, ordered so the correctness-critical one (the guarded balance change) goes first, with the ledger write retried on failure and a reconciliation pass as the safety net. A transactional batch would remove that whole reconciliation need.

**Current mitigation:** ordered writes (balance, then ledger) plus an on-demand reconciliation report a human can trigger from the backoffice.

---

### 6. [Data Gateway] Return the updated document from `updateX`

**What's missing:** mutation responses carry only `acknowledged` and `totalImpactedData` — never the document's new state.

**Why it matters:** every stock-changing operation currently needs a follow-up read to know the resulting balance, which costs a full round trip at exactly the point (POS checkout) where the plan sets a p95 < 1s target.

---

### 7. [Data Gateway] TTL index support

**What's missing:** no way to declare a MongoDB TTL index through the schema/index tooling.

**Why it matters:** a TTL index on a reservation's `ExpiresAt` would let the database itself expire abandoned reservations with zero scheduler and zero client involvement — directly closing the residual gap in item 1's lazy-expiry mitigation (a reservation nobody's client ever revisits currently stays held).

---

### 8. [Data Gateway] Reliable delivery for `DataChangeEvent`

**What's missing:** `DataChangeEventPublisher.PublishAsync` catches and logs its own exceptions (`blocks-data/server/DataGateway.DomainService/Services/DataChangeEventPublisher.cs:74`) — a successfully committed write can still fail to publish its event, silently.

**Why it matters:** anything downstream that reacts to data changes (notifications, the workflow engine in item 10, future reporting) can miss events with no signal that it happened.

**What would satisfy this:** an outbox — persist the event in the same write path as the document change, deliver from there with retry, so "committed" and "will eventually publish" become the same guarantee.

---

### 9. [IAM] Documented custom-claim support usable from Data Gateway RLS

**What's missing:** `ConditionSource.AUTH` in a row-level-security rule can reference "UserId, Email, Roles, Permissions, TenantId, and custom claims" per the Data Gateway's own model, but how a custom claim (e.g., a cashier's assigned store) gets onto the token isn't documented or confirmed against a live project.

**Why it matters:** the plan's location-scoped permissions ("this cashier can only act on this store's inventory") are clean to express as an RLS rule *if* a location claim exists on the token. Without it, the fallback is one IAM role per store, which doesn't scale past a few dozen locations.

---

### 10. [Notification / Notifier] Clarify and expose the workflow-engine trigger path

**What's missing:** the Data Gateway already publishes every insert/update/delete as a `DataChangeEvent` to a queue explicitly documented as feeding a workflow engine (`blocks-utilities-net`) — but that engine isn't reachable from the `blocks` CLI or the `@seliseblocks/client` SDK in this workspace, so its availability and authoring model for a given project can't be confirmed or designed against.

**Why it matters:** if it's available and project-authorable, a meaningful share of this project's "run this in response to a data change" needs (low-stock alerts, order-confirmed notifications, replenishment triggers) move there directly — no client-side polling, no manual trigger, no new capability needed from item 1 at all for these specific cases.

---

## P2 — quality-of-life, mainly reporting

### 11. [Data Gateway] Aggregation primitives on the query surface

**What's missing:** the generated GraphQL surface is exactly `getXs`/`insertX`/`updateX`/`deleteX` (plus `Many` variants) — no `$group`, `$sum`, `$avg`, or joins.

**Why it matters:** inventory valuation, sales metrics, stock-ageing, and supplier-performance dashboards (plan §7) need sums and grouping over potentially large collections. Even count-with-filter works today (`totalCount` on a filtered query) for tiles like "out-of-stock count" — but a valuation total across every SKU does not.

**Current mitigation:** precomputed summary schemas populated by whichever client last had reason to compute one, plus client-side aggregation over bounded, paged results for anything not read on every dashboard load.

---

### 12. [Data Gateway] Relevance-ranked / full-text search

**What's missing:** filtering is `eq`/`contains`/`startsWith`/etc. — no ranked full-text search, no facet-count aggregation.

**Why it matters:** the storefront's product discovery (plan §5) wants faceted filters with live counts and reasonable search relevance; today that's approximated with `contains` filters and client-computed counts over a page, which doesn't scale to a large catalog.

---

### 13. [Release] A first-class, still-fully-managed scheduled deployment type

**What's missing:** `blocks release deploy` triggers a configured pipeline for a linked repo; there's no notion of a periodic/cron-triggered deployment.

**Why it matters:** if item 1 (managed triggers/scheduling) isn't feasible as a standalone capability, exposing a *scheduled* deploy target through Release — still entirely Blocks-managed, never something the customer operates — would give reconciliation, replenishment-suggestion, and summary-precomputation logic a home without this project (or any project) standing up its own infrastructure to get it.

---

### 14. [Data Gateway] Sparse / partial unique indexes

`POST /schemas/indexes` builds its index with `new CreateIndexOptions { Name, Unique }` and
nothing else (`server/DataGateway.DomainService/Repositories/DbRepository.cs:396`) — no
`Sparse`, no `PartialFilterExpression`, no collation, no TTL.

MongoDB treats a missing field as `null`, so a unique index without `sparse: true` permits
**exactly one** document that omits the field. That makes a unique index unusable on any
optional field. Concretely, in this project: `ProductVariant.Barcode` is declared unique and is
genuinely optional — a unique index on it would allow one barcode-less variant in the entire
catalog and reject the second. The same trap waits on every `IdempotencyKey` field until every
existing row is backfilled.

**Ask:** expose `IsSparse` (and ideally `PartialFilterExpression`) on `CreateSchemaIndexRequest`.
`IsSparse` alone is a two-line change and removes the whole class of problem.

Without it, "unique but optional" is simply not expressible, and every unique index needs a
manual backfill pass first — with no way to find out except a 409 on creation.

### 15. [Data Gateway] Indexable embedded and array sub-fields

`IsFieldIndexable` requires `!field.IsReferenceField`
(`server/DataGateway.DomainService/Services/Implementations/SchemaIndexService.cs:123-127`), and
every dotted sub-field in a schema export carries `IsReferenceField: true`. So no field inside
a composite type can ever be indexed: not `Order.Items.VariantId`, not `Cart.Items.Sku`, not
`WarehouseInventory.Quantity.OnHand`, not `Warehouse.Address.City`.

MongoDB indexes embedded and array fields natively (multikey indexes) — this is a gateway-side
restriction, not a database one.

The consequence is architectural rather than cosmetic: it means **anything you need to query or
constrain by must be lifted to a top-level scalar**, even when that duplicates a value already
present inside an embedded array, and nothing then keeps the two copies in agreement. A design
that models line items as an embedded array — the natural shape, and the one this project's
`InventoryReservation`, `Cart` and `Order` all use — has no indexable path to "which
reservations hold this variant".

**Ask:** allow dotted field paths whose leaf type is scalar, or stop setting
`IsReferenceField: true` on sub-fields of composite types (it appears to mean "is a sub-field",
not "is a reference", which is a separate naming problem worth untangling).

---

## Summary table

| # | Priority | Service | One-line ask |
|---|---|---|---|
| 1 | P0 | New capability | Managed inbound trigger / scheduled execution |
| 2 | P0 | Data Gateway | Atomic `$inc` on update mutations |
| 3 | P0 | Data Gateway | Real unique-index enforcement behind `IsUniqueData` |
| 4 | P1 | Data Gateway | Index declaration in schema file / `blocks data sync` |
| 5 | P1 | Data Gateway | Transactional multi-collection batch mutation |
| 6 | P1 | Data Gateway | Return updated document from `updateX` |
| 7 | P1 | Data Gateway | TTL index support |
| 8 | P1 | Data Gateway | Reliable/outboxed `DataChangeEvent` delivery |
| 9 | P1 | IAM | Documented custom claims usable in RLS |
| 10 | P1 | Notification / Notifier | Clarify + expose workflow-engine trigger path |
| 11 | P2 | Data Gateway | Aggregation primitives (`$group`/`$sum`/`$avg`) |
| 12 | P2 | Data Gateway | Relevance-ranked full-text search + facet counts |
| 13 | P2 | Release | Fully-managed scheduled deployment type |
| 14 | P1 | Data Gateway | Sparse / partial unique indexes (`IsSparse` on index create) |
| 15 | P1 | Data Gateway | Indexable embedded + array sub-fields (dotted scalar paths) |
