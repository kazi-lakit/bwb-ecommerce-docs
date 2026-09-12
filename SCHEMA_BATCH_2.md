# Schema Batch 2 — Quantity Buckets, Indexes, Roles

**Status: drafted, not pushed.** Three files, one import session — sequence step S3, covering
Track U's U3, U4 and U5.

| File | What it is | Import path |
|---|---|---|
| `SCHEMA_BATCH_2.json` | 2 schema objects (`InventoryQuantity` DTO, `WarehouseInventory` entity) | `blocks data schema push` |
| `INDEX_PLAN.json` | 24 index definitions, tiered by why they matter | `POST /schemas/indexes` (REST only) |
| `IAM_ROLES_DRAFT.json` | The plan's 9 backoffice roles + how each maps onto existing placeholder policies | `blocks iam roles ...` |

---

## ⚠️ Read this first: `WarehouseInventory.Version` is typed `Long`, which is not a valid Data Gateway type

Found while drafting the index plan, and it matters more than anything else in this batch.

`ECOMMERCE_INVENTORY_SCHEMAS.json` declares `WarehouseInventory.Version` as `Long`. The
platform's own scalar list is **`String`, `Int`, `Float`, `Boolean`, `DateTime`, `ID`** and
nothing else (`blocks-data/server/DataGateway.DomainService/Helpers/GraphQlTypeHelper.cs:12`).
The Data Gateway's bulk-import validator names `Long` explicitly as a type it must reject,
with a comment saying why: *"Anything else (e.g. "Decimal", "Long") must be rejected here,
rather than being persisted and only failing later when the GraphQL schema is built"*
(`Validators/SchemaImportValidator.cs:32`).

So one of two things is true, and **you should check which before relying on this field**:

1. `Version` really is `Long` on the live project (it predates that validation) — in which case
   the generated GraphQL treats it as a composite type named `Long` that no schema defines,
   and it is unusable.
2. The export in this repo is stale or was hand-edited, and the live field is something else.

Either way this is on the critical path: **`Version` is the optimistic-concurrency guard the
entire compare-and-swap design rests on** (`ECOMMERCE_PLATFORM_ON_BLOCKS.md` §5.1, sequence
step S4). A version field that can't be read or written in GraphQL means no safe
reserve/commit/release, which means overselling.

`SCHEMA_BATCH_2.json` retypes it to **`Int`** — the gateway maps C# `long` to `Int` anyway
(`GraphQlTypeHelper.GetScalarType`), so `Int` is the correct spelling for a 64-bit counter here.

**Verify before importing:**

```bash
blocks data schema pull --json
# then check what the live WarehouseInventory actually says for Version
```

If the live type is already `Int`, drop that one change from the batch and update
`ECOMMERCE_INVENTORY_SCHEMAS.json` instead — the export, not the project, is what's wrong.

### The client-side half of this is already fixed

Both apps' `collections.ts` built a GraphQL sub-selection for every non-scalar field. For
`Version: Long` — a "composite" type with no definition — that produced `Version { }`, which
is a GraphQL **parse error that fails the entire query**, not just that field. Every
`WarehouseInventory` read through the generic collection layer would have failed: the
backoffice inventory screens and the storefront's `useVariantAvailability` alike.

Fixed in both apps as part of S3: a field whose type has no `COMPLEX_TYPES` entry is now
selected bare instead. `ID` was also added to the generator's scalar list, where it belongs.
This makes the client robust regardless of which of the two possibilities above turns out to
be true — but it does not make a `Long` field work server-side, so still fix the schema.

---

## 1. `SCHEMA_BATCH_2.json` — the missing quantity buckets

The plan specifies nine quantity buckets (`ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` §8.2);
`InventoryQuantity` has five. This adds the three that were missing:

| Bucket | Meaning | Counts against `AvailableToSell`? |
|---|---|---|
| `Blocked` | Administratively withheld — hold, recall, legal | **Yes** |
| `Backordered` | Sold beyond on-hand stock, owed to customers | No — it's a liability, not stock |
| `InTransit` | Dispatched on an outbound transfer, not yet received | No — already removed from `OnHand` at dispatch |

Which fixes the formula the CAS module (S4) will implement:

```
AvailableToSell = OnHand − Reserved − Damaged − QualityHold − Blocked
```

`Blocked` was the one genuinely missing term. Without it, blocking stock administratively had
no way to stop it being sold.

Both schema objects are included in full, with two things folded in deliberately:

- **`WarehouseInventory` carries the P0 WRITE/EDIT allow policies** from
  `P0_POLICY_FIXES.json`. This matters: `blocks data schema push` replaces a schema object
  wholesale, so pushing a `WarehouseInventory` without those policies **would silently undo
  the P0 fix** if you have already applied it. Import order between U1 and this batch is now
  safe in either direction.
- **The dotted mirror fields** (`Quantity.Blocked`, `Quantity.Backordered`,
  `Quantity.InTransit`) are added to the entity alongside the DTO fields, matching how every
  other composite field is represented in this export.

---

## 2. `INDEX_PLAN.json` — 24 indexes, and two hazards

Verified field-by-field against the platform's own implementation, not documentation:
`SchemaIndexController.cs`, `SchemaIndexService.cs`, `CreateSchemaIndexRequest.cs`,
`CreateSchemaIndexRequestValidator.cs`.

