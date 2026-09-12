# E-Commerce Platform — Task Breakdown & Implementation Cross-Check

**Sources this is built from:** `ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` (original plan),
`ECOMMERCE_PLATFORM_ON_BLOCKS.md` (Blocks-native architecture and phased roadmap this
breakdown follows), the live `ECOMMERCE_INVENTORY_SCHEMAS.json` schema/policy export, and a
full source read of both app repos (`ecommerce-back-office`, `ecommerce-consumer`) done for
this document. Nothing was assumed — every DONE/PARTIAL/NOT STARTED call below has a
file/line or a schema-field citation next to it.

**Scope note:** by user direction, **POS is out of scope for this breakdown.** Everything
below covers only the storefront (`ecommerce-consumer`), the backoffice
(`ecommerce-back-office`), and the Data Gateway/Storage work behind both. The plan's
POS-specific requirements (§6, POS terminals/shifts/cash movements) are not tracked here.

**How to read status tags:** **[DONE]** — implemented and wired to real Data Gateway calls,
no mocking. **[PARTIAL]** — real, but narrower than the plan, or working with a gap noted
inline. **[NOT STARTED]** — no code exists for it. A task list under each phase carries its
own checkbox; check it off as it's completed.

---

## 0. Snapshot

| Domain | Data model (Data Gateway) | Backoffice | Storefront |
|---|---|---|---|
| **Catalog** | Mostly solid: 4/4 core entities exist, but price lists, promotions/coupons, reviews, bundles, channel publication are all missing schemas | Strong — 10/11 entities have full CRUD | Partial — browse/PDP work, no search relevance/SEO/reviews/pagination |
| **Commerce** | **Zero schemas** — no Cart, Order, CommerceCustomer, Payment, Invoice, anything | **Zero code** — confirmed by repo-wide search | **Zero real code** — cart/wishlist/checkout are localStorage-only, order placement is a `setTimeout` |
| **Inventory** | 7/~25 needed entities exist; the 2 most important ones currently can't be written to (see §1) | Strong CRUD, but no reservation lifecycle, no immutability enforcement, no bins/lots/counting/receiving/picking | **Zero** — no availability check anywhere in the storefront |
| **Platform (IAM/Storage)** | — | IAM auth wired; **zero** permission-gating; **zero** Storage/file-upload usage | IAM auth wired; **zero** Storage usage |

**Bottom line:** Catalog + basic inventory administration is genuinely solid. Everything
Commerce-shaped (cart, checkout, orders, payments) is either entirely absent or entirely
simulated. This matches the plan's own difficulty curve (Commerce Service has the least Blocks
precedent) but means Phase 1 has more ground to cover than "basic sale" suggests.

---

## 1. P0 — Fix before any new feature work

These are bugs in the *current* project configuration, not gaps in scope. Two of them mean
screens that already exist in the backoffice almost certainly fail right now.

**The exact fix for every item below is drafted and ready to apply — see `P0_POLICY_FIXES.json`
+ `P0_POLICY_FIXES.md`** (before/after access levels, exact policy objects to add/remove, and
the CLI steps to apply them). Not applied yet — that's yours to run.

### 1.1 `WarehouseInventory` cannot be created or updated via the Data Gateway

