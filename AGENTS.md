# bwb E-Commerce Platform — Project Docs

This repo is **documentation and planning only** — no app code lives here. It exists so any
agent (Claude Code, Codex, or a human) can pick up this project on a fresh machine with full
context, without re-deriving anything already worked out.

**Read this file first, in full, before doing anything else on this project.**

## The actual code lives in three sibling repos, not here

This project spans four repos total. On the machine this was authored on, they sit as
siblings under one workspace directory (`bwb/`); reconstruct that layout on a new machine by
cloning all four next to each other:

```
some-workspace-dir/
├── bwb-ecommerce-docs/          (this repo)
├── ecommerce-back-office/        git@github.com:kazi-lakit/bwb-ecommerce-back-office.git
├── ecommerce-consumer/           git@github.com:kazi-lakit/bwb-ecommerce-consumer.git
└── blocks-data/                  git@github.com:SELISEdigitalplatforms/blocks-data.git
```

```bash
git clone git@github.com:kazi-lakit/bwb-ecommerce-back-office.git
git clone git@github.com:kazi-lakit/bwb-ecommerce-consumer.git
git clone git@github.com:SELISEdigitalplatforms/blocks-data.git
```

- **`ecommerce-back-office`** — the staff console (React/Vite). Where most of the completed
  work in `ECOMMERCE_TASK_BREAKDOWN.md` actually lives.
- **`ecommerce-consumer`** — the public storefront (React/Vite). The rest of the completed
  work lives here.
- **`blocks-data`** — SELISE Blocks' own Data Gateway + Storage service source. **Read-only
  reference for this project** — nothing here should be modified as part of building the
  e-commerce platform; it's cited throughout these docs (`file:line`) as evidence for
  findings and feature requests, not a repo to commit to.

**All three already have their own `AGENTS.md`/`CLAUDE.md`** with app-specific conventions —
read those too, once cloned. This file covers only what's cross-cutting.

## Before switching machines: push the app repos too

These docs describe substantial code changes in `ecommerce-back-office` and
`ecommerce-consumer` (cart/order/customer wiring, inventory-availability checks, image
upload, permission-gated UI, guided lifecycle actions for reservations/transfers/purchase
orders). **That work is committed and pushed** as of 2026-09-12 —
`ecommerce-back-office@d2fba23` and `ecommerce-consumer@2ddb608`, both on `dev`. A fresh
clone of both repos' `dev` branches gets everything the task breakdown describes as done.

If you make further changes on one machine, push them before moving: these docs travel in
their own repo and will happily describe code that never left your laptop.

## What's actually in this repo

| File | What it is |
|---|---|
| `ECOMMERCE_POS_PLATFORM_BUILD_PLAN.md` | The original plan: three services (Catalog/Commerce/Inventory), three clients (storefront/POS/backoffice). Written before any Blocks-specific constraints were applied. |
| `ECOMMERCE_PLATFORM_ON_BLOCKS.md` | How that plan actually gets built on SELISE Blocks, given the hard constraint below. The architecture reference. |
| `BLOCKS_FEATURE_SUGGESTIONS.md` | Cross-service (IAM/Release/Data Gateway/Storage) platform gaps found while designing this, each tagged with the Blocks service it belongs to. |
| `DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md` | A deeper, source-cited audit of Data Gateway + Storage specifically (the two services this project leans on most) — includes two **critical** security findings. |
| `EXECUTION_SEQUENCE.md` | **The running order.** Which task to do next and why, split into a Track U (your `blocks` CLI imports) and a Track C (app code), sequenced so code work is never blocked waiting on an import. Start here; use the breakdown for detail. |
| `ECOMMERCE_TASK_BREAKDOWN.md` | **The live task tracker.** Phase-by-phase, cross-checked against actual app code, with a checkbox per task. Read this to see exactly what's done and what's next. |
| `ECOMMERCE_INVENTORY_SCHEMAS.json` | The live Data Gateway schema/policy export for this project — source of truth for what's actually deployed today (11 entities: Brand, Category, Product, ProductVariant, Warehouse, WarehouseInventory, InventoryReservation, InventoryMovement, StockTransfer, Supplier, PurchaseOrder). |
| `COMMERCE_SCHEMAS_DRAFT.json` + `.md` | Drafted (not yet imported) Commerce schemas — `CommerceCustomer`, `Cart`, `Order` + line-item DTOs — with exact import steps. |
| `SCHEMA_BATCH_2.json` + `INDEX_PLAN.json` + `IAM_ROLES_DRAFT.json` + `SCHEMA_BATCH_2.md` | Drafted (not yet imported) batch 2: the three missing quantity buckets, a 24-index plan with its two hazards, and the plan's nine IAM roles. Also documents a blocker — `WarehouseInventory.Version` is typed `Long`, which is not a valid Data Gateway type. |
| `P0_POLICY_FIXES.json` + `.md` | Drafted (not yet applied) fixes for four access-policy bugs currently blocking real inventory writes — with exact import steps. |

## The hard constraint — read before proposing any architecture

