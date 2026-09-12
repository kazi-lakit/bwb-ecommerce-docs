# E-Commerce Platform — Execution Sequence

The ordered "what to do next, and why that order" companion to
`ECOMMERCE_TASK_BREAKDOWN.md`. The breakdown says **what** is open (44 items); this says
**in what order** and **who does it**.

Two tracks run in parallel:

- **Track U — yours.** Anything touching the `blocks` CLI / Data Gateway imports. Per this
  project's standing convention, schema, policy, index and role changes are *prepared* here
  as reviewable files and *imported* by you. Nothing in Track C is allowed to run these.
- **Track C — mine.** App code in `ecommerce-back-office` / `ecommerce-consumer`, plus
  drafting the files Track U imports.

**Track C steps are ordered so that nothing is ever blocked on Track U at authoring time.**
Code that needs a not-yet-imported schema is written behind the existing
`VITE_COMMERCE_SCHEMAS_LIVE` flag (or an equivalent), exactly as Cart/Order/CommerceCustomer
already are. "Runtime-blocked on U*n*" below means the code is complete and type-checked but
inert until you import.

---

## Track U — your CLI steps

| # | Step | Prepared in | Unblocks |
|---|---|---|---|
| **U1** | Import the 4 P0 access-policy fixes | `P0_POLICY_FIXES.json` + `.md` | Task-breakdown §1 (7 checkboxes), S4, S5, S6, S17 |
| **U2** | Import the Commerce schemas (`CommerceCustomer`, `Cart`, `Order` + line DTOs) | `COMMERCE_SCHEMAS_DRAFT.json` + `.md` | Already-written cart/checkout/customer code; S5, S9, S10, S11 |
| **U3** | Create the MongoDB indexes — REST-only, `POST /schemas/indexes` | `INDEX_PLAN.json` (24, tiered) | Trustworthy idempotent writes in S4/S5 |
| **U4** | Create the backoffice IAM roles, then renarrow the `admin` placeholder policies | `IAM_ROLES_DRAFT.json` | Real permission gating beyond the built-in `admin` |
| **U5** | Import schema batch 2 (quantity buckets + the `Version` retype) | `SCHEMA_BATCH_2.json` + `.md` | Each corresponding feature |

U1 and U2 are the two highest-leverage actions in the whole project and are already drafted
and waiting. Everything else in Track U is drafted as its step comes up.

---

## Track C — ordered implementation steps

### Wave A — Unblock and de-risk (no dependencies)

- [x] **S1. Fix `gen-schema-meta.mjs` in both apps.** The generator didn't emit the
      `isComplexFieldType`/`PRIMITIVE_FIELD_TYPES` helper that lived in the "AUTO-GENERATED —
      do not hand-edit" file and is imported by `collections.ts` and `field-input.tsx`, so
      running it silently deleted a function two modules depend on. Fixed before S3/U2,
      because U2 ends in exactly that regeneration. Regenerating is now a byte-for-byte no-op
      in both apps; both typecheck clean.

- [x] **S2. Refresh the docs.** `AGENTS.md` warned that the app repos were uncommitted —
      they aren't (`ecommerce-back-office@d2fba23`, `ecommerce-consumer@2ddb608`, both pushed
      on `dev`). Warning corrected, the "Gotcha" section rewritten as a standing check now
      that S1 fixed its cause, this file added to the doc table, and "Where to pick up next"
      repointed here.

- [x] **S3. Draft schema batch 2 + the index and role payloads.** `SCHEMA_BATCH_2.json`
      (the missing `Blocked`/`Backordered`/`InTransit` buckets, plus a `Version` retype — see
      below), `INDEX_PLAN.json` (24 indexes in three tiers, verified against the platform's own
      `SchemaIndexService`), `IAM_ROLES_DRAFT.json` (the plan's nine roles, with a per-schema
      map for renarrowing the `admin` placeholders). All in `SCHEMA_BATCH_2.md`, one import
      session. Two new platform gaps written up as `BLOCKS_FEATURE_SUGGESTIONS.md` #14/#15.

      **Found while drafting, and it outranks everything else here:**
      `WarehouseInventory.Version` — the optimistic-concurrency guard the whole CAS design
      rests on — is typed `Long`, which is not in the Data Gateway's scalar list and is a type
      its own import validator rejects by name. The client half is fixed (both apps were
      emitting `Version { }`, a parse error that killed every `WarehouseInventory` query); the
      schema half is in batch 2 and needs your `blocks data schema pull` to confirm which way
      it actually is live. **S4 depends on the answer.**

### Wave B — The oversell-prevention core (the actual point of the inventory model) ✅ complete