```
POST /schemas/indexes
{ "SchemaDefinitionItemId": "...", "Fields": [{"FieldName": "...", "Direction": "ASC"}], "IsUnique": true }
```

`SchemaDefinitionItemId` is not in this repo's schema export (it has no `ItemId` key), so fill
it in from `blocks data schema pull --json` or `GET /schemas` before posting.

**Tiers:**

- **Tier 1 — correctness (4).** These prevent data corruption, not slowness. The one to do
  first is `WarehouseInventory(WarehouseId, VariantId)` **unique**: nothing enforces the
  natural key today, and a duplicate balance row makes `AvailableToSell` sum wrong *and* makes
  every compare-and-swap update whichever row Mongo happens to return — a silent oversell that
  looks like a rounding error. Then the two `IdempotencyKey` uniques, which are what make
  "retry the order" safe rather than "place it twice".
- **Tier 2 — declared unique, unenforced (10).** Ten fields carry `IsUniqueData: true`, which
  is a check-then-act query, not a constraint. These make them real.
- **Tier 3 — performance (10).** Query shapes the apps actually issue today or in steps
  S6/S9/S10/S12.

### Hazard 1: unique indexes cannot be sparse

`CreateIndexOptions` sets only `Name` and `Unique` (`DbRepository.cs:396`) — no sparse, no
partial filter. MongoDB treats a missing field as `null`, so **a unique index permits exactly
one document without that field.** Before creating any Tier-1/Tier-2 unique index, backfill
the field on every existing record, or creation fails with `UNIQUE_INDEX_CONFLICT` (409) — and
worse, later inserts that legitimately omit the field start failing too.

This is why `ProductVariant.Barcode` is in `tier2_deliberately_excluded` despite being declared
unique: barcodes are genuinely optional, so a unique index there would allow exactly one
barcode-less variant in the entire catalog. Either drop `IsUniqueData` from that field or wait
for sparse-index support.

### Hazard 2: embedded sub-fields can never be indexed

`IsFieldIndexable` requires `!field.IsReferenceField` (`SchemaIndexService.cs:123-127`), and in
this project's export **every dotted sub-field has `IsReferenceField: true`**. So
`InventoryReservation.Items.VariantId` — precisely the field a "what is reserved for this
variant" lookup filters on — cannot be indexed, ever. Same for `Cart.Items`, `Order.Items`,
`WarehouseInventory.Quantity.*`, every `Address.*`.

That's a platform limitation, not a project mistake, and it shapes the data model: anything
you need to query by has to be a **top-level scalar**, even when it duplicates something inside
an embedded array. Both hazards are written up in `BLOCKS_FEATURE_SUGGESTIONS.md`.

---

## 3. `IAM_ROLES_DRAFT.json` — the plan's nine roles

`ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` §13 names nine backoffice roles. None exist beyond the
built-in `admin`, which is why **every** placeholder policy written so far — in
`P0_POLICY_FIXES.json` and `COMMERCE_SCHEMAS_DRAFT.json` alike — is scoped to `admin` with a
comment saying "narrow this once the role model exists". This is that role model.

Permissions follow the platform's own `service::resource::action` convention. The file also
carries a `schemaPolicyRenarrowing` map saying, per schema, which roles should replace `admin`
in each existing policy — so the renarrowing pass is mechanical rather than a re-derivation.

Two deliberate choices worth noting: **Inventory operator has no `::approve` permission**
(approvals belong to Warehouse manager / Procurement manager), and **Auditor is read-only by
construction** — grant it nothing ending in `::write`, `::edit` or `::delete`, or the ledger's
immutability guarantee stops being a guarantee.

**Check first** — this was drafted without CLI access, so some may already exist:

```bash
blocks iam roles list
```

---

## Import order

```
1. blocks data schema pull --json          # and check what Version actually is
2. Apply P0_POLICY_FIXES.json              # U1 — unblocks inventory writes
3. Push SCHEMA_BATCH_2.json                # U5 — buckets + Version retype (P0 policies included)
4. Push COMMERCE_SCHEMAS_DRAFT.json        # U2 — Cart/Order/CommerceCustomer
5. blocks data reload --yes --json         # nothing above is live until this
6. POST the Tier 1 indexes from INDEX_PLAN.json, after backfilling their fields
7. blocks iam roles create ... from IAM_ROLES_DRAFT.json, then renarrow the placeholder policies
8. Merge steps 3+4 into ECOMMERCE_INVENTORY_SCHEMAS.json, then in BOTH apps:
      node scripts/gen-schema-meta.mjs && git diff src/lib/blocks/schema-meta.ts
```

Step 8 last, and only after the reload in step 5 — regenerating against schemas that aren't
live yet produces a `schema-meta.ts` referencing fields the gateway doesn't have. That
regeneration is now safe to run (sequence step S1 fixed the generator's habit of deleting
`isComplexFieldType` on the way past); the diff it produces should be exactly your schema
changes and nothing else.

Once step 5 succeeds, set `VITE_COMMERCE_SCHEMAS_LIVE=true` in both apps' `.env.local` to turn
on the cart/checkout/customer code that's been written and waiting.
