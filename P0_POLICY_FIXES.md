# P0 Policy Fixes — Draft, Ready to Import

**Status:** drafted, **not applied**. Same shape as `COMMERCE_SCHEMAS_DRAFT.md` — CLI access
stays yours; this is the exact set of changes for `ECOMMERCE_TASK_BREAKDOWN.md` §1, ready for
you to review and apply.

**File:** `P0_POLICY_FIXES.json` — one object per affected schema, each with the exact
access-level change and the exact `RowLevelPolicies` objects to add or remove (same JSON shape
as a policy entry in `ECOMMERCE_INVENTORY_SCHEMAS.json`).

---

## Where these fields actually live

`ECOMMERCE_INVENTORY_SCHEMAS.json` is a combined *export* — a schema's access levels
(`ReadAccessLevel` etc.) and its `RowLevelPolicies` are shown together for convenience, but the
`blocks-data-gateway-configuration` skill describes them as two separate local files
(`blocks/data/schemas/<Name>.json` for the schema itself, `blocks/data/rules.json` for
policies). **Run `blocks data schema pull` and `blocks data rules pull` first** to see the
actual split for this project before editing — I can't confirm which file owns
`ReadAccessLevel` without a live pull, so don't assume it matches the export's layout exactly.

---

## The fixes, schema by schema

### `WarehouseInventory`
- **`ReadAccessLevel` correction, superseding an earlier version of this file:** originally
  recommended flipping this to `User (1)` alongside the other five schemas below. **Reverted
  — leave it `Public (2)`.** Once the storefront's inventory-availability check was actually
  built (`ecommerce-consumer`'s `ProductDetailPage`/`ProductListingPage`), it became clear
  anonymous visitors need to read stock levels to see "in stock"/quantity-available *before*
  logging in — the same reason `Product`/`Category`/`Brand` are public. Locking this down
  would have broken that feature the moment it shipped.
  **Follow-up worth doing, not included here** (no existing field-level/CLS policy example
  in this project to safely copy the exact JSON shape from): add **column-level security** to
  hide `ReorderPoint`, `ReorderQuantity`, `BinLocation`, and `LastCountedDate` from public
  reads — these are operational fields with no storefront use, unlike `Quantity`/
  `AvailableToSell` which the storefront actually needs. Pull the live rules first
  (`blocks data rules pull`) to see a real CLS policy shape before authoring one; this file
  didn't guess at it rather than hand you an unverified structure.
- **Add** a WRITE allow policy and an EDIT allow policy (both currently missing entirely,
  which is why creates/edits are denied to everyone today). Scoped to `admin` role as a
  placeholder — narrow to Inventory operator/Warehouse manager once Phase 0's IAM role model
  exists.
- `Delete` is untouched — its existing admin-allow policy is fine for a balance row.

### `InventoryMovement`
- `ReadAccessLevel`: **2 (Public) → 1 (User)**.
- **Add** a WRITE allow policy (currently missing — nothing can be inserted at all today).
- **Remove** the existing `"only admin can delete data"` policy entirely. Don't replace it with
  anything — Custom + zero allow policies on Delete means default-deny for everyone, which is
  exactly what an immutable ledger needs. Currently this schema's Delete is `Custom` with that
  one admin-allow policy present, meaning **admins can hard-delete movement history today** —
  removing the policy (not editing it) is the fix.
- `Edit` is untouched — it's already `Custom` with zero allow policies, i.e. already correctly
  blocked. Leave it exactly as-is; just don't let a later "fix the write bug" pass accidentally
  add an EDIT allow policy here too.

### `InventoryReservation`
- `ReadAccessLevel`: **2 (Public) → 1 (User)**.
- `WriteAccessLevel`: **2 (Public) → 1 (User)**. Today anyone unauthenticated can insert a
  reservation directly — this at least requires a session.
- **Add** an EDIT allow policy so commit/release/expire status transitions stop being blocked.
  Scoped to `admin` as a placeholder; owner-scoped editing (only the session that created a
  reservation may transition it) is flagged as a Phase 2 refinement, not attempted here.

### `StockTransfer`, `Supplier`, `PurchaseOrder`, `Warehouse`
- `ReadAccessLevel`: **2 (Public) → 1 (User)** on all four. Nothing else on these four schemas
  is broken — their `Write`/`Edit` were already `User (1)`, not `Custom`, so no policy
  additions are needed here, just the read-level change.

`Brand`, `Category`, `Product`, `ProductVariant` are **not** in this fix set — their
`Read = Public` is correct and intentional (the storefront depends on it).

---

## How to apply

```bash
blocks login
blocks use D2b2b7d30b27f41bdb97833319e639e3f
blocks data schema pull --json
blocks data rules pull --json
```

Then, per schema in `P0_POLICY_FIXES.json`:
1. Change the access-level field(s) listed in the pulled schema file.
2. Add the listed policy object(s) to `rules.json` (or wherever the pull put
   `RowLevelPolicies` for that schema) — copy the object verbatim.
3. Remove the listed policy name(s) where noted (`InventoryMovement` only).

```bash
blocks data validate --json
blocks data schema push --dry-run --json
blocks data rules deploy --dry-run --json
# review both, then:
blocks data schema push --yes --json
blocks data rules deploy --yes --json
blocks data reload --dry-run --json
blocks data reload --yes --json
```

(Or `blocks data sync --dry-run` → `--yes` for the push+deploy+reload in one step, once you've
hand-edited the local files.)

## Verify it worked

Open `ecommerce-back-office`, go to `/admin/warehouse-inventory`, and try creating a record.
Before this fix it should fail; after, it should succeed. Same check on
`/admin/inventory-movement`.

---

## Update: the CLS follow-up is no longer blocked

This document notes a column-level follow-up (hiding `ReorderPoint`/`ReorderQuantity`/
`BinLocation`/`LastCountedDate` from public reads of `WarehouseInventory`) left undone for lack
of a verified field-policy JSON shape to copy.

**That shape is now verified and demonstrated** — see `REVIEW_SCHEMA_DRAFT.json`, where `Status`
and `ModeratorNote` carry CLS policies, and the explanation in `REVIEW_SCHEMA_DRAFT.md`. In
short: a field's `AccessPolicies` holds the same objects as `RowLevelPolicies` with
`PolicyType: 1` instead of `0`, and the importer groups them **by `PolicyName` across fields**
to recover which fields each policy covers. The `WarehouseInventory` field masking can be
drafted the same way whenever you want it.
