# Coupon Schema — Draft, Ready to Import

**Status: drafted, not pushed.** One entity, `COUPON_SCHEMA_DRAFT.json`, replacing the two
hardcoded demo codes in `ecommerce-consumer/src/lib/coupons.ts`. Sequence step S18.

The app code is already written and sits behind `VITE_COUPON_SCHEMA_LIVE` (off), falling back
to the demo codes until you import. Backoffice CRUD comes free: `Coupon` will appear in the
generic resource screens once the schema is live and `schema-meta.ts` is regenerated.

---

## ⚠️ Read this before deciding whether coupons are worth having yet

Drafting this surfaced something larger than coupons, now recorded as **§1.6** of
`ECOMMERCE_TASK_BREAKDOWN.md`: **an order's monetary fields are written by the customer's
browser and nothing verifies them.**

`Order.WriteAccessLevel = User`, and `SubTotal`, `DiscountTotal` and `GrandTotal` are all sent
by the storefront (`commerce.ts`'s `placeOrder`). The Data Gateway has no hook, no computed
field and no server-side validation, so there is no mechanism on this platform that can
recompute a total from the line items and reject one that doesn't match. A customer who opens
devtools can place an order for anything at any price.

**Coupons don't make that worse, and fixing coupons doesn't fix it.** Whatever rules the
client applies, a client that ignores them posts the discount it likes. The rules in
`coupons.ts` exist to give an honest answer to an honest customer — nothing more, and the
module says so at the top.

**What actually catches it** is the manual payment confirmation already built in S9: a staff
member compares the order against what the payment provider received before marking it Paid.
That mitigation only works if the person can see what to compare, so as part of this step the
backoffice payment dialog now states the amount and says plainly that order totals come from
the customer's browser. That is the control; treat it as one.

---

## Access levels, and the one that isn't obvious

| Operation | Level | Why |
|---|---|---|
| Read | **User (1)** | **Not Public.** The tenant key ships in the storefront bundle, so a Public read would let anyone run `getCoupons` unfiltered and dump every code you have. |
| Write | Custom, staff-only | Customers never create coupons. |
| Edit | Custom, staff-only | Also covers `UsageCount`. |
| Delete | Custom, admin | A coupon isn't a ledger entry; deleting a mistake is reasonable. |

**Read = User is a real trade, not a formality.** It means a signed-out visitor can't apply a
coupon — acceptable here only because checkout already requires signing in. If guest checkout
is ever added, this decision has to be revisited, and the honest options are poor: Public read
(codes enumerable by anyone) or no coupons for guests.

**And User read is not airtight either.** Any signed-in customer can still run an unfiltered
`getCoupons` and read every code, including ones you meant for a single segment. There is no
way on this platform to evaluate a code without exposing the collection that holds it, because
evaluation has to happen somewhere and the only somewhere is the client. Treat coupon codes as
semi-public: fine for "10% off this week", wrong for "£200 off, one specific customer". This
is written up as a platform gap in `BLOCKS_FEATURE_SUGGESTIONS.md`.

## Fields worth explaining

- **`UsageLimit` / `UsageCount`** — modelled, and honestly not reliably enforceable. Redemption
  would need a compare-and-swap increment at order placement (the pattern
  `inventory-ops.ts` uses for stock) plus a policy letting customers edit that one field, which
  would also let them edit it downward. Not wired up; the client checks the count it reads, and
  a determined customer can beat it. Given the §1.6 problem above, this isn't the weak link.
- **`PerCustomerLimit`** — modelled for completeness, not enforced at all: it needs order
  history per customer at validation time, which is a second query on a path that should stay
  fast, and it's defeated by the same client-trust problem.
- **A limit of `0` means "unset", not "zero allowed"** — the same reading as an unset
  `ReorderPoint` in the inventory code. The default value of an optional number shouldn't
  behave like a deliberate zero.

## How to import

```bash
blocks login
blocks use D2b2b7d30b27f41bdb97833319e639e3f
blocks data schema pull --json
# add blocks/data/schemas/Coupon.json with the object from COUPON_SCHEMA_DRAFT.json
blocks data validate --json
blocks data schema push --dry-run --json
blocks data schema push --yes --json
blocks data reload --yes --json
```

Then:

1. Add `Coupon` to `ECOMMERCE_INVENTORY_SCHEMAS.json` and regenerate `schema-meta.ts` in both
   apps — this is what gives you backoffice CRUD with no further work.
2. Create a unique index on `Coupon.Code` (see `INDEX_PLAN.json` for the endpoint and the
   sparse-index hazard — `Code` is required, so no backfill is needed).
3. Set `VITE_COUPON_SCHEMA_LIVE=true` in the storefront.
4. Create some coupons, and delete the two demo codes from `coupons.ts` once you no longer
   need the flag-off path.