Schema export: `WarehouseInventory` has `WriteAccessLevel=Custom(3)` and
`EditAccessLevel=Custom(3)`, but the schema's only policy is
`"only admin can delete data"` (`Operation=DELETE`, allow). Per the Data Gateway's own
default-deny rule (verified against source in `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.2/§9), a
`Custom` access level with **zero** applicable allow policies denies the operation to everyone,
admins included. There is no WRITE or EDIT allow policy on this schema at all.

**Concretely:** `ecommerce-back-office`'s `WarehouseInventory` create/edit forms
(`/admin/warehouse-inventory`, and the scoped tab inside `WarehouseDetailPage.tsx`) exist,
render, and look functional — but every Create/Save action against them should currently fail
with an access-denied error from the Data Gateway. **Test this first** to confirm before
building anything on top of it.

- [ ] Add a WRITE allow policy to `WarehouseInventory` (e.g., permission
      `inventory::balance::write`, or role-scoped to Warehouse manager/Inventory operator).
- [ ] Add an EDIT allow policy the same way — this is what every reserve/commit/release
      compare-and-swap write in `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1 depends on.

### 1.2 `InventoryMovement` cannot be created via the Data Gateway — and can currently be deleted by admins, backwards from the plan's requirement

Same shape as 1.1 for WRITE (`WriteAccessLevel=Custom`, no WRITE allow policy — movements can't
be inserted at all right now) and EDIT (also blocked, which happens to match the plan's
"movements must not be edited" rule, but for the wrong reason — it's an oversight, not a
deliberate immutability design).

The schema's one policy — `"only admin can delete data"`, `Operation=DELETE`, **allow** — does
apply here, meaning **admins can currently hard-delete ledger movements**, directly
contradicting the plan's core invariant ("Movements must not be edited or deleted. Corrections
require reversing movements," `ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` §8.3).

- [ ] Add a WRITE allow policy so movements can be inserted (this is the entire point of the
      schema — nothing about inventory can work without it).
- [ ] **Remove or override the delete-allow policy for this schema specifically** — Delete
      should have no allow policy at all here, unlike most other entities where "admin can
      delete" is reasonable. Document this schema as the one place that boilerplate policy
      must not be reused.
- [ ] Leave EDIT with no allow policy (already correct, just make it intentional rather than
      accidental — add a comment/description on the schema saying so).

### 1.3 Every entity schema's `ReadAccessLevel` is Public — including cost and financial data

All 11 entity schemas currently have `ReadAccessLevel=Public(2)`, not just the catalog-facing
ones. `Supplier` (unit costs, payment terms), `PurchaseOrder` (costs, supplier relationships),
`InventoryMovement`, and `InventoryReservation` are all readable by anyone holding the
tenant's `x-blocks-key` — which is, by design, embedded in the public storefront's JS bundle.
Only `Product`/`Category`/`Brand`/`ProductVariant` were meant to be public per
`ecommerce-back-office/AGENTS.md`'s own stated intent.

- [ ] Change `ReadAccessLevel` to `User` (or `Custom`, role/permission-scoped) for
      `InventoryMovement`, `InventoryReservation`, `StockTransfer`, `Supplier`,
      `PurchaseOrder`, `Warehouse`. Keep `Product`, `Category`, `Brand`, `ProductVariant`
      public.

**`WarehouseInventory` is deliberately excluded from this list, correcting an earlier version
of this section** — it needs to stay `Public` because the storefront's inventory-availability
check (§3 below) reads it unauthenticated, the same reason catalog schemas are public. See
`P0_POLICY_FIXES.md` for the full reasoning and a CLS follow-up (hide `ReorderPoint`/
`ReorderQuantity`/`BinLocation`/`LastCountedDate` from public reads while keeping `Quantity`
visible) that wasn't included here for lack of a verified field-policy JSON shape to copy.

### 1.4 `InventoryReservation` can currently be created by anyone, unauthenticated

`WriteAccessLevel=Public(2)` on this schema — any caller with the tenant key can insert a
reservation record directly against the Data Gateway, bypassing cart/checkout logic entirely.

- [ ] Change to `Custom`, scoped to an authenticated checkout flow or a service permission —
      not open write.

### 1.5 `WarehouseInventory.Version` is typed `Long`, which the Data Gateway does not support

Found while drafting the index plan (sequence step S3). The gateway's scalar list is `String`,
`Int`, `Float`, `Boolean`, `DateTime`, `ID`
(`blocks-data/server/DataGateway.DomainService/Helpers/GraphQlTypeHelper.cs:12`), and its
bulk-import validator names `Long` explicitly as a type it must reject "rather than being
persisted and only failing later when the GraphQL schema is built"
(`Validators/SchemaImportValidator.cs:32`).

This is the optimistic-concurrency guard the entire compare-and-swap design depends on
(`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1). If it isn't a usable scalar, there is no safe
reserve/commit/release — which is the mechanism that prevents overselling.

- [x] **Client half fixed.** Both apps' `collections.ts` emitted `Version { }` for it — an
      empty sub-selection is a GraphQL *parse error* that fails the whole query, so every
      `WarehouseInventory` read through the generic collection layer was broken: the
      backoffice inventory screens and the storefront's `useVariantAvailability` alike. Fields
      whose type has no `COMPLEX_TYPES` entry are now selected bare. `ID` was also added to
      the generator's scalar list, where it always belonged.
- [ ] **Confirm what's actually live** — `blocks data schema pull --json`, check the real type.
      Either the project has an invalid field, or this repo's export is stale. Both are worth
      knowing; only the first needs a push.
- [ ] **Retype to `Int` if needed** — drafted in `SCHEMA_BATCH_2.json`. The gateway maps C#
      `long` to `Int` anyway (`GraphQlTypeHelper.GetScalarType`), so `Int` is the correct
      spelling for this counter.

---

## 2. Phase 0 — Foundation

Per `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §8: schema inventory, access-policy baseline, index
plan, role model, shared inventory module. Status against what's actually there:

- [x] **Catalog schemas exist** — `Brand`, `Category`, `Product`, `ProductVariant`, all pushed
      and reloaded, all with real backoffice CRUD.
- [x] **Core inventory schemas exist** — `Warehouse`, `WarehouseInventory`,
      `InventoryReservation`, `InventoryMovement`, `StockTransfer`, `Supplier`,
      `PurchaseOrder`.
- [ ] **Access-level/policy baseline is not correct yet** — see §1 above; this is the actual
      first task, not a later hardening pass.
- [ ] **No Commerce schema exists at all** — `CommerceCustomers`, `CustomerAddresses`,
      `Favourites`, `Carts`, `CartItems`, `Orders`, `OrderItems`, `Payments`, `Invoices`,
      `Receipts` (10+ schemas) all need to be authored from zero. This is the single largest
      chunk of remaining schema work — bigger than the rest of inventory combined.
- [ ] **Remaining inventory schemas missing:** `WarehouseZones`, `Bins` (only an embedded
      `BinLocation` DTO exists on `WarehouseInventory`, no standalone entity), `Suppliers`
      exists but `GoodsReceipts`, `PurchaseRequisitions`, `Fulfillments`, `PickLists`,
      `Packages`, `ReturnInspections`, `StockCounts`, `Lots`, `SerialNumbers`, `CostLayers`,
      `ReplenishmentRules` — none exist.
- [ ] **Remaining catalog schemas missing:** `PriceLists`/`Prices` as standalone entities (only
      an embedded `Pricing` DTO on `ProductVariant` exists — no multi-channel/price-list
      support), `Promotions`, `Coupons`, `Reviews`, `ProductBundles`, `RelatedProducts`,
      `Collections`, `SalesChannelPublications`, `Attributes`/`AttributeGroups` as
      admin-managed entities (currently just an embedded DTO with free-text values).
- [x] **`WarehouseInventory` already has a `Version` field** and `InventoryMovement` already
      has an `IdempotencyKey` field — both schemas were designed with the compare-and-swap
      pattern (§5.1) in mind, which is good news for Phase 1/2 work, once §1.1 is fixed.
- [ ] **Quantity buckets are incomplete — drafted, not applied.** `WarehouseInventory.Quantity`
      has `OnHand`, `Reserved`, `Damaged`, `QualityHold`, `Incoming`; `Blocked`, `Backordered`
      and `InTransit` from the plan's nine (§8.2) are added in `SCHEMA_BATCH_2.json`. `Blocked`
      is the one that changes behaviour rather than reporting: it belongs in
      `AvailableToSell = OnHand − Reserved − Damaged − QualityHold − Blocked`, so without it
      administratively held stock can still be sold.
- [ ] **Index plan drafted, not applied.** `INDEX_PLAN.json` — 24 indexes in three tiers,
      verified against the platform's own `SchemaIndexService`/`CreateSchemaIndexRequest`
      rather than documentation. Tier 1 is correctness, not speed: besides
      `InventoryMovement.IdempotencyKey`, it includes a **unique
      `WarehouseInventory(WarehouseId, VariantId)`** — nothing enforces that natural key today,
      and a duplicate balance row would make `AvailableToSell` sum wrong *and* make every
      compare-and-swap update whichever row Mongo returned first: a silent oversell. Two
      hazards documented: unique indexes can't be sparse (so every unique field needs a
      backfill first, and `ProductVariant.Barcode` shouldn't get one at all), and embedded
      sub-fields can never be indexed. Applying these is yours — see `SCHEMA_BATCH_2.md`.
- [x] **Feature-gating mechanism wired** (`ecommerce-back-office/src/lib/blocks/access.ts`,
      `useHasRole`/`useHasPermission`) — the IAM `roles`/`permissions` arrays were being
      fetched but never consulted anywhere. Applied to the Edit/Delete actions across all
      three table variants (`ResourceTable`, `ProductTable`, `WarehouseCardGrid`) and
      `WarehouseDetailPage`'s scoped views, via a per-schema config
      (`list-config.ts`'s `NO_EDIT_SCHEMAS`/`NO_DELETE_SCHEMAS`/`ADMIN_ONLY_EDIT_SCHEMAS`)
      that mirrors the actual (or P0-drafted) Data Gateway access policies — Edit/Delete
      hidden for `InventoryMovement` unconditionally (ledger immutability, in the UI now, not
      just in policy), Edit admin-gated for `WarehouseInventory`/`InventoryReservation`,
      Delete admin-gated everywhere else (matching the "only admin can delete" policy present
      on every existing schema).
- [ ] **The plan's nine backoffice roles (§13) don't exist as real IAM roles yet** — the
      gating above only checks whatever roles a token actually carries (today, realistically
      just the built-in `admin`). Creating `blocks iam roles create` entries for Order
      manager/Warehouse manager/Inventory operator/etc. is IAM-CLI work, left to you per this
      project's CLI-stays-user-owned convention (unverified whether any already exist —
      check `blocks iam roles list` directly). **Drafted** in `IAM_ROLES_DRAFT.json`: all nine
      with `service::resource::action` permissions, plus a per-schema map saying which role
      replaces `admin` in each placeholder policy already written, so renarrowing is
      mechanical.
- [ ] **Shared inventory CAS module not started.** No client-side code in either app currently
      performs a guarded reserve/commit/release write — because it can't (§1.1) and because
      nothing calls `WarehouseInventory`/`InventoryMovement` for a business operation yet,
      only for CRUD-as-data-entry.

---

## 3. Phase 1 — Catalog + basic sale

### CommerceCustomer + saved addresses
- [x] **App code wired**, same `VITE_COMMERCE_SCHEMAS_LIVE` gate as Cart/Order — auto-create/
      read on first login (`CommerceCustomerProvider`), address prefill from the customer's
      most recently saved address, and a "save this address" checkbox at checkout that
      appends to `CommerceCustomer.Addresses` on successful order placement. See
      `commerce-customer-provider.tsx` + `commerce.ts`'s `ensureCommerceCustomer`/
      `addCustomerAddress`. Inert until the draft schema is imported.
- [ ] No full saved-address *picker* yet (multiple addresses just prefill from the last one
      saved) — fine for a first pass, a real selector is a small follow-up once there's more
      than one address to choose from.

### Data Gateway
- [x] **Drafted** — `CommerceCustomer`, `Cart`, `Order` entities + `CartItem`/`OrderItem` DTOs
      (reusing the existing `Address` DTO), with access policies deliberately designed to avoid
      repeating the §1 mistakes. See `COMMERCE_SCHEMAS_DRAFT.json` +
      `COMMERCE_SCHEMAS_DRAFT.md` for the definitions and import steps. **Not pushed yet** —
      staged for you to review and import via the `blocks` CLI.
- [ ] Push + reload the draft above, then merge it into `ECOMMERCE_INVENTORY_SCHEMAS.json` and
      regenerate `schema-meta.ts` in both apps (see `COMMERCE_SCHEMAS_DRAFT.md` for the exact
      order — regenerating before the push+reload will break both apps).
- [ ] Add a real MongoDB unique index on `Order.IdempotencyKey` (the field exists in the
      draft; the index doesn't — `IsUniqueData` alone isn't a constraint).
- [ ] Fix §1 policy issues on the 11 existing entities (separate from the above).
- [x] **Small tooling bug found while consolidating docs into `bwb-ecommerce-docs/` — fixed.**
      `scripts/gen-schema-meta.mjs` (both apps) didn't emit `isComplexFieldType`/
      `PRIMITIVE_FIELD_TYPES` — that helper was hand-maintained directly in the committed
      "AUTO-GENERATED — do not hand-edit" `schema-meta.ts` and imported by `collections.ts`
      and `field-input.tsx`, so running the regenerator silently deleted it. The generator
      now emits it (sequence step S1), and regenerating against the current schema export is
      a byte-for-byte no-op in both apps — `node scripts/gen-schema-meta.mjs && git diff
      --stat src/lib/blocks/schema-meta.ts` producing no output is the standing check. Fixed
      before the Commerce import rather than after, since that import ends in a regeneration.

### Backoffice (`ecommerce-back-office`) — mostly built
- [x] Product/Variant/Category/Brand CRUD — `src/components/resource/*`,
      `src/pages/ResourceListPage.tsx`. Real GraphQL via `blocksClient.data.graphql()`, no
      mocking anywhere in this path.
- [x] Structured sub-forms for nested fields (`Media`, `Pricing`, `Dimensions`, `Attributes`,
      etc.) via `object-fieldset.tsx`/`sub-field-input.tsx`/`repeater-field.tsx` — better than
      the raw-JSON fallback the app's own `AGENTS.md` describes; worth updating that doc.
- [x] Warehouse detail page with scoped inventory/transfer management
      (`src/pages/WarehouseDetailPage.tsx`).
- [x] Dashboard with real (not fabricated) counts via cheap `totalCount` queries
      (`src/pages/DashboardPage.tsx`), explicitly and correctly declining to show
      available/reserved/low-stock numbers because the Data Gateway can't aggregate them yet.
- [x] **`ProductVariant` isn't in the sidebar/search nav** — excluded at
      `src/components/layout/nav-items.ts:55`. CRUD works if you know the URL; no discoverable
      entry point. **Fixed** — added back to `ADMIN_NAV_ITEMS` with a `Layers` icon and
      "Variants" label.
- [ ] No embedded variant editor inside the Product form — variants are a fully separate
      screen, not a natural "add variants to this product" flow.
- [x] **No product image upload — fixed.** `Media.Url` was a hand-typed string field; now
      `sub-field-input.tsx` renders a real upload control (`image-upload-field.tsx`) for it
      specifically, backed by `lib/blocks/storage.ts`'s `uploadProductImage()` (presign → PUT
      → read back the served URL via `blocksClient.data.files`). Public access modifier
      (storefront needs to render these unauthenticated), with a client-side size/type check
      as a courtesy since the platform itself doesn't enforce one yet. No flag needed — the
      Storage service is live today, unlike the drafted Commerce schemas.
- [ ] No Orders/Commerce admin screens (nothing exists to administer yet — depends on the
      Commerce schemas above existing first).
- [ ] No bulk import/export.
- [x] Permission-gated UI — see Phase 0 above. Currently scoped to Edit/Delete actions on
      the entity tables; nav-level/route-level gating and finer-grained per-resource
      permissions (beyond the built-in `admin` role) are still open.

### Storefront (`ecommerce-consumer`) — browse works, everything transactional is fake
- [x] Home page with live Product/Category/ProductVariant-backed rails
      (`src/pages/HomePage.tsx`).
- [x] Category browse + search + client-side facet/sort/price-range filtering
      (`src/pages/ProductListingPage.tsx`) — filtering itself is real data, but computed
      entirely client-side over a flat `pageSize: 100` fetch, not server-side filters.
- [x] Product detail page with gallery, variant selector, price/sale display
      (`src/pages/ProductDetailPage.tsx`).
- [x] **Cart was `localStorage` only, guest-only, no server record.** **Server sync wired**,
      gated behind the same `VITE_COMMERCE_SCHEMAS_LIVE` flag as checkout: localStorage
      remains the persistence layer for guests and as an instant-load cache; a signed-in
      customer's cart is now additionally fetched once per session and merged with local
      state (quantities summed on matching lines — real cart-merge-on-login), then kept in
      sync on every change via a debounced create/update against a server `Cart` record. Off
      by default, so nothing changes until the flag flips. See `lib/blocks/commerce.ts`
      (`getActiveCart`/`createRemoteCart`/`updateRemoteCart`) and `cart-provider.tsx`.
- [ ] **Wishlist is still localStorage-only** — `src/components/providers/wishlist-provider.tsx:16-18`,
      same pattern the cart used to have. Not wired in this pass; same approach would apply
      if a `Favourite`/`Wishlist` schema is added to the Commerce draft.
- [x] **Checkout order placement is fully simulated** — was
      `src/pages/CheckoutPage.tsx:126-144`, a `setTimeout(..., 500)` with no persisted record.
      **Real placement wired**, gated behind `VITE_COMMERCE_SCHEMAS_LIVE` (default off, so
      today's behavior is unchanged until you flip it): `src/lib/blocks/commerce.ts` builds an
      `insertOrder` mutation with price/tax/discount snapshots, an idempotency key generated
      once per checkout attempt and reused on retry, and the customer's IAM `itemId` as
      `CustomerId`. Inert until `COMMERCE_SCHEMAS_DRAFT.json` is imported — flip the env var
      once it's live. One known gap carried over honestly rather than papered over:
      `CartLine` doesn't carry a real SKU yet, so `OrderItem.Sku` falls back to
      `variantId`/`productId` (noted in `commerce.ts`'s own comment) — fix once cart-provider
      is extended to carry it, or once Cart itself moves server-side (still open below).
- [ ] **Coupons are a hardcoded array** — `src/lib/coupons.ts:1-6,13-16`
      (`DEMO_COUPONS`, two static codes), explicitly flagged in its own comment as a
      placeholder pending a real `Coupon` schema.
- [x] **No inventory-availability check anywhere — fixed.** `WarehouseInventory` is a live
      schema (unlike Cart/Order), so this needed no feature flag and works today. See
      `src/lib/blocks/inventory.ts` (`useVariantAvailability`/`useSingleVariantAvailability`,
      client-side sum of `AvailableToSell` across warehouses — the Data Gateway can't
      aggregate server-side). Wired into `ProductDetailPage` (stock status text, disabled
      Add to Cart/Buy Now when out of stock, quantity stepper capped at available stock) and
      `ProductListingPage` (an "In stock only" filter). Both respect
      `IsInventoryTracked`/`AllowBackorder` so untracked or backorderable items are never
      blocked. **Not yet covered:** the cart itself doesn't re-check stock on quantity
      increase, and checkout doesn't revalidate availability at the moment of order placement
      — real inventory *reservation* (§1.1/§4) is what actually prevents overselling; this is
      storefront UX, not the enforcement mechanism. **The enforcement mechanism now exists:**
      checkout allocates cart lines to warehouses and holds the stock via `inventory-ops.ts`
      before placing the order, releasing it if placement fails (`checkout-inventory.ts`,
      sequence step S5). Gated behind `VITE_INVENTORY_WRITES_LIVE` until §1.1 is imported.
      Still open below: re-checking stock when a quantity is raised in the cart (S7).
- [ ] Brand schema has generated metadata but zero call sites — no brand pages exist despite
      the schema being ready.
- [ ] No pagination/infinite scroll (fixed `pageSize: 100`), no search suggestions, no SEO
      metadata anywhere.
- [ ] No delivery/pickup ETA logic (static copy only).
- [ ] No reviews/ratings — no schema, no UI.
- [ ] No recommended/recently-viewed tracking.
- [ ] No real tax calculation (flat `DELIVERY_CHARGE = 120` constant) or shipping-rate logic.
- [ ] No saved-address selection (raw text inputs at checkout).
- [ ] No customer account area at all — no `/account`, `/orders`, `/profile`, `/addresses`
      routes exist in `App.tsx`; nothing beyond the login gate on `/checkout`.
- [x] `src/components/providers/auth-provider.tsx:20-24` carried a **stale comment** copied
      from a sibling app, claiming this app gates "every product/inventory screen" behind
      auth — it doesn't (confirmed no `ProtectedLayout` exists here). **Fixed** — comment now
      accurately describes this app's actual (public-by-default, checkout-gated) behavior.

---

## 4. Phase 2 — Core inventory

- [x] **Reservation lifecycle actions wired.** `ResourceListPage.tsx`'s `reservationActions()`
      adds Commit/Release/Mark expired row actions for `InventoryReservation`, shown only
      from `Status === "active"` (the plan's only valid starting state) and gated by the same
      admin-only `canEdit` this schema already requires. Each sets the matching date field
      (`CommittedDate`/`ReleasedDate`) alongside `Status`, instead of hand-typing the status
      string via the generic form. Also added: an "Overdue" badge next to the Status cell for
      any reservation still `active` past its own `ExpiresDate` (`ResourceTable`'s new
      `rowWarning` prop) — with no scheduler in this project
      (`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §6.3), something still has to notice before
      "Mark expired" is useful, and this is that noticing.
- [x] **The compare-and-swap reserve/commit/release helper module — built.**
      `src/lib/blocks/inventory-ops.ts` in both apps (mirrored, like `collections.ts`),
      implementing `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1: guarded `updateWarehouseInventory`
      on `{ItemId, Version, AvailableToSell:{gte:qty}}`, `totalImpactedData` checked, bounded
      jittered retry, an idempotent `InventoryMovement` per applied line, and cross-line
      compensation standing in for the transaction the gateway doesn't have. Three things it
      does that the spec didn't call for and that turned out to matter: `AvailableToSell` is
      recomputed from the buckets rather than adjusted (so stored drift can't slip past the
      gateway's own guard and oversell), an unreadable `Version` is fatal rather than
      degrading to an unguarded write (§1.5), and the balance is written before the ledger so
      a failed ledger row is an auditing gap rather than a phantom movement. Gated behind
      `VITE_INVENTORY_WRITES_LIVE`, **off until §1.1 is imported** — writes are denied for
      everyone today. Verified by `npm run verify:inventory` (65 assertions, 19 scenarios,
      against a simulated gateway that enforces the CAS filter the way one Mongo `UpdateOne`
      would).
- [ ] Allocation strategy logic — a placeholder exists (`checkout-inventory.ts`'s
      `allocateCartLines`: fullest warehouse first, split a line only when one warehouse can't
      cover it, treat a variant with no inventory row as untracked rather than out of stock).
      Deliberately the simplest defensible rule — it minimises how many warehouses a line
      touches, which is right when nothing is known about shipping cost or customer location.
      A real strategy (proximity, cost, split penalties) is still open; callers won't change
      when it lands.
- [x] **`StockTransfer` approval/dispatch/receive workflow — fixed**, matching the pattern
      just built for reservations. `stock-transfer-actions.ts`'s `stockTransferActions()` adds
      guided Approve (sets `ApprovedBy`/`ApprovedDate` for real, from the signed-in user, not
      an editable field)/Dispatch/Mark partially received/Mark received/Cancel actions, keyed
      off the schema's actual deployed `Status` values (draft/approved/in_transit/
      partially_received/received/cancelled — narrower than the plan's full
      Draft→Requested→Approved→Picking→Dispatched→InTransit→PartiallyReceived→Received→Closed
      lifecycle in §8.6, since this schema doesn't model every one of those as a distinct
      state). Approve/Cancel are gated on the `admin` role — an app-level choice stricter than
      this schema's actual Edit policy (any authenticated user), never looser. Same "Overdue"
      row-warning pattern as reservations, keyed off `ExpectedArrivalDate`. Wired into both
      `/admin/stock-transfer` and the scoped view on `WarehouseDetailPage`.
- [x] **`PurchaseOrder` guided workflow — fixed**, same pattern again. `purchase-order-actions.ts`
      adds Submit for approval (draft→submitted, routine, `canEdit`-gated) / Approve
      (submitted→approved, sets real `ApprovedBy`/`ApprovedDate`, `admin`-gated) / Mark
      partially received / Mark received (routine) / Close (received→closed, `admin`-gated —
      treated as a financial checkpoint like Approve, not routine) / Cancel order
      (`admin`-gated), keyed off this schema's actual seven-value `Status` set (draft,
      submitted, approved, partially_received, received, cancelled, closed — richer than
      `StockTransfer`'s, since this schema does model an explicit submit-for-approval step and
      a terminal Closed state). Same "Overdue" row-warning, keyed off
      `ExpectedDeliveryDate`. Only wired into `/admin/purchase-order` — unlike `StockTransfer`,
      `PurchaseOrder` isn't shown as its own scoped table on `WarehouseDetailPage` (only
      counted in its mini-dashboard), so there's no second call site to update.
- [x] **Refactored into a small registry** (`lifecycle-actions.ts`'s
      `LIFECYCLE_ACTIONS_BY_SCHEMA`/`ROW_WARNING_BY_SCHEMA`) rather than letting
      `ResourceListPage.tsx` grow a sequential `if (schemaName === ...)` chain — each schema's
      guided actions now live in their own `<schema>-actions.ts` module
      (`reservation-actions.ts`/`stock-transfer-actions.ts`/`purchase-order-actions.ts`,
      sharing one `LifecycleActionDeps` type), and adding a fourth is a two-line registry
      entry plus a new module, not another `if`.
- [ ] `WarehouseZones`/`Bins` as real entities (currently only an embedded `BinLocation` DTO).
- [ ] Low-stock/out-of-stock detection — explicitly and correctly omitted from the current
      dashboard (`DashboardPage.tsx:51-54` states why: no aggregation capability yet); needs
      either a precomputed summary schema or per-record client-side checking once
      `AvailableToSell` is reliably maintained (§1.1 again).
- [ ] Inventory dashboards beyond raw counts — valuation, ageing, accuracy: none exist; same
      aggregation gap.
- [x] **Immutability enforcement for `InventoryMovement` in the UI, not just policy — fixed.**
      Edit/Delete actions are now hidden unconditionally for this schema (see the
      permission-gating work in Phase 0 above) — the UI reflects the immutable-ledger rule
      even before §1.2's policy fix is imported.

---

## 5. Phase 3 — Procurement, fulfillment, payments

- [x] Suppliers and Purchase Orders have full CRUD already (`Supplier`, `PurchaseOrder`
      entities + backoffice screens) — ahead of where a strict phase-by-phase build would put
      them.
- [ ] Goods receiving workflow — no schema (`GoodsReceipt`), no UI beyond editing a
      `PurchaseOrder`'s status field directly.
- [ ] Partial receiving — same gap.
- [ ] Picking/packing/dispatch — no schemas, no UI.
- [ ] Customer returns and inspection — no schema, no UI, on either app.
- [ ] Payment integration — no payment SDK dependency in either app's `package.json`; the
      storefront's "Select Payment Method" control
      (`src/pages/CheckoutPage.tsx:177-182`) and the `PaymentPartnersBar` component
      (`src/components/storefront/payment-partners-bar.tsx`) are static/decorative, wired to
      nothing. Per `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §6.1/§6.3, this needs a PSP hosted
      checkout plus a manual staff-side confirmation screen in the backoffice — neither exists.
- [x] **Reservation expiry (lazy/client-triggered per §6.3) — built.**
      `lib/blocks/reservation-sweep.ts` in both apps: finds `active` reservations past their
      `ExpiresDate`, claims each with a compare-and-swap on its own `Status` (so two
      concurrent sweepers can't both release the same lines — which would eat into other
      reservations' stock, since `releaseStock` clamps to what's currently reserved rather
      than to this reservation's share), releases the stock, then stamps `ReleasedDate`.
      Claim-before-release deliberately trades a stranded-stock failure (reported, and visible
      as expired-with-no-released-date) for never corrupting a neighbouring reservation.
      Triggered on checkout entry and on opening the backoffice reservation list; bounded to
      25 per pass and throttled to once a minute. Gated on `VITE_INVENTORY_WRITES_LIVE`.

---

## 6. Phase 4 — Enterprise

Nothing here has been started, and nothing here is currently blocked on anything except the
foundation above: lots/serials/expiry/FIFO-FEFO, cycle counting, costing/valuation,
replenishment, approval workflows, entitlement tiers (depends on the role/permission model in
Phase 0).

---

## Appendix A — Backoffice entity-by-entity CRUD status

| Schema | Status | Notes |
|---|---|---|
| Product | DONE | Custom table UI, full sub-forms, image thumbnails |
| Category | DONE | Generic table/form, parent resolved via reference-select |
| Brand | DONE | Generic table/form |
| ProductVariant | PARTIAL | Full CRUD works, but hidden from nav (`nav-items.ts:55`) |
| Warehouse | DONE | Card-grid list + dedicated detail page |
| WarehouseInventory | **BLOCKED** | UI exists; Create/Edit currently denied by policy (§1.1) |
| InventoryReservation | DONE (but see §1.4) | Generic CRUD only, no lifecycle actions |
| InventoryMovement | **BLOCKED at the Data Gateway, fixed in the UI** | Create denied (§1.2); Delete currently allowed for admins at the policy level (contradicting immutability, pending the P0 fix import) — but Edit/Delete are no longer even offered in the backoffice UI |
| StockTransfer | DONE | Generic CRUD + scoped in Warehouse detail page |
| Supplier | DONE | Generic CRUD, custom field layout |
| PurchaseOrder | DONE | Generic CRUD, custom field layout, counted on dashboards |

## Appendix B — Storefront plan §5 checklist status

| Plan bullet | Status |
|---|---|
| Home page | PARTIAL — live rails, no CMS content/recommendations |
| Product discovery | PARTIAL — client-side facets/sort only, no brand pages, no pagination |
| Product details | PARTIAL — no availability check, no reviews, no SEO |
| Cart and checkout | **NOT REAL** — localStorage cart, simulated order placement |
| Customer account | NOT STARTED — no routes exist |

---

## Related documents

- `EXECUTION_SEQUENCE.md` — **the running order** for everything still unchecked here, split
  into a Track U (`blocks` CLI imports, user-owned) and a Track C (app code), sequenced so no
  code step waits on an import. This document stays the detailed per-task record; that one
  decides what comes next.

- `ECOMMERCE_PLATFORM_ON_BLOCKS.md` — the Blocks-native architecture this breakdown's phases
  come from, including the no-backend mitigations (manual payment confirmation, lazy
  reservation expiry) that Phase 3 above depends on.
- `BLOCKS_FEATURE_SUGGESTIONS.md` / `DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md` — platform
  gaps and security findings; §1.1-1.4 above are project-configuration bugs, distinct from the
  platform-level findings in those documents (e.g. this document's §1.3 public-read issue is a
  misconfigured access level on *this* project's schemas, not the platform's own default-public
  behavior for Storage objects covered in the Storage security doc).
- `AGENTS.md` — persisted constraint (no new backend service) and Blocks technical guidance
  this breakdown assumes throughout.