- [x] **S4. Shared compare-and-swap inventory-ops module.** `src/lib/blocks/inventory-ops.ts`,
      mirrored byte-for-byte in both apps (this workspace has no shared package —
      `collections.ts` and `schema-meta.ts` are duplicated the same way). `reserveStock` /
      `releaseStock` / `commitStock`, each line a read → compute → guarded
      `updateWarehouseInventory(where:{ItemId, Version, AvailableToSell:{gte}}) ` → check
      `totalImpactedData`, with bounded jittered retry on contention, an idempotent
      `InventoryMovement` ledger row per applied line, and compensation across lines (a
      multi-line reserve rolls back its successful lines when a later one fails, since the
      gateway has no cross-document transaction).

      Design points worth knowing: **`AvailableToSell` is recomputed from the buckets on every
      write, never adjusted**, so drift above what the buckets support is caught locally rather
      than sailing through the gateway's own `gte` guard and overselling. A **missing or
      unreadable `Version` is fatal** — the module refuses the write rather than falling back
      to an unguarded one (the §1.5 `Long` problem). Balance is written **before** the ledger
      row, and a failed ledger write is reported (`ledgerWriteFailed`) rather than triggering a
      rollback that could itself fail. Behind `VITE_INVENTORY_WRITES_LIVE`, off by default,
      because `WarehouseInventory` writes are denied for everyone until U1 is imported.

      **Verified:** `npm run verify:inventory` in `ecommerce-consumer` — 65 assertions across
      19 scenarios (happy path, insufficient stock, contention retry and exhaustion, a real
      concurrent writer taking the stock mid-flight, multi-line rollback, drift detection and
      self-repair, unreadable `Version`, denied writes, release clamping, commit semantics,
      ledger failure, per-line idempotency keys, pre-batch-2 schema fallback, `Blocked` stock).
      The harness SSR-loads the real module through Vite with only its two gateway imports
      stubbed, so it can't drift from what ships — confirmed by injecting a regression and
      watching it fail.

