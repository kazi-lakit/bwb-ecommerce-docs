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

### Wave B — The oversell-prevention core (the actual point of the inventory model)

- [ ] **S4. Shared compare-and-swap inventory-ops module.** `reserve` / `commit` / `release`,
      each one guarded `updateWarehouseInventory` (`Version` + `AvailableToSell: {gte: qty}`),
      `totalImpactedData` checked, bounded retry on contention, paired with an idempotent
      `InventoryMovement` ledger insert. `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1. Everything
      else inventory-shaped is UI around this. *Runtime-blocked on U1.*
- [ ] **S5. Wire reserve → place order → commit into storefront checkout**, with release on
      failure and on abandonment. *Runtime-blocked on U1 + U2.*
- [ ] **S6. Lazy reservation-expiry sweep.** No scheduler exists (§6.3), so expiry is
      client-triggered: on backoffice list load and on checkout entry. Builds on the
      "Overdue" badge already shipped. *Runtime-blocked on U1.*
- [ ] **S7. Cart and checkout stock revalidation.** The storefront checks availability on the
      PDP and listing today, but not when a quantity is raised in the cart, and not at the
      moment of placement. Closes the honest gap noted in the breakdown's §3.
- [ ] **S8. Carry a real SKU on cart lines.** Removes the `OrderItem.Sku` →
      `variantId`/`productId` fallback that `commerce.ts` currently documents against itself.

### Wave C — Commerce completion (written now, live on U2)

- [ ] **S9. Backoffice Orders admin.** List, detail, guided status transitions via the
      existing `LIFECYCLE_ACTIONS_BY_SCHEMA` registry, and the manual payment-confirmation
      screen §6.1/§6.3 requires in place of a webhook receiver.
- [ ] **S10. Storefront customer account area.** `/account`, `/orders`, `/orders/:id`,
      `/addresses`, `/profile` — none of these routes exist today.
- [ ] **S11. Saved-address picker at checkout** (today: prefill from the last saved address
      only).

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
