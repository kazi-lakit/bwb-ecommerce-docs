# Favourite Schema — Draft, Ready to Import

**Status: drafted, not pushed.** One entity, `FAVOURITE_SCHEMA_DRAFT.json`. Sequence step S20 —
the last localStorage-only provider in the storefront.

Storefront code is written and sits behind `VITE_FAVOURITE_SCHEMA_LIVE` (off). Until you
import, the wishlist behaves exactly as it always has: localStorage, per-browser.

---

## One row per favourite, not an array on `CommerceCustomer`

The array looks simpler and is worse on both counts that matter here.

An embedded array **can't be indexed or filtered on** — `IsFieldIndexable` rejects anything
with `IsReferenceField: true`, which every dotted sub-field in this project's export carries
(`INDEX_PLAN.json`, hazard 2). "Which customers favourited this product" would be unanswerable,
and it's the obvious next question someone asks of a wishlist.

Removing one entry from an array means **rewriting the whole array**, which makes
last-writer-wins the real behaviour across two open tabs — the same limitation the saved-address
book has (S10), accepted there because an address book is small and rarely edited concurrently.
A wishlist is neither.

A row per favourite removes and queries independently, and costs one extra query on sign-in.

## Owner-scoped on every operation

Read, and delete-by-the-customer, are both **SCHEMA_FIELD vs AUTH** rules
(`CustomerId == AUTH.UserId`), which the gateway compiles into data filters. So a customer
physically cannot read or delete another customer's favourites — this is isolation, not a
list the UI happens not to show.

**There is deliberately no admin read policy.** Staff have no business browsing wishlists; a
wishlist is a statement about someone's intentions, and "we can see what you're saving up for"
is not a capability to grant by default. Admin *delete* exists for right-to-erasure and spam
cleanup. If a support workflow later needs read access, that's a decision to make explicitly —
not an omission to be tidied up.

`Write` is `User`, and the app stamps `CustomerId` from the authenticated caller. A client that
sends someone else's id creates a row it then can't read, which is pointless rather than
dangerous.

## A merge never removes

On sign-in, a guest's local wishlist is unioned with the server's. Nothing is ever deleted by a
merge: an item favourited on a phone and missing locally means "not synced here", not
"removed", and there's no per-item timestamp to tell those apart. Of the two possible mistakes,
silently deleting something someone saved is the one they'd notice and mind.

## How to import

```bash
blocks login
blocks use D2b2b7d30b27f41bdb97833319e639e3f
blocks data schema pull --json
# add blocks/data/schemas/Favourite.json from FAVOURITE_SCHEMA_DRAFT.json
blocks data validate --json
blocks data schema push --dry-run --json
blocks data schema push --yes --json
blocks data reload --yes --json
```

**Check the isolation before trusting it** — it's the whole design. Signed in as customer A,
run `getFavourites` with no `where` at all. You should get only A's rows, not everyone's.

Then consider a compound unique index on `(CustomerId, ProductId)` so the same product can't be
favourited twice by one customer (`INDEX_PLAN.json` has the endpoint; both fields are required,
so no backfill is needed), and set `VITE_FAVOURITE_SCHEMA_LIVE=true`.
