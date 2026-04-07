# Integrations (Directus -> Medusa)

## 1. Intent
This document defines the integration contract for projecting Directus source data into Medusa.
It is intentionally short and prescriptive for AI-driven development.

Core boundary:
- Directus is the source of truth for commerce topology, catalog authoring, media, and upstream base prices.
- Medusa is the transactional engine and stores runtime-ready commerce projections only.
- Integration logic MUST live in explicit Medusa modules and workflows.

## 2. Source-of-record boundary

| Data | System of record | Projected / read by | Notes |
| --- | --- | --- | --- |
| Commerce topology | Directus | Medusa | Canonical collections are `regions`, `touchpoints`, and `channels`. There is no separate runtime `market` entity. |
| Product family structure | Directus | Medusa, Storefront | `families`, `family_variants`, `product_models`, and `product_variants` are authoring structures. |
| Sellable SKUs | Directus | Medusa, Storefront | `products` is the sellable SKU collection. `sku` is immutable. |
| Media, alt text, channel SEO | Directus | Storefront, optional Medusa refs | Directus DAM and translations remain authoritative. |
| Base prices | Directus | Medusa | Source collection is `products_channel_values`: one row per `SKU × channel`. |

## 3. Canonical runtime mapping

### 3.1 Commerce topology
- Directus `regions` map 1:1 to Medusa `Region`.
- Directus `channels` map 1:1 to Medusa `SalesChannel`.
- A Directus channel is the product of `touchpoint × region`; its `code` is immutable.
- Directus `touchpoints` are upstream selling-surface vocabulary. Preserve them in Medusa sales-channel metadata; do not model them as a separate commerce runtime entity.
- Region currency is owned by Directus `regions.currency`. One region maps to one Medusa region and one currency.
- Release 1 uses one stock location. Inventory and stock-location sync are deferred.
- Every touchpoint participates in Medusa rollout through generated `channels`.

### 3.2 Catalog projection
- Directus `product_models` map to Medusa `Product`.
- Directus `products` map to Medusa `ProductVariant`.
- Directus `product_variants` are L1 authoring groupings for 2-axis families. They are not first-class Medusa runtime entities unless Medusa-side option projection becomes necessary later.
- `families` and `family_variants` drive validation and mapping rules; they do not need 1:1 Medusa runtime entities.
- `sku` is the canonical cross-boundary identity for sellable items.
- Directus media and localized text remain authoritative. Only project fields Medusa needs for commerce execution.

### 3.3 Pricing projection
- Directus `products_channel_values` is the base-price source.
- One row represents one `product_id × channel_id` price.
- Base-price identity is `sku + channel.code`.
- `price` is required and is expressed in the channel's currency, which is inherited from the channel's region.
- `products_channel_values.translations` contains channel-scoped SEO only; it is not part of transactional pricing sync.
- Medusa promotions and discounts remain Medusa-owned and MUST never be overwritten by upstream base-price sync.
- Shipping-cost absorption and marketplace-fee uplift belong in upstream base-price authoring, not in Medusa promotions.
- A SKU is sellable in a channel only when the channel is active and a valid `products_channel_values` row exists for that `sku × channel`.

## 4. Identity and idempotency keys

| Domain | Business key | Notes |
| --- | --- | --- |
| Region | `regions.code` | Immutable source identity. |
| Touchpoint | `touchpoints.code` | Immutable source identity. |
| Channel | `channels.code` | Generated from `touchpoint × region`; immutable. |
| Product model | `product_models.id` | UUID source identity for authoring parent. |
| Product variant group | `product_variants.id` | UUID source identity for L1 authoring group. |
| SKU | `products.sku` | Immutable commerce identity. |
| Channel price | `products.sku + channels.code` | One base price row per channel. |

Rules:
- Every sync operation MUST be idempotent.
- Deduplicate by `business_key + revision_id` when a revision exists.
- Otherwise deduplicate by `business_key + source hash + effective status`.
- Webhook deliveries MUST also be deduplicated by `event_id` when present.

## 5. Publication and visibility rules

### 5.1 Topology
- `regions`, `touchpoints`, and `channels` are active when `status = active`.
- `channels` are deterministic derived records in Directus. Medusa consumes the Directus `channels` collection; it does not invent channels independently.

### 5.2 Catalog
A SKU is export-eligible only when all applicable upstream conditions are true:
- `families.status = published`
- `product_models.status = published`
- `product_models.enabled = true`
- `products.status = published`
- when `product_variant_id` exists, `product_variants.status = published`

### 5.3 Pricing
A channel price is active only when:
- the parent SKU is export-eligible,
- the target channel is active,
- the parent region is active,
- the `products_channel_values` row exists and has a valid price.

Absence or removal of a channel-price row removes sellability in that channel, but MUST NOT archive the SKU globally.

## 6. Mandatory integration modes
Every sync domain MUST support the same three modes and MUST reuse the same mapping and upsert logic.

| Mode | Goal | Acquisition | Output |
| --- | --- | --- | --- |
| Bootstrap | Initial or full import | Bulk pull from Directus | Full projection, chunked, resumable, safe to re-run. |
| Incremental | Apply recent changes quickly | Webhook fast path plus revision catch-up | Targeted upserts, deactivations, and replays. |
| Reconcile | Detect and repair drift | Scheduled compare plus manual replay | Drift report plus safe repair. |