**Build entirely on SELISE Blocks platform services. Do not develop, deploy, or operate any
new backend service for this project** — no custom API server, no worker, no cron daemon, no
background process beyond the client apps themselves. This is a standing decision, not a
first-pass simplification. See `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §6.3 for the reasoning and
the mitigation patterns (client-triggered/lazy checks, manual backoffice actions) used
instead. Anything Blocks genuinely can't do goes into `BLOCKS_FEATURE_SUGGESTIONS.md` /
`DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md`, tagged by service — never worked around with
new infrastructure.

**POS is out of scope for now** (user direction) — don't add POS tasks or a POS app without
being asked again.

## Convention: schema/policy/data changes are prepared, not pushed

The user owns their own `blocks` CLI session (which project is selected, what actually gets
pushed and when). Any Data Gateway schema, policy, or seed-data change needed for this
project gets prepared as a reviewable file pair (a `.json` with the exact content + a `.md`
explaining what it is and the precise `blocks` CLI steps to import it) — see
`COMMERCE_SCHEMAS_DRAFT.*` and `P0_POLICY_FIXES.*` for the established pattern. Don't run
`blocks login`, `blocks use <tenantId>`, or any mutating `blocks data`/`blocks iam` command
for this project without being asked to.

The e-commerce project's tenant is `D2b2b7d30b27f41bdb97833319e639e3f` (project "bwb") — the
`blocks` CLI may be pointed at a different project by default; check before running anything.

## Condensed Blocks technical guidance

Verified against `blocks-data`'s own source (not assumed from documentation) — full detail
and `file:line` references in `ECOMMERCE_PLATFORM_ON_BLOCKS.md` §2/§5 and
`DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md`:

- **Compare-and-swap is the only atomicity primitive.** `updateX(where:, input:)` compiles to
  one Mongo `UpdateOne` — combine an optimistic `Version` field with a guard on the value
  being changed (e.g. `AvailableToSell: {gte: qty}`), check `totalImpactedData` on the
  response, retry on 0. No atomic `$inc` exists yet.
- **Immutability is a policy, not a flag.** `Custom` access level with zero allow policies
  denies an operation to everyone, including admins — how an append-only ledger is built.
- **`IsUniqueData` is check-then-act, not a constraint.** Real idempotency needs a genuine
  MongoDB unique index via `POST /schemas/indexes` (REST-only, not in the CLI/SDK yet).
- **No aggregation, no scheduler, no TTL index, no multi-document transaction, no inbound
  webhook receiver.** These are why payment confirmation is a manual backoffice action and
  reservation expiry is a lazy, client-triggered check rather than a background job.
- **Two critical, still-unfixed platform security findings** (not this project's bug —
  `blocks-data`'s): an Azure Storage container created with anonymous blob-read access
  (bypasses the whole DMS access-policy layer), and a legacy `filter` GraphQL argument
  deserialized as an unsanitized Mongo filter (NoSQL injection, reachable unauthenticated on
  any Public schema). See `DATA_GATEWAY_STORAGE_FEATURES_AND_SECURITY.md` A1/A2.
- **Four project-configuration bugs currently block real inventory writes** — see
  `ECOMMERCE_TASK_BREAKDOWN.md` §1 and `P0_POLICY_FIXES.md`. Fix these before building
  anything that depends on `WarehouseInventory`/`InventoryMovement`/`InventoryReservation`.

## Convention: `CLAUDE.md` points to `AGENTS.md`

This repo's `CLAUDE.md` is a single-line pointer (`@AGENTS.md`), matching every other repo in
this project. Edit `AGENTS.md`, not `CLAUDE.md`, when updating instructions.

## Note on `scripts/gen-schema-meta.mjs`

`schema-meta.ts` in both apps is fully generated from this repo's
`ECOMMERCE_INVENTORY_SCHEMAS.json` — including the `isComplexFieldType`/`PRIMITIVE_FIELD_TYPES`
helper that `collections.ts` and `field-input.tsx` import. That helper used to be
hand-maintained inside the "AUTO-GENERATED — do not hand-edit" file and *not* emitted by the
generator, so every regeneration silently deleted it; fixed 2026-09-12 (sequence step S1).
Running the generator against the current schema export is now a byte-for-byte no-op in both
apps, which is the check to repeat if you ever doubt it:

```bash
node scripts/gen-schema-meta.mjs && git diff --stat src/lib/blocks/schema-meta.ts
```

Empty output means source and generator agree. Anything else is a real schema change — read
the diff before committing it.

## Where to pick up next

**Read `EXECUTION_SEQUENCE.md`.** It carries the ordered plan (Track U = your CLI imports,
Track C = app code) and is kept current as steps complete. `ECOMMERCE_TASK_BREAKDOWN.md`
remains the detailed per-task record with file/line citations; the sequence file decides
order, the breakdown records state.

The two highest-leverage things outstanding, both already drafted and waiting only on your
`blocks` CLI session: importing `P0_POLICY_FIXES.json` (four access-policy bugs that block
all real inventory writes today) and `COMMERCE_SCHEMAS_DRAFT.json` (the Cart/Order/
CommerceCustomer code is already written and sits inert behind `VITE_COMMERCE_SCHEMAS_LIVE`
until this lands).
