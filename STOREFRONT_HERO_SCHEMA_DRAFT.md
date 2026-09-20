# StorefrontHero schema draft

> Deployed to `blocks-shop` on 2026-09-15 as schema `ce715cf1-28a9-410e-8fba-bb177928d036`. Live access was verified as Public read and Custom write/edit/delete with admin allow policies.

This draft supports the back-office **Storefront content** editor and the public homepage hero. It keeps one stable record per placement (`home-primary`) and stores the hero copy, CTA paths, reassurance labels, and the public Blocks Storage image reference.

Data Gateway generates the GraphQL operation name by naive concatenation, so the application correctly uses `getStorefrontHeros` even though the physical collection is named `blx_StorefrontHeroes`.

## Required access model

Configure access before enabling either app:

| Operation | Access | Reason |
|---|---|---|
| Read | Public | The signed-out consumer homepage must read the published hero. |
| Write | Custom, admin allow policy only | Only back-office administrators may create content. |
| Edit | Custom, admin allow policy only | Only back-office administrators may publish or change content. |
| Delete | Custom, admin allow policy only | Prevent public or ordinary signed-in users from removing content. |

The consumer additionally filters by both `PlacementKey = home-primary` and `Status = published`. Draft records therefore never render on the homepage.

## Safe import procedure

Target project: `D9d1f667bf6a940828c66e196544e536a` (`blocks-shop`).

1. Confirm the selected project with a read-only CLI command:

   ```bash
   blocks data schema aggregation --project D9d1f667bf6a940828c66e196544e536a --schema-name StorefrontHero --json
   ```

2. In Blocks OS, create the entity schema from `STOREFRONT_HERO_SCHEMA_DRAFT.json` and configure the four access levels in the same maintenance window: public read, custom write/edit/delete.
3. Add allow policies for the `admin` role on write (`operation: 1`), edit (`operation: 2`), and delete (`operation: 3`).
4. Reload the Data Gateway configuration.
5. Verify the result before enabling the apps:

   ```bash
   blocks data schema aggregation --project D9d1f667bf6a940828c66e196544e536a --schema-name StorefrontHero --json
   blocks data rules policy get StorefrontHero --project D9d1f667bf6a940828c66e196544e536a --json
   ```

6. Set `VITE_STOREFRONT_CONTENT_LIVE=true` in both `ecommerce-back-office` and `ecommerce-consumer`, then rebuild/redeploy both apps.
7. Open **Back office → Storefront content**, upload the hero image, enter the content, select **Published**, and save.

## CLI safety note

Do not use a bare `blocks data schema push --yes` for the initial creation with Blocks CLI 0.5.0. That command currently initializes a new schema with public access for read, write, edit, and delete. The portal workflow above avoids exposing mutations while the access policies are being configured. The applications remain feature-gated and the consumer keeps its current built-in hero until the schema is secured and enabled.

`PlacementKey` is marked `isUniqueData` for editor ergonomics, but Data Gateway uniqueness is check-then-act rather than a database constraint. The back office is the single expected writer and updates the existing `home-primary` record by `ItemId`.
