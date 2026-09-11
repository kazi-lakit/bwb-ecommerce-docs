# Commerce Schemas — Draft, Ready to Import

**Status:** drafted, **not pushed**. `blocks-data/` and the live Data Gateway project are
untouched — CLI access was deliberately skipped for this session (per your direction), so
these are prepared for you to review and import yourself.

**Revised:** `CartItem` gained `Slug` and `ImageUrl` fields while wiring the cart-sync code
below — without them, a cart line synced in from another device would have nothing to link to
the product page or render a thumbnail with. Caught before anything was imported, so this is a
field addition to the draft, not a migration.

**File:** `COMMERCE_SCHEMAS_DRAFT.json` — 5 schema objects, exact same JSON shape as
`ECOMMERCE_INVENTORY_SCHEMAS.json`'s existing entries (verified valid JSON, field-by-field
against the `Brand` schema as a template). Covers the Phase 1 minimum identified in
`ECOMMERCE_TASK_BREAKDOWN.md` §3 ("Author Commerce schemas... at minimum for a basic sale to
be real"):

| Schema | Type | Fields | Purpose |
|---|---|---|---|
| `CommerceCustomer` | Entity | 15 | Commerce-specific profile + saved addresses, linked to an IAM user by `UserId` |
| `Cart` | Entity | 17 | Server-side cart, replacing `ecommerce-consumer`'s current localStorage-only cart |
| `Order` | Entity | 28 | Order placement target, replacing the simulated `setTimeout` in `CheckoutPage.tsx` |
| `CartItem` | DTO | 7 | Embedded line item on `Cart.Items` |
| `OrderItem` | DTO | 9 | Embedded line item on `Order.Items`, with price/tax/discount snapshots |

`Order.ShippingAddress`/`BillingAddress` and `CommerceCustomer.Addresses` reuse the **existing**
`Address` DTO already in `ECOMMERCE_INVENTORY_SCHEMAS.json` (used today by `Warehouse`/
`Supplier`) — no new address type was created.

---

## Access policies — designed to avoid the bugs in §1 of `ECOMMERCE_TASK_BREAKDOWN.md`

Every access-level choice below was made deliberately in light of the P0 findings (public-read
on cost data, write-blocked schemas with no allow policy, an admin-delete policy wrongly
applied to an immutable-ledger schema). None of that is repeated here:

- **`CommerceCustomer`** — `Read`/`Edit` = Custom, owner-scoped (`UserId == AUTH.UserId`) plus
  an admin-role read policy (placeholder — narrow to Order manager/Customer-support once
  Phase 0's role model exists). `Write` = User (any authenticated caller may create their own
  profile). `Delete` = Custom, admin-only (appropriate here — a customer profile is a
  reasonable thing to support right-to-erasure on, unlike a ledger).
- **`Cart`** — `Read`/`Edit`/`Delete` = Custom, all owner-scoped (`CustomerId == AUTH.UserId`),
  plus an admin-delete fallback for cleanup. `Write` = User.
- **`Order`** — `Read` = Custom, owner-scoped plus an admin-role placeholder policy. `Edit` =
  Custom, **staff-only** for now (a narrower "customer can cancel their own order while it's
  still `Draft`/`PendingPayment`" policy is flagged as a deliberate follow-up, not implemented
  here — it needs a two-condition AND rule this draft didn't want to guess the exact shape of
  without a live `blocks data rules pull` to confirm against). `Write` = User (checkout creates
  it as itself). **`Delete` has no allow policy at all, intentionally** — Custom with zero
  allow policies means default-deny for everyone including admins, matching the plan's "orders
  are never hard-deleted, only cancelled via `Status`" rule. Don't add an admin-delete policy
  to this schema — that's exactly the mistake found on `InventoryMovement` in §1.2.
- **`CartItem`/`OrderItem`** are DTOs (`SchemaType: 2`) — no access levels of their own; they
  inherit whatever the containing entity's policy allows.

`CustomerId`/`UserId` fields are always meant to be **set server-side from the authenticated
caller**, never trusted from client input — the app code that creates `Cart`/`Order`/
`CommerceCustomer` records must stamp these itself rather than accept them from the request
body, or the owner-scoped RLS policies above become meaningless.

---

## How to import this when you're ready

Matches the standard workflow from the `blocks-data-gateway-configuration` skill:

```bash
blocks login                              # account session needs a fresh login
blocks use D2b2b7d30b27f41bdb97833319e639e3f   # the "bwb"/e-commerce project, not "beef"
blocks data schema pull --json            # sync current state into blocks/data/schemas/ first
```

Then, for each of the 5 objects in `COMMERCE_SCHEMAS_DRAFT.json`, create
`blocks/data/schemas/<SchemaName>.json` with that object's exact content (or merge them in
however your local pull is structured — the shape matches).

