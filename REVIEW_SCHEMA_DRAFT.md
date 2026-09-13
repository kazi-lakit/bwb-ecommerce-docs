# Review Schema — Draft, Ready to Import

**Status: drafted, not pushed.** One entity, `REVIEW_SCHEMA_DRAFT.json`. Sequence step S19.
Storefront code is written and sits behind `VITE_REVIEW_SCHEMA_LIVE` (off); backoffice
moderation comes free from the generic resource screens once the schema is live and
`schema-meta.ts` is regenerated.

This is the first schema in the project to use two policy features nothing here had used
before, and both were verified against `blocks-data`'s own source rather than guessed.

---

## The moderation problem, and why it's solved in policy rather than in the app

A review has three states and only one of them is public. The obvious shape — `Read = Public`,
filter to `Status == "approved"` in the client — is wrong in a way that isn't obvious until
you say it out loud: **a Public read means anyone with the tenant key can run `getReviews`
unfiltered**, and the tenant key ships in the storefront bundle. The abusive review a moderator
just rejected stays readable by anyone who asks for it directly.

So `Read` is `Custom`, with a row-level policy whose left operand is a **schema field**:

```jsonc
{ "LeftSource": 1, "LeftOperand": "Status", "Operator": 0, "RightSource": 2, "StaticValue": "approved" }
//  SCHEMA_FIELD              EQUAL                          STATIC_VALUE
```

`DataAccessPolicyHelper.EvaluateRule` turns a rule like this into a **Mongo data filter applied
to every read** (its "Case 2: Schema field vs Token/Static — Requires data filter"). An
anonymous visitor gets approved reviews and nothing else, from the server, without the client
asking for it. Two more read policies sit alongside: customers can read their own review (so a
pending one doesn't appear to vanish), and staff can read everything.

`lib/blocks/reviews.ts` deliberately **does not** also filter by status. Doing so would imply
the client is what keeps unapproved reviews private, and someone would eventually simplify the
policy away.

## The self-approval problem, and the first CLS policy in this project

`Write` is `User` — any signed-in customer may post a review. But a document written by the
client includes every field the client chooses to send, so nothing stops a customer posting
`Status: "approved"` and publishing themselves. This is the same client-trust problem as
§1.6 (order totals), and here it *is* fixable, because the fix is field-scoped rather than
arithmetic.

`Status` and `ModeratorNote` carry **column-level policies** restricting WRITE and EDIT to
staff. The shape was previously unknown to this project — `P0_POLICY_FIXES.md` records a CLS
follow-up left undone "for lack of a verified field-policy JSON shape to copy". It is verified
now:

- A field's `AccessPolicies` is a list of the same objects as `RowLevelPolicies`, with
  `PolicyType: 1` (CLS) instead of `0` (RLS).
- **Policies are grouped by `PolicyName` across fields** to recover which fields they cover —
  `SchemaImportMapping.MapToClsPolicies` builds `FieldNames` from every field carrying a policy
  of the same name. So one policy covering two fields is written as the *same policy object,
  same name*, repeated on both fields. That grouping is not obvious and is easy to get wrong.

**This unblocks the P0 CLS follow-up** — hiding `ReorderPoint`/`ReorderQuantity`/`BinLocation`
from public reads of `WarehouseInventory` can now be drafted the same way.

## Fields that are modelled but not enforced

Stated here rather than discovered later:

- **`IsVerifiedPurchase`** — set by nobody. Proving it means checking the author's order history
  at submission time, and a client-set boolean is worth nothing (a customer can send `true`).
  It would need the same CLS treatment as `Status` plus a staff or batch process to set it.
  The badge renders if the field is true, so it works the moment something trustworthy sets it.
- **`HelpfulCount`** — modelled, no UI. Incrementing it from the client needs an edit policy on
  that field, which also permits decrementing it.
- **One review per customer per product** — enforced in the UI only. A real constraint needs a
  compound unique index on `(ProductId, CustomerId)`, which `INDEX_PLAN.json`'s endpoint
  supports; add it after import.

## How to import

```bash
blocks login
blocks use D2b2b7d30b27f41bdb97833319e639e3f
blocks data schema pull --json
# add blocks/data/schemas/Review.json from REVIEW_SCHEMA_DRAFT.json
blocks data validate --json
blocks data schema push --dry-run --json    # check the CLS policies survive the round trip
blocks data schema push --yes --json
blocks data reload --yes --json
```

**Verify the policies actually bit** before trusting them — they're the whole design:

1. Query `getReviews` with no session at all. You should see only approved rows.
2. Signed in as a customer, try `insertReview` with `Status: "approved"`. It should be refused
   or stored as pending, not published.

Then add `Review` to `ECOMMERCE_INVENTORY_SCHEMAS.json`, regenerate `schema-meta.ts` in both
apps for backoffice moderation, create the `(ProductId, CustomerId)` unique index, and set
`VITE_REVIEW_SCHEMA_LIVE=true`.
