# Building the E-Commerce / POS / Inventory Platform on SELISE Blocks

**Companion to** `ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md`.
**Scope:** how to deliver that plan using Blocks platform services instead of building the Catalog, Commerce and Inventory services described in it.
**Status:** analysis and architecture. No code changed. Nothing in `blocks-data/` was modified — proposed platform changes are collected in `BLOCKS_FEATURE_SUGGESTIONS.md`, each tagged with the Blocks service it belongs to.

**Hard constraint (binding on this whole document):** the platform is built entirely on SELISE Blocks services. **No new backend service, worker, daemon, or scheduled process may be developed or operated for this project.** Where that constraint means a plan requirement isn't fully achievable today, this document says so explicitly and routes the gap into `BLOCKS_FEATURE_SUGGESTIONS.md` as a platform ask instead of quietly reintroducing a service to work around it. See root `AGENTS.md` for the persisted version of this constraint.

---

## 1. What this document is

The build plan specifies three custom backend services (Catalog, Commerce, Inventory) with their own persistence, transactions, event publishing and provider abstraction layer. The instruction here is the opposite: **do not build those services — achieve the same outcomes with Blocks.**

That is mostly possible, and the parts that are not possible are specific and nameable. This document:

- inventories what Blocks actually provides (§2, verified against the CLI, the SDK typings and the `blocks-data` source in this workspace);
- restates the plan's three-service architecture in Blocks terms (§3);
- maps every plan section to a Blocks mechanism with a verdict (§4);
- solves the five genuinely hard problems — atomic reservation, immutable ledger, idempotency, events, reporting (§5);
- states plainly what Blocks cannot do and what the options are (§6);
- gives the target architecture (§7) and a revised roadmap (§8);
- lists the `blocks-data` changes that would remove the sharpest constraints (§9);
- ends with what still needs verifying against the live environment (§10).

---

## 2. What Blocks actually provides

Verified from `blocks --help`, `@seliseblocks/client` typings, the `blocks-skills/` pack, and the `blocks-data` source.

| Service | Surface | What it gives you |
|---|---|---|
| **IAM** | CLI + SDK | Users, roles, permissions (`resource::action`), role hierarchy by `slug`, organizations (multi-workspace), OIDC/SSO hosted login, MFA, client credentials (machine-to-machine), signup settings |
| **Data Gateway** | CLI (schemas/rules/reload) + SDK (runtime GraphQL) | Schema-driven MongoDB collections, per-tenant GraphQL, row-level security (RLS), column-level security (CLS), per-operation access levels, field validation rules, compound + unique indexes |
| **Storage / DMS** | SDK + CLI config | File storage over Azure Blob / S3 / SFTP, pre-signed upload & download URLs, versioning, nested folders, per-resource access policies, trash/restore, audit trail |
| **Mail** | SDK (`send`) + CLI (config/templates/mailbox) | Transactional email, server-side templates selected by `purpose` + `language` |
| **Notifier** | SDK + CLI | Real-time/offline push to users, roles or subscription filters; per-user inbox with read state |
| **Notification** | CLI only | Which channel a notification type uses |
| **Secrets** | CLI | Key/value secret vault |
| **Localization** | CLI + SDK | Per-module, per-culture translation dictionaries |
| **Release** | CLI only | Triggers the configured pipeline for a repo linked to the project; deploys containers |

### The three primitives everything else is built on

Three Data Gateway behaviours, confirmed in source, carry most of the weight in this design:

1. **Guarded conditional update.** `updateX(where: …, input: …)` builds a Mongo filter from the typed `where` (which supports `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `contains`, …) and issues a single `UpdateOne`, returning `totalImpactedData` = `ModifiedCount`.
   → `blocks-data/server/DataGateway.DomainService/Repositories/MongoCollectionOperations.cs:49`
   This is a **compare-and-swap**. It is the reason reservations can be made safe without a backend service.
   The gateway always injects `LastUpdatedDate = UtcNow` on update (`Helpers/DefaultValueInjection.cs:133`), so `ModifiedCount` is `1` whenever the filter matched — the usual "matched but identical, so ModifiedCount is 0" trap does not apply here.

2. **Default-deny per operation.** Access level is set independently for Read / Write / Edit / Delete. Setting an operation to `Custom` (3) with no allow policy denies it outright — the evaluator returns "No policy grants access to this resource."
   → `Helpers/DataAccessPolicyHelper.cs:87`
   This is how a collection becomes **append-only**, which is what the plan's immutable ledger requires.

3. **Real unique and compound indexes.** `SchemaIndexController` (`POST /schemas/indexes`) creates genuine MongoDB indexes, unique or not, single-field or compound, up to **15 per schema**.
   → `Services/Implementations/SchemaIndexService.cs:18`
   A unique index on an idempotency key is the only true duplicate-suppression mechanism available.

### What the Data Gateway does *not* expose

- **No aggregation.** The generated GraphQL surface is exactly `getXs`, `insertX`, `updateX`, `deleteX`, `insertManyX`, `updateManyX`, `deleteManyX` (`Resolvers/SchemaResolver.cs`). No `$group`, no `$sum`, no joins, no computed fields.
- **No `$inc`.** Updates are `$set` only. Numeric changes are read → compute → guarded write.
- **No multi-document transaction.** Each mutation is independent.
- **No TTL indexes**, no scheduled jobs, no cron.
- **No server-side hook you can author.** `DataChangeEvent` is published to the queue `blocks_logic_workflow_data_trigger_listener` for a platform workflow engine (`blocks-utilities-net`) — see §5.4 and §10.

---

## 3. Restating the plan's architecture in Blocks terms

The plan's central structure — three services with separate databases, talking over APIs — **does not survive the translation, and should not be forced to.** Blocks gives one Data Gateway per project, one GraphQL endpoint, one MongoDB. There is no place to run Catalog Service.

What survives, and what must survive, is the *boundary* — the plan's real requirement is that Commerce cannot write Inventory's records and Catalog cannot hold authoritative stock. On Blocks that boundary moves from **network isolation** to **policy isolation**:

| Plan concept | Blocks realisation |
|---|---|
| Catalog / Commerce / Inventory **services** | Three **schema domains** in one Data Gateway project, with a naming convention and separate permission resource groups |
| "Service A must not write service B's database" | **RLS + per-operation access levels + IAM permissions.** A storefront session literally cannot write `InventoryBalances`; only the reservation permission held by the checkout path can, and only through the guarded update in §5.1 |
| API Gateway / BFF | Nothing. Clients call the Data Gateway directly. A BFF exists only if you deploy one (§6.3) |
| Cross-service API calls | Client-side orchestration in the three apps, sharing one `@seliseblocks/client` instance |
| Domain events | `DataChangeEvent` → platform workflow engine (verify), or client-driven Notifier/Mail, or the trusted worker (§6.3) |
| `ICatalogRepository` / `IUnitOfWork` / provider adapters | **Drop this layer.** Provider independence was a design goal to keep Blocks/Supabase/Postgres interchangeable; on Blocks the honest abstraction is a thin typed data-access module per domain (`src/lib/catalog/`, `src/lib/inventory/`) over `blocksClient.data`. Keep the *shape* (one module per domain, domain rules in one place) and abandon the adapter framework — it buys nothing here and costs real complexity |
| Multi-tenancy | **Project = tenant** (`x-blocks-key`). **IAM Organization = the plan's OrganizationId**, and `OrganizationId` is already a platform-managed system field on every record |
| Product editions (Starter/Growth/Enterprise) | IAM roles + permissions, gated in UI by `iam.me()` and enforced server-side by RLS |

**Consequence to accept openly:** the plan's §11 (provider-independent design) is not delivered. Domain rules will live in TypeScript in the client apps and in Data Gateway policies, not behind `IInventoryRepository`. Moving to Postgres later would be a rewrite of the data layer. That is the price of not building the three services, and it should be a conscious trade, not a surprise.

---

## 4. Plan-section mapping

Legend: **✅** achievable on Blocks as-is · **⚠️** achievable with a stated compromise · **❌** needs something Blocks does not provide (see §6)

### 4.1 Catalog (plan §4.1)

| Requirement | Blocks mechanism | |
|---|---|---|
| Products, variants, categories, brands | Data schemas — **already exist** (4 of them) | ✅ |
| Attributes, attribute groups | New schemas + existing `Attribute` DTO | ✅ |
| Product media | Storage service: pre-signed upload, file id stored on the product; existing `Media` DTO | ✅ |
| Price lists, channel prices | New `PriceList` / `Price` schemas keyed by channel + currency | ✅ |
| Promotions, coupons | New schemas; **coupon validity must be revalidated at order placement** — client-side evaluation alone is not trustworthy | ⚠️ |
| Reviews, verified-purchase check | New schema; verification requires reading the customer's orders — enforce with an RLS policy tying `CreatedBy` to the review author, and approve moderation server-side by permission | ⚠️ |
| Channel publication, SEO | Fields + a `SalesChannelPublication` schema | ✅ |
| Bulk import/export | `insertManyProduct` from the backoffice, batched client-side | ⚠️ |
| Multi-language catalog | Every record carries a platform `Language` field; Blocks Localization covers UI strings | ✅ |

### 4.2 Commerce (plan §4.2)

Nothing in this domain exists yet. The consumer app's cart is `localStorage` (`ecommerce-consumer/src/components/providers/cart-provider.tsx:41`) and checkout is simulated (`pages/CheckoutPage.tsx:136`).

| Requirement | Blocks mechanism | |
|---|---|---|
| Customer profile, addresses, favourites | New schemas; identity stays in IAM, commerce data keyed by IAM user id | ✅ |
| Cart, cart merge on login | Server-side `Cart` schema with RLS `CreatedBy == AUTH.UserId`; guest cart in `localStorage`, merged on login | ✅ |
| Orders, order lines with price/tax/discount snapshots | New schemas; snapshots are just fields | ✅ |
| Order / payment / fulfillment / return status as separate fields | Fields + validation rules | ✅ |
| Idempotent order submission | Unique index on `IdempotencyKey` (§5.3) | ✅ |
| **Payments** | **No Blocks payment service.** Requires an external PSP; card data must never reach your code, and the webhook needs a trusted server | ❌ §6.1 |
| Refunds | Same constraint as payments | ❌ §6.1 |
| Invoices, receipts (PDF) | Generate client-side, store via Storage with access policy scoped to the customer | ⚠️ |
| POS terminals, shifts, cash movements | New schemas; shift reconciliation is arithmetic over an append-only `POSCashMovements` collection | ✅ |
| Returns, exchanges | New schemas + inventory movements on receipt | ✅ |

### 4.3 Inventory (plan §4.3, §8)

| Requirement | Blocks mechanism | |
|---|---|---|
| Locations, warehouses, stores | `Warehouse` exists; add `InventoryLocation` to cover stores uniformly | ✅ |
| Zones and bins | New schemas; `BinLocation` DTO exists | ✅ |
| Balances, quantity buckets (OnHand/Reserved/…) | `WarehouseInventory` exists with `InventoryQuantity` DTO | ✅ |
| `AvailableToSell` derivation | **Must be a stored field, not computed** — the `where` guard in §5.1 has to filter on it. Recompute it in the same guarded write that changes any input bucket | ⚠️ |
| **Atomic availability check + reservation** | Compare-and-swap, §5.1 | ✅ |
| Reservation expiry / background release | No scheduler in Blocks | ❌ §6.2 |
| **Immutable ledger** | Append-only collection via default-deny on Edit/Delete, §5.2 | ✅ |
| Balance + ledger written as one transaction | **Not possible.** Two writes, ordered, with a reconciliation pass, §5.5 | ⚠️ |
| Allocation strategies (priority, nearest, FEFO…) | Pure logic — runs client-side in backoffice/POS, or in the worker | ✅ |
| Transfers with full lifecycle | New schema; `StockTransfer` exists, extend states | ✅ |
| Suppliers, POs, goods receipt, partial receipt | `Supplier`/`PurchaseOrder` exist; add `GoodsReceipt`, `PurchaseRequisition` | ✅ |
| Picking, packing, fulfillment, packages | New schemas | ✅ |
| Lots, serials, expiry, FIFO/FEFO | New schemas + compound indexes | ✅ |
| Stock counting, variance approval | New schemas; approval is a permission-gated status change | ✅ |
| Costing, cost layers, valuation | `CostLayer` schema; **valuation totals cannot be aggregated by the gateway** (§5.6) | ⚠️ |
| Replenishment rules, reorder suggestions | Rules as data; suggestion generation needs a scheduler for anything proactive | ⚠️ §6.2 |

### 4.4 Clients (plan §5, §6, §7)

| Requirement | Blocks mechanism | |
|---|---|---|
| Storefront browse without login | Product reads are already configured Public on the Data Gateway | ✅ |
| Faceted search, sorting, pagination | `where` + `order` + paging on `getProducts`; **facet counts need aggregation** — precompute or count client-side over a capped result | ⚠️ |
| Search suggestions / relevance ranking | No search provider in Blocks; regex/`contains` filters only | ⚠️ |
| POS barcode scan, fast SKU lookup | Compound index on `(Barcode)` / `(Sku)`; sub-200ms is realistic | ✅ |
| POS offline: local cache, queue, sync | Client-side (IndexedDB) + stable terminal/transaction ids + idempotency keys on replay (§5.3) | ✅ |
| POS offline stock-selling policy | Configurable; conflicts resolved on replay because the guarded write will simply fail when stock is gone | ✅ |
| Manager approval for voids/discounts | IAM permission + re-authentication; audit row in an append-only collection | ✅ |
| Backoffice CRUD for all entities | Schema-driven resource UI already exists in `ecommerce-back-office/src/components/resource/` | ✅ |
| Dashboards and reports | No aggregation (§5.6) | ⚠️ |

### 4.5 Cross-cutting (plan §10, §12, §13, §14)

| Requirement | Blocks mechanism | |
|---|---|---|
| Tenant isolation on every record/query/event | Project-scoped by `x-blocks-key`; `OrganizationId` system field; RLS by `AUTH.TenantId` | ✅ |
| Role model (9 backoffice roles) | IAM roles + permissions, `blocks iam roles create` / `assign-permissions` | ✅ |
| Location-scoped permissions ("this cashier, this store") | RLS comparing a token claim to a record field — **depends on IAM supporting a custom claim for location** | ⚠️ §10 |
| Sensitive-field permissions (hide unit cost from cashiers) | **CLS** — per-field access policies, already in the schema format | ✅ |
| Entitlements per edition | Permissions + `iam.resources.features()` for UI gating; RLS for enforcement | ✅ |
| Audit log of sensitive operations | Append-only collection, same pattern as the ledger | ✅ |
| MFA for privileged users | Blocks MFA | ✅ |
| Encryption, secret management, PII flags | Platform-managed; `IsPIIData` per field; Blocks Secrets | ✅ |
| Rate limiting | Not exposed | ❌ platform-side |
| Structured logs, traces, slow-query monitoring | Platform-side; the gateway records `GatewayOperationActivity` per call. Your own clients get whatever you instrument | ⚠️ |
| Transactional outbox, DLQ, consumer inbox | Not available (§5.4) | ❌ |
| Deployment models (shared SaaS → on-prem) | Project-per-tenant covers shared SaaS and dedicated DB; on-prem means self-hosting the Blocks stack | ⚠️ |

---

## 5. The five hard problems, solved

### 5.1 Preventing oversell without a backend — guarded compare-and-swap

This is the plan's single most important invariant (§18: "Inventory reservations prevent overselling"). It is achievable.

**Balance shape** (`WarehouseInventory`): store `OnHand`, `Reserved`, `Damaged`, `QualityHold`, `Blocked`, **`AvailableToSell`** and **`Version`** as fields. `AvailableToSell` is *stored*, not derived at read time, because the guard filters on it.

**Reserve `qty` of a variant at a location:**

```graphql
mutation reserve($where: WarehouseInventoryFilterInput, $input: WarehouseInventoryUpdateInput!) {
  updateWarehouseInventory(where: $where, input: $input) {
    acknowledged
    totalImpactedData
  }
}
```

```jsonc
// variables — read the balance first to compute the new values
{
  "where": {
    "ItemId":          { "eq":  "<balanceId>" },
    "Version":         { "eq":  7 },           // optimistic concurrency
    "AvailableToSell": { "gte": 3 }            // the actual oversell guard
  },
  "input": {
    "Reserved":        12,                     // read value + 3
    "AvailableToSell": 5,                      // read value - 3
    "Version":         8
  }
}
```

- `totalImpactedData === 1` → the reservation is yours. Nothing else could have taken that stock, because the filter and the write are one `UpdateOne`.
- `totalImpactedData === 0` → either someone else moved the version, or availability dropped below 3. Re-read and retry (bounded, 3–5 attempts with jitter); after that, report unavailable.

**Why both guards.** `Version` alone would fail spuriously on unrelated concurrent edits; `AvailableToSell >= qty` alone would allow a lost update on the other buckets. Together they give correctness and a low retry rate.

**Multi-line / multi-location reservations are not atomic across documents.** A cart with four lines is four CAS operations. Compensate by releasing the successful ones if a later line fails — the release is itself a guarded write, and a release can always be retried safely because it only ever *increases* availability.

The same pattern covers POS deduction, goods receipt, transfer dispatch/receipt and adjustments. Every stock-changing operation in the plan reduces to: read balance → compute → guarded write → verify `totalImpactedData`.

### 5.2 The immutable ledger

`InventoryMovements` must never be edited or deleted (plan §8.3). Configure the schema as:

| Operation | Access level | Policy |
|---|---|---|
| Write (insert) | `Custom` | Allow when `AUTH.Permissions` contains `inventory::movement::write` |
| Read | `Custom` | Allow by role (warehouse/finance/auditor); CLS hides `UnitCost` from cashier roles |
| **Edit** | `Custom` | **No allow policy → denied for everyone, including admins** |
| **Delete** | `Custom` | **No allow policy → denied for everyone** |

Verified against `DataAccessPolicyHelper.cs:87` — with `Custom` and zero applicable allow policies, evaluation returns access denied. Corrections happen the way the plan wants: a compensating reversing movement, never an edit.

Apply the identical pattern to `POSCashMovements`, `OrderActivity` and the audit log.

### 5.3 Idempotency

Two mechanisms, and only one of them is trustworthy:

- **`IsUniqueData` on a field is a pre-check, not a constraint.** The gateway queries for a conflict and then inserts (`MutationService.cs:309`). Two concurrent identical requests can both pass the check. Do not rely on it for money or stock.
- **A real unique index is a constraint.** `POST /schemas/indexes` with `isUnique: true` creates a MongoDB unique index; the duplicate insert fails at the database.

So: every stock-changing and money-moving collection gets an `IdempotencyKey` field with a **unique index** — `InventoryMovements`, `Orders`, `POSTransactions`, `Payments`, `Refunds`, `GoodsReceipts`. The key is generated by the client at the start of the operation and reused on every retry, including POS offline replay after days of disconnection.

Caveat: this index API is **not in the CLI and not in the SDK's typed surface** (see §9.4). Today it is called through `blocksClient.http.request` or the portal, which means index creation is not part of `blocks data sync` and must be scripted separately and checked into the repo.

### 5.4 Events

The plan lists ~35 domain events and demands a transactional outbox, consumer inbox and DLQ. Here is the honest position:

- **The gateway does publish a `DataChangeEvent`** on every insert/update/delete, to the queue `blocks_logic_workflow_data_trigger_listener`, consumed by a platform workflow engine (`blocks-utilities-net`). If that engine is available and authorable on your project, a meaningful share of the plan's event-driven work (low-stock alerts, order notifications, replenishment suggestions) can live there. **This is visible in the data service's source but is exposed through neither the CLI nor the SDK in this workspace — confirm availability in the portal before designing around it (§10).**
- **It is not an outbox.** The publish is fire-and-forget and swallows its own exceptions (`Services/DataChangeEventPublisher.cs:74`). An event can be lost after a successful commit. The plan's "no lost events after a successful database commit" is not met.
- **What to do instead:** treat events as *best-effort enrichment*, never as the mechanism that maintains correctness. Anything that must be true regardless of event delivery — balances matching the ledger, reservations released after payment failure — needs a reconciliation pass (§5.5), not an event handler.
- **User-facing signalling** (order confirmed, shipment dispatched, low-stock warning to a manager) goes through **Blocks Notifier** and **Blocks Mail**, triggered by the client that performed the action.

### 5.5 Consistency without transactions

Balance and ledger cannot be written atomically. The mitigation is ordering plus reconciliation:

1. **Write the guarded balance change first.** It is the operation that can fail for business reasons, and it is the one that must never be duplicated.
2. **Write the ledger movement second**, carrying `BalanceBefore`, `BalanceAfter`, the idempotency key and a correlation id.
3. **If step 2 fails,** retry it — it is an insert with a unique key, so retrying is safe and the compensating alternative (reversing step 1) is strictly worse.
4. **Reconcile.** A periodic pass recomputes each balance by replaying its movements and reports drift. This is the safety net for a step-2 loss, and it needs somewhere to run (§6.2).

State this as an accepted, monitored risk: the window between the two writes is the one place where the system can be briefly inconsistent, and reconciliation is what closes it.

### 5.6 Reporting without aggregation

The Data Gateway returns items and a total count. It cannot sum, group or join. Plan §7's dashboards — inventory valuation, stock ageing, supplier performance, sales metrics — therefore need one of:

- **Precomputed summary schemas** (`DailySalesSummary`, `InventoryValuationSnapshot`), written by whatever runs the scheduled work (§6.2). This is the right answer for anything a dashboard loads on open.
- **Client-side aggregation over a bounded page** — acceptable for "top 20 low-stock items", not for valuation across 50,000 SKUs.
- **`totalCount` with a filter** — genuinely useful and cheap: out-of-stock count, open POs, pending transfers all reduce to a filtered count with `pageSize: 1`.

Design every dashboard tile as one of those three from the start. Retrofitting is expensive.

---

## 6. What Blocks cannot do

Three real gaps. Each needs a decision.

### 6.1 Payments

There is no payment service in Blocks, and the plan correctly forbids storing card data. Card entry must happen in a PSP-hosted page or PSP-hosted fields — never in your form. That part is fine from a pure client app.

The problem is **confirmation**. A PSP tells you a payment succeeded by calling a webhook on a server you control. A browser cannot receive one, and trusting the browser's word that payment succeeded is exactly the fraud the webhook exists to prevent.

**Options:**

| Option | Assessment |
|---|---|
| Client marks the order paid on PSP redirect | **Not acceptable** for real money. Trivially forged |
| Poll the PSP from the browser with a publishable key | Weak — depends on the PSP exposing a safe read, still client-trusted |
| A backend receives the webhook and writes payment state | The only fully sound answer — **and out of scope**: this project builds no backend, so this option is not taken |
| PSP with a no-code integration that writes back | Possible with some PSPs; still needs a receiving endpoint, which is the same gap |
| **Manual staff-side confirmation (chosen for now)** | A finance/customer-support role gets a permission-gated "Confirm payment" action on an order in `PendingPayment`, used after checking the PSP's own dashboard. No backend, no webhook. Slower and human-dependent, but correct and fully within Blocks | ⚠️ interim |

For **POS cash and card-present sales** none of this applies — cash has no webhook, and a card-present terminal confirms directly with the card network over its own secure channel, not through your infrastructure. Both are legitimate day-one scope that works fully on Blocks.

**This is the top platform ask in `BLOCKS_FEATURE_SUGGESTIONS.md`:** a managed way for an external system to push a trusted, authenticated update into Blocks (an inbound webhook receiver, or scheduled/triggered execution Blocks itself runs) would close this gap without anyone operating a backend. Until that exists, online orders use manual confirmation; card payment does not block launching POS + backoffice + browse/cart-only storefront.

### 6.2 Scheduled and background work

Blocks has no cron, no TTL indexes and no job runner. The plan needs, at minimum:

- reservation expiry and release (plan §8.4 — without it, abandoned carts hold stock forever);
- balance-vs-ledger reconciliation (§5.5);
- replenishment suggestions and low-stock sweeps (§8.11);
- dashboard summary precomputation (§5.6);
- payment/event reconciliation jobs (§12).

Reservation expiry is the one that will hurt in week one. **No backend may be built to run a scheduler**, so the mitigation has to be entirely client-triggered: every client that reads availability treats a reservation with `ExpiresAt < now` as **logically released**, and opportunistically issues the release write when it encounters one (on cart open, on checkout start, on the backoffice reservations screen loading). This reduces the damage — busy items self-heal quickly because something is always looking — but does not fully fix it: a reservation nobody's client ever looks at again stays visibly held. That residual gap is accepted, not solved, and is recorded as a platform ask below.

### 6.3 No custom backend — what that rules out, and what to do instead

The plan's transactional outbox, reconciliation jobs, and payment webhook receiver all assume *some* trusted process outside the browser. This project builds none: no worker, no cron daemon, no custom API server, nothing deployed and operated by this team beyond the three (or four, with POS) client apps. That is a deliberate scope decision, not an oversight, and it applies everywhere in this document that earlier drafts reached for "run this in a small service."

Concretely, with that constraint:

- **Payment confirmation** is manual (§6.1) until Blocks offers a trusted way to receive an external confirmation.
- **Reservation expiry** is lazy/client-triggered (above), not scheduled.
- **Balance/ledger reconciliation** (§5.5) runs as an on-demand backoffice report a human triggers, not a background job.
- **Replenishment suggestions and dashboard summaries** (§5.6, §8.11) are computed client-side, on demand, over bounded pages — not precomputed by a batch process.
- **Low-stock alerts** fire when a client's guarded write crosses the threshold in the same request (the client that just sold the last-but-one unit sends the notifier call itself), not from a monitoring loop.

**The platform ask.** Every one of the items above would be better served by a Blocks-managed capability for trusted external triggers and scheduled execution — see `BLOCKS_FEATURE_SUGGESTIONS.md` (P0, item 1). That is a feature request *to the Blocks platform*, not a service *this project* stands up: if Blocks adds managed scheduled/triggered execution, this project consumes it like any other Blocks service, and every mitigation above upgrades from best-effort to reliable with no architecture change. Until then, the mitigations above are the accepted, documented shape of the product.

**What this means for scope:** the platform ships as POS + backoffice with cash and card-present payment, plus a storefront that browses, carts, and checks out to `PendingPayment` with manual confirmation. That is a coherent, sellable product today. It does not fully meet the plan's §18 acceptance criteria ("no lost events after a successful commit", reservation expiry, automated reconciliation) — those depend on the platform ask above landing.

---

## 7. Target architecture

```text
┌─ ecommerce-consumer ─┐  ┌─ pos-app (new) ─┐  ┌─ ecommerce-back-office ─┐
│  storefront          │  │  offline-first  │  │  staff console          │
└──────────┬───────────┘  └────────┬────────┘  └───────────┬─────────────┘
           │                       │                       │
           └───────────────────────┼───────────────────────┘
                                   │  @seliseblocks/client
                                   ▼
        ┌──────────────────────────────────────────────────────┐
        │                    SELISE Blocks                      │
        │  IAM — users, roles, permissions, orgs, OIDC, MFA     │
        │  Data Gateway — Catalog | Commerce | Inventory        │
        │                 schemas, RLS/CLS, indexes             │
        │  Storage — media, invoices, receipts                  │
        │  Mail · Notifier · Localization · Secrets             │
        └──────────────────────────────────────────────────────┘
```

No fifth box. Every job that isn't one of the three clients is either done by whichever client happens to trigger it (lazy reservation release, threshold-crossing low-stock alerts) or done by a human through a backoffice screen (payment confirmation, reconciliation). See §6.3.

**Domain boundaries are policy, not network.** Enforce them with:

- a permission namespace per domain — `catalog::*`, `commerce::*`, `inventory::*`;
- per-operation access levels on every schema, defaulting to deny;
- one client-side data module per domain (`src/lib/catalog/`, `src/lib/commerce/`, `src/lib/inventory/`), shared across the three apps — **all reservation logic lives in exactly one place**, or it will drift between storefront and POS and the invariant will be lost.

---

## 8. Revised roadmap

The plan's four phases hold. What changes is the artifacts each produces.

### Phase 0 — Foundation (new, and not optional)

Before any feature work:

1. **Schema inventory.** 11 entity schemas exist; the plan implies ~61. Author the remaining ~50 in `blocks/data/schemas/`, push and reload via `blocks data sync`.
2. **Access-level and policy baseline.** Every schema gets explicit Read/Write/Edit/Delete levels. Append-only collections get default-deny on Edit/Delete (§5.2).
3. **Index plan.** Unique indexes on every `IdempotencyKey`; compound indexes for `(VariantId, LocationId)`, `(Barcode)`, `(OrderId, Status)`. Watch the 15-per-schema ceiling. Script this (§5.3) and check the script in.
4. **Role and permission model.** The plan's nine backoffice roles plus cashier, store manager, customer. `blocks iam roles create` + `assign-permissions`, `--dry-run` first.
5. **The shared inventory module.** The CAS reserve/commit/release helpers from §5.1, with retry and idempotency built in, written once and consumed by all three apps.

### Phase 1 — Catalog + basic sale

Catalog schemas complete · storefront discovery on real data · Commerce schemas (cart, order) · cart moved off `localStorage` · POS cash sale end to end · inventory balances and ledger · reservations working with the CAS guard.

### Phase 2 — Core inventory

Allocation strategies · adjustments · multi-location · transfers · picking and packing · low-stock detection · inventory dashboards on precomputed summaries.

### Phase 3 — Procurement, fulfillment, payments

Suppliers · POs · goods receipt with partial receiving · shipments · returns and inspection · PSP hosted checkout with **manual staff-side payment confirmation** (§6.1) · lazy reservation-expiry checks wired into every client (§6.3) · on-demand reconciliation report. No backend is introduced at this or any later phase.

### Phase 4 — Enterprise

Bins and zones · lots, serials, expiry, FIFO/FEFO · cycle counting · costing and valuation · replenishment · approval workflows · offline POS sync · entitlement tiers.

---

## 9. Proposed platform changes

Per the instruction, nothing in `blocks-data/` was modified. The full, prioritized list of Blocks platform changes this design would benefit from — each tagged with the exact Blocks service it belongs to — now lives in **`BLOCKS_FEATURE_SUGGESTIONS.md`**, not here, so there is one canonical place to track them instead of duplicating the list across documents.

The single highest-leverage item there: **[Data Gateway] atomic `$inc`** — every inventory quantity change today needs read → compute → guarded write → retry (§5.1) because updates are `$set`-only (`MongoCollectionOperations.UpdateOneAsync`, `blocks-data/server/DataGateway.DomainService/Repositories/MongoCollectionOperations.cs:54`). The second: **[new capability] a managed way for Blocks to receive a trusted external signal or run something on a schedule** — see §6.3; it's what would let payment confirmation and reservation expiry stop being manual/lazy without this project operating a backend.

---

## 10. To verify before committing to this design

These affect the architecture and could not be settled from the workspace alone:

1. **Is the workflow engine (`blocks-utilities-net`) available and authorable on the target project?** It changes the size of the trusted worker, and possibly removes the need for it outside payments. Check the portal's automation/workflow section.
2. **Can IAM issue a custom claim** (e.g. assigned store/location) usable in an RLS rule? `ConditionSource.AUTH` documents "UserId, Email, Roles, Permissions, TenantId, and custom claims" — confirm how a custom claim is populated. Without it, store-scoped data access falls back to role-per-store, which does not scale past a few dozen stores.
3. **Is there a schema-count limit per project?** The design needs ~61 entity schemas plus DTOs.
4. **Realistic Data Gateway latency** for a filtered `getXs` with a compound index, against the plan's targets (availability p95 < 200ms, POS checkout p95 < 1s).
5. **Whether the Blocks platform team has (or plans) a managed capability for trusted external triggers or scheduled execution** — the top item in `BLOCKS_FEATURE_SUGGESTIONS.md`. Its absence is why payment confirmation and reservation expiry are manual/lazy in this design.
6. **PSP choice**, since it determines the hosted-checkout contract and what a manual confirmation screen needs to show staff (which fields from the PSP dashboard to cross-check) for both storefront and POS card-present flows.

---

## 11. Summary

- The plan's **outcomes** are largely achievable on Blocks. The plan's **architecture** — three services, provider abstraction, transactional outbox — is not, and forcing it would produce a worse system than adapting it.
- **Oversell prevention, the immutable ledger, idempotency, tenant isolation, RBAC down to the field level, and the full POS cash flow all work on Blocks today**, using the compare-and-swap, default-deny and unique-index primitives in §5.
- **Three things genuinely do not fit, and no backend is built to paper over them:** card payments, scheduled work, and cross-document transactions. Payment confirmation and reservation expiry get client-driven, manual-first mitigations (§6.3) instead of a worker; cross-document consistency is mitigated by ordering and on-demand reconciliation (§5.5). All three would be properly fixed by platform changes, not project infrastructure — see `BLOCKS_FEATURE_SUGGESTIONS.md`.
- **Biggest risk if nothing changes:** reservation expiry. With only lazy, client-triggered release and no scheduler, a reservation nobody's client revisits stays held indefinitely — visible to customers within days of launch.
- **Highest-leverage platform change:** atomic `$inc` on the Data Gateway, followed by a managed capability for trusted external triggers/scheduled execution. Both are P0 in `BLOCKS_FEATURE_SUGGESTIONS.md`.