- [x] **S5. Reserve → place order → release, wired into checkout.**
      `lib/blocks/checkout-inventory.ts` + `CheckoutPage.tsx`. Allocates cart lines to
      warehouses (greedy: fullest warehouse first, splitting a line only when one can't cover
      it — a deliberate placeholder until Phase 2's real allocation strategy), writes an
      `InventoryReservation` record, holds the stock, and gives it back if order placement
      fails. A sold-out line stops checkout with a message naming the item.

      **Correction to this step as originally written:** it said "reserve → place order →
      commit". Commit is wrong here. Committing reduces *on-hand*, which is what happens when
      goods physically leave — a fulfillment action in the backoffice (S9/S24), not something
      a storefront checkout can know has occurred. Placing an order leaves the reservation
      `active`, repointed from the checkout attempt at the order it became.

      The ordering inside the hold is the substance: **the reservation record is written
      before the balances move.** There's no transaction spanning record, balances and order,
      so the sequence is picked so every crash point leaves recoverable state — the worst case
      becomes a record whose stock was never taken (harmless; the sweep releases zero) rather
      than reserved stock with nothing naming it (stranded until someone reconciles the
      ledger by hand).

      *Runtime-blocked on U1 + U2.* Verified: 43 further assertions across 11 scenarios in
      `npm run verify:inventory` — allocation preference and splitting, untracked variants,
      all-or-nothing on shortfall, record-before-stock, rollback closing the record, double
      release, order attachment leaving on-hand alone.

- [x] **S6. Lazy reservation-expiry sweep.** `lib/blocks/reservation-sweep.ts`, mirrored in
      both apps. Queries `InventoryReservation` where `Status = active` and `ExpiresDate` is
      past, claims each one, releases its stock and stamps `ReleasedDate`. Triggered on
      checkout entry (storefront, fire-and-forget) and on opening the reservation list
      (backoffice, which then refetches and reports what it freed). Bounded to 25 per pass and
      throttled to once a minute per session — it runs in front of a page load and must never
      be what makes it slow.

      **The claim is a compare-and-swap on the reservation's own `Status`**, and it carries
      the whole design. Two clients sweeping at once would otherwise both release the same
      lines, and since `releaseStock` clamps to whatever is currently reserved — not to this
      reservation's share — the second release eats into stock held by *other people's*
      reservations. That's a silent oversell arriving by a different route than the one the
      CAS module guards. Filtering the claim on `Status: {eq: "active"}` means exactly one
      sweeper gets `totalImpactedData === 1`.

      Claim happens **before** release, which chooses the opposite failure: if the release
      then fails, the record reads `expired` with no `ReleasedDate`. That's reported and
      visible in the data, and strictly better than corrupting other reservations' stock.

      *Runtime-blocked on U1.* Verified: 29 assertions across 8 scenarios, including two
      sweepers racing. Removing the `Status` filter from the claim makes that scenario fail
      with 2 units eaten from a neighbouring reservation — checked, not assumed.

- [x] **S7. Cart and checkout stock revalidation.** A cart line records a price and a
      quantity, not a stock position — six in the cart stays six long after someone else buys
      the last four. `useCartStock` re-reads availability for every line: the quantity stepper
      caps at what's left, a line over its limit says so, a line with five or fewer warns, and
      Checkout is disabled while any line is short.

      **The bigger half: the checkout availability check now runs on every path, unflagged.**
      It was only reachable inside the reserve-and-hold flow, which needs both U1 and U2 — so
      until then, checkout had no stock check at all, which is exactly the state the app is in
      today. Reading `WarehouseInventory` works right now (live schema, publicly readable), so
      the check was split from the hold: the check always runs, only the write waits on the
      imports. Holding stock against the simulated order would strand it until expiry, so the
      hold is passed explicitly as `{ hold: COMMERCE_SCHEMAS_LIVE }` rather than inferred.

      Also deliberate: with holds off, a failed stock read is advisory and must not stop a
      sale; with holds on it's load-bearing, because stock that can't be read can't be held.
      Verified: 6 further assertions, and the cart/checkout messages confirmed present in the
      default flags-off bundle — the point of the change.

- [x] **S8. Carry a real SKU on cart lines.** `CartLine.sku`, captured from the variant when
      the line is added, and threaded into `OrderItem.Sku`, `CartItem.Sku` (both directions of
      the server-cart round trip), `ReservationItem.Sku` and `InventoryMovement.Sku`. Removes
      the `commerce.ts` fallback the code documented against itself.

      Optional, not required, because carts persisted in localStorage before the field existed
      are still in customers' browsers. Two details that follow from that: the cart merge
      backfills a missing local SKU from the server copy rather than letting the local line's
      absence win, and **the fallback behaviour differs on purpose** — an order line falls back
      to the variant id (a line with no SKU is worse than one carrying an id) while a ledger
      row does not (a variant id sitting in a column labelled `Sku` is a lie an auditor has no
      way to spot, and the row already carries `VariantId`).

### Wave C — Commerce completion (written now, live on U2) ✅ complete

- [x] **S9. Backoffice Orders admin.** `lib/blocks/orders.ts` + `pages/OrdersPage.tsx` +
      a nav entry and route. List with status/payment/order-number filters and paging, a
      detail drawer with line snapshots, totals and shipping address, and the staff actions.

      Deliberately **not** built on `ResourceListPage`/`createEntityApi` like every other
      entity: those read their shape from the generated `schema-meta.ts`, which has no `Order`
      in it and must not until the schema is live. Hand-written GraphQL instead, so it starts
      working the moment the import lands with no regeneration step. With the flag off the
      screen exists and explains what it's waiting for rather than 404ing.

      **This is where `commitStock` finally gets called** — the half S5 deliberately left out.
      Marking an order shipped finds the reservation it's holding, commits that stock
      (on-hand drops, availability unchanged because it was already spoken for), settles the
      reservation and records the fulfilment. Cancelling releases instead. Payment
      confirmation is a staff action against the provider's own dashboard, because the
      platform can't receive a webhook — the §6.1/§6.3 mitigation, admin-gated as a money
      decision.

      Two orderings chosen the same way as everywhere else in this wave: **stock before order**
      on fulfilment (a half-committed ship leaves the order un-shipped and reported, rather
      than shipped with quietly wrong balances) and **release before cancel** (the worst case
      is an order that still reads open, not one silently holding stock nobody will collect).

      *Runtime-blocked on U1 + U2.* Verified: `npm run verify:orders` — 41 assertions across
      12 scenarios, including another customer's reservation never being consumed, a
      different order's reservation never being consumed, a failed commit leaving the order
      un-shipped, and shipping twice not consuming stock twice. Removing the order-match
      condition makes two of them fail, checked rather than assumed.

- [x] **S10. Storefront customer account area.** `/account/orders`, `/account/orders/:orderId`,
      `/account/addresses`, `/account/profile`, plus `lib/blocks/account.ts` and a
      `RequireAuth` gate. None of these routes existed. The customer-facing other side of S9:
      order history with a single plain-English status line derived from the three separate
      status fields, order detail with the line snapshots and totals as placed, an address
      book with add and remove, and a read-only profile.

      Three deliberate calls: **`RequireAuth` shows a sign-in prompt rather than bouncing
      straight to SSO** (this is a public storefront — someone arriving from a bookmark should
      be told where they are), it is a convenience and not the security boundary (`Order`'s
      row-level policy is); **every account query filters on `CustomerId` anyway**, so a client
      that forgets to scope gets an empty list rather than silently depending on the server to
      save it; and **profile is read-only**, because deciding which IAM identity fields a
      customer may change about themselves isn't a call to make as a side effect of building
      an account page.

      Also needed `setCustomerAddresses` — `Addresses` is an embedded array, so removing one
      means rewriting the whole list, which makes last-writer-wins the real concurrency
      behaviour. Fine for a personal address book, noted where it matters.

      *Runtime-blocked on U2.* Each page explains what it's waiting for until then, rather
      than 404ing or showing an empty list that reads as a bug.

- [x] **S11. Saved-address picker at checkout.** Replaces "prefill from the last one saved":
      the profile's addresses are listed as choices, plus "Use a different address", defaulting
      once to the most recent.

      Two details that make it behave rather than just render: **the picker fills the same form
      fields a typed address would**, so validation, order placement and the save path stay one
      code path instead of two that can drift; and **the fields stay editable** — tweaking a
      saved address for one delivery switches the choice to "new" so the edit isn't silently
      stamped back over by the effect that mirrors the selection. The default is applied once,
      guarded on the choice still being unset, so a profile refresh can't undo a deliberate
      pick. "Save this address" is offered only for an address that isn't already on file.

### Wave D — Catalog and storefront depth (no dependencies)

- [ ] **S12. Brand pages.** The `Brand` schema has generated metadata and zero call sites.
- [ ] **S13. Pagination / load-more**, replacing the fixed `pageSize: 100` fetch, with
      server-side `where` filters wherever the Data Gateway can express them.
- [ ] **S14. SEO metadata** — title/description/canonical/OG/JSON-LD. None exists anywhere.
- [ ] **S15. Recently-viewed and basic recommendations.**
- [ ] **S16. Embedded variant editor inside the Product form** (variants are a wholly separate
      screen today).
- [ ] **S17. Low-stock / out-of-stock surfacing in the backoffice** — bounded client-side per
      warehouse, since the Data Gateway still can't aggregate. *Runtime-blocked on U1*, which
      is what makes `AvailableToSell` trustworthy.

### Wave E — New schema families (draft → you import → build)

Each of these is one `.json` + `.md` draft pair, then the feature on top.

- [ ] **S18. `Coupon` / `Promotion`** — replaces `src/lib/coupons.ts`'s two hardcoded demo codes.
- [ ] **S19. `Review` / `Rating`** — schema and UI, both absent.
- [ ] **S20. `Favourite`** — makes the wishlist server-backed (last localStorage-only provider).
- [ ] **S21. `TaxRate` / `ShippingRate`** — replaces the flat `DELIVERY_CHARGE = 120` constant.
- [ ] **S22. Bulk import/export** in the backoffice (CSV).

### Wave F — Procurement, fulfillment, payments

- [ ] **S23. `GoodsReceipt` + receiving workflow** against `PurchaseOrder`, including partial
      receipt — currently you can only hand-edit a PO's status field.
- [ ] **S24. `Fulfillment` / `PickList` / `Package`** + dispatch.
- [ ] **S25. `ReturnRequest` / `ReturnInspection`.**
- [ ] **S26. Payment.** PSP hosted checkout plus the manual staff-side confirmation screen
      (S9). No payment SDK exists in either app today; both payment controls are decorative.

### Wave G — Phase 4, enterprise

- [ ] **S27.** `WarehouseZone`/`Bin` as real entities, `Lot`/`SerialNumber` with FIFO-FEFO,
      `StockCount` cycle counting, `CostLayer` valuation, `ReplenishmentRule`, approval
      workflows, entitlement tiers. Each is a draft-then-build pass like Wave E; none is
      blocked on anything but the foundation above.

---

## Why this order

1. **S1 first** because every schema import in Track U ends in a regeneration, and that
   regeneration currently destroys working code. Fixing the trap costs minutes; discovering
   it mid-import costs a debugging session.
2. **Wave B before Wave C.** Orders admin and account pages are more visible, but an order
   that doesn't decrement stock atomically is a bug factory. The CAS module is the single
   mechanism preventing overselling; everything shipped so far (stock badges, disabled
   buttons, the quantity cap) is presentation over a number nothing yet maintains correctly.
3. **Wave D anywhere.** These are genuinely independent — pull one forward if you'd rather see
   storefront progress while waiting on an import.
4. **Waves E–G last** because each begins with a schema you have to import before the code
   means anything, so batching them behind the two imports already queued keeps your CLI
   sessions few.

## Status

Updated as steps complete. Checkboxes here mirror `ECOMMERCE_TASK_BREAKDOWN.md`; that file
stays the detailed record, this one stays the running order.