```bash
blocks data validate --json               # local-only, catches malformed JSON first
blocks data schema push --dry-run --json  # review exactly what will be created
blocks data schema push --yes --json      # after you're satisfied with the dry-run
blocks data reload --dry-run --json
blocks data reload --yes --json           # makes it live — nothing above is visible until this
```

(Or the one-shot equivalent for all of the above except the initial pull:
`blocks data sync --dry-run` → `--yes`.)

**After it's live**, three follow-ups, all noted inline in the draft:
1. Add a **real MongoDB unique index** on `Order.IdempotencyKey` (`IsUniqueData` alone is a
   check-then-act query, not a constraint — see
   `DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md` B2).
2. Merge these 5 objects into `ECOMMERCE_INVENTORY_SCHEMAS.json` (the project's schema export,
   used as the source of truth by both apps) and run
   `node scripts/gen-schema-meta.mjs` in **both** `ecommerce-back-office` and
   `ecommerce-consumer` to regenerate `schema-meta.ts`. **Do this only after the push+reload
   above succeeds** — regenerating against schemas that aren't live yet will produce a
   `schema-meta.ts` that references fields the Data Gateway doesn't have, and any screen using
   them will fail with a GraphQL "field does not exist" error.
3. Narrow the two admin-role placeholder policies (`CommerceCustomer`/`Order` "staff can read
   all ___") to whatever real roles Phase 0 ends up creating (Order manager,
   Customer-support), rather than leaving them scoped to `admin`.

## App code already wired, waiting on the import

`ecommerce-consumer` has two real, schema-backed flows wired now — both gated behind the same
`VITE_COMMERCE_SCHEMAS_LIVE` flag (checked via `import.meta.env`, **off by default**), so
today's app keeps working exactly as before until you set it to `"true"` in
`.env.local`/`.env.dev` *after* the push+reload above succeeds:

1. **Order placement** — `commerce.ts`'s `placeOrder()` + `CheckoutPage.tsx`. Real
   `insertOrder` mutation with price/tax/discount snapshots and a retry-safe idempotency key.
2. **Server-backed cart** — `commerce.ts`'s `getActiveCart`/`createRemoteCart`/`updateRemoteCart`
   + `cart-provider.tsx`. localStorage stays the persistence layer for guests and as an
   instant-load cache; a signed-in customer's cart is additionally synced to a server `Cart`
   record, fetched-and-merged once per session (quantities summed on matching lines) and kept
   in sync on every change via a debounced save.
3. **Commerce customer profile** — `commerce.ts`'s `ensureCommerceCustomer`/`addCustomerAddress`
   + `commerce-customer-provider.tsx`. Auto-creates (or reads back) the signed-in customer's
   `CommerceCustomer` profile once per session; checkout prefills the address from the most
   recently saved one and can save a new one on successful order placement.

Both are self-contained hand-written GraphQL — no dependency on `schema-meta.ts` regeneration,
so they work immediately once the schema is live, no rebuild step needed. Typechecked, linted,
and build-verified against the current (schema-less) state.

## Data (sample/seed records)

Not prepared yet — these schemas need to actually exist on the Data Gateway before any sample
`Cart`/`Order`/`CommerceCustomer` record would be valid to import. Once you've pushed and
reloaded the schemas above, say so and I'll prepare seed data (a few sample customers/carts/
orders) in the same "you review, you import" shape as this document.