## 7. Incremental signal contract

### 7.1 Webhook fast path
Use Directus Flows with HTTP actions as the low-latency signal path.

Supported source events:
- `items.create`
- `items.update`
- `items.delete`
- manual or scheduled backfill flows that call the same sync contract

Rules:
- Treat webhooks as the fast path, not the only source of truth.
- Use separate flows for generic content changes and publication-state changes.
- Webhook payloads MUST carry stable business identity whenever possible, not only the Directus PK.
- Prefer non-blocking Directus action-style flows unless blocking behavior is explicitly required.

### 7.2 Revision catch-up
Use `directus_revisions` as the durable pull signal.

Rules:
- Poll by monotonically increasing revision `id`.
- Checkpoint the last successfully processed revision ID.
- Revisions are the authoritative catch-up lane after downtime, missed webhooks, or bulk edits.
- Revisions MUST support replay by cursor range or time window.
- Refetch the canonical Directus entity before applying a mutation when the revision delta is not sufficient by itself.

Reference pattern:
```text
GET /revisions?filter[id][_gt]=<last_seen>&sort=id&limit=<n>
```

### 7.3 Publication-state lane
Publication is a first-class contract.

Rules:
- React to `status` transitions explicitly, not only generic field edits.
- Use a dedicated Directus Flow for publication transitions.
- `draft -> published` creates or activates the Medusa projection.
- `published -> archived` or `active -> inactive` deactivates or archives the Medusa projection without damaging historical orders.

## 8. Delete and deactivation policy
- Hard deletes in Directus MUST NOT hard delete historical commerce records in Medusa.
- If a source record never became commerce-visible, sync may no-op or remove only integration ledger state.
- If a source record was commerce-visible, sync MUST archive or deactivate it in Medusa.
- Deleting or deactivating a `products_channel_values` row removes only that channel-specific price and sellability.
- Deleting or deactivating a channel archives or deactivates the Medusa sales channel, but historical orders remain intact.
- Deleting or deactivating a SKU archives or deactivates the Medusa variant and removes active sellability, but historical orders remain intact.

## 9. Medusa implementation mechanism

### 9.1 Required structure
```text
src/modules/directus/
  contracts/v1/
  models/
  mappers/
  service.ts

src/workflows/directus/
  sync-topology.ts
  sync-catalog.ts
  sync-pricing.ts
  reconcile-topology.ts
  reconcile-catalog.ts
  reconcile-pricing.ts

src/api/internal/directus/webhooks/
src/jobs/
src/scripts/
```

### 9.2 Rules
- The custom `directus` module owns contracts, source adapters, mapping state, revision checkpoints, webhook receipts, sync runs, and reconcile reports.
- Workflows own orchestration.
- API routes, jobs, and scripts are wrappers only; they MUST call workflows and MUST NOT contain business logic.
- Cross-module reads use Medusa query facilities.
- Cross-module associations use Medusa links.
- Reuse Medusa core workflows whenever possible.
- Locks MUST protect entity-level writes.

Recommended lock keys:
- `region:<code>`
- `channel:<code>`
- `sku:<sku>`
- `sku:<sku>:channel:<code>`

## 10. Domain workflow contracts

### 10.1 Commerce topology workflow
Input domains:
- `regions`
- `touchpoints`
- `channels`

Responsibilities:
- upsert Medusa regions,
- upsert Medusa sales channels,
- preserve touchpoint identity in sales-channel metadata,
- link channels to the default stock location once inventory is activated.

### 10.2 Catalog workflow
Input domains:
- `families`
- `family_variants`
- `product_models`
- `product_variants`
- `products`

Responsibilities:
- fetch the canonical Directus chain,
- validate export eligibility,
- upsert Medusa products and variants by source identity and SKU,
- keep Directus-only authoring concerns out of Medusa unless runtime commerce needs them.

### 10.3 Pricing workflow
Input domain:
- `products_channel_values`

Responsibilities:
- resolve `sku + channel.code`,
- upsert base prices for the target Medusa variant,
- remove or deactivate channel-specific prices when the source row disappears or becomes inactive,
- keep Medusa promotions untouched,
- keep price resolution channel-aware; do not collapse channel-specific pricing to currency-only pricing.

Product-channel availability rule:
- Product-to-sales-channel links SHOULD be derived from active channel price coverage.
- Missing `sku × channel` base price means that SKU is not sellable in that channel even if the parent product exists.

## 11. Bootstrap order
1. `regions`
2. `touchpoints`
3. `channels`
4. `families`
5. `family_variants`
6. `product_models`
7. `product_variants`
8. `products`
9. `products_channel_values`

## 12. Minimum test matrix
Each sync domain is incomplete until it passes these tests:
- bootstrap re-run is safe,
- duplicate webhook delivery is safe,
- missed webhook is recovered by revisions,
- out-of-order updates do not corrupt the final projection,
- publish and unpublish transitions behave correctly,
- delete and archive transitions preserve historical orders,
- reconcile can detect and repair drift,
- replay by entity and replay by cursor/time window both work.

