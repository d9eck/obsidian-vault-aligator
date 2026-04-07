# Integrations (Directus -> Medusa)

## 1. Intent
This document defines the integration contract for projecting Directus source data into Medusa. It is written for AI-driven development: short, prescriptive, and implementation-oriented.

Directus is the source of truth for catalog structure, media, regional topology, and upstream-authored base prices. Medusa is the transactional commerce engine and stores execution-ready projections only.

## 2. Scope and canonical model
- All integration behavior MUST be implemented as explicit Medusa modules and workflows.
- Release 1 inventory is out of scope for sync. Use one Medusa-managed stock location.
- Do not use `market` as a runtime entity in implementation unless it is explicitly mapped. The active topology is:
  - `regions` in Directus -> Medusa `Region`
  - `touchpoints` in Directus -> upstream selling-surface metadata
  - `channels` in Directus -> Medusa `SalesChannel`
- A Directus channel is the product of one `touchpoint × region`. Its code is immutable and generated from that matrix.
- Currency is owned by Directus `regions`. One Directus region maps to one Medusa region and one currency.
- Provider and stock-location links are deferred for now.

## 3. Source-of-record boundary

| Data | System of record | Read by / replicated to | Notes |
| --- | --- | --- | --- |
| Commerce topology | Directus | Medusa | Canonical collections are `regions`, `touchpoints`, and generated `channels`. Currency, countries, locales, and measurement system live on `regions`. |
| Product family structure | Directus | Medusa, Storefront | `families`, `family_variants`, and `product_models` are authoritative authoring structures. Medusa stores only the commerce projection needed for runtime. |
| Sellable SKUs | Directus | Medusa, Storefront | `products` is the sellable SKU collection. `sku` is immutable and is the cross-boundary identity. `product_variants` is the Directus L1 grouping for 2-axis families, not the sellable record. |
| Media assets and alt text | Directus | Storefront, optional Medusa refs | Directus DAM is authoritative. Do not copy binaries into Medusa unless a concrete commerce need appears. |
| Base prices | Directus | Medusa | Base prices are channel-scoped and SKU-scoped. Current assumption: they are sourced from the `channel_values` relation behind `products.channel_values`. |

## 4. Core design rules
1. Build sync behavior as Medusa modules and workflows, not scattered helpers.
2. Every inbound payload MUST be schema-validated and versioned.
3. Every sync operation MUST be idempotent.
4. Every source entity MUST have a stable external identity.
5. Every applied change MUST be auditable and replay-safe.
6. Medusa catalog objects are commerce projections, not the editorial master model.
7. Do not duplicate Directus-only modeling concerns into Medusa unless commerce execution needs them.
8. Checkout MUST continue when Directus is unavailable because Medusa already holds the required transactional projection.
9. The same mapping and upsert logic MUST be reused across bootstrap, incremental, and reconcile modes.

## 5. Mandatory integration modes
Every sync domain MUST support exactly three modes.

| Mode | Purpose | Input acquisition | Rules |
| --- | --- | --- | --- |
| Bootstrap | Initial or full projection into Medusa | Bulk pull from Directus collections | Ordered, chunked, resumable, safe to re-run. |
| Incremental | Apply newly changed source entities quickly | Webhook fast path plus revision-cursor catch-up | Entity-level idempotent upsert, ordered where required, retryable. |
| Reconcile | Detect and repair drift between Directus and Medusa | Scheduled compare plus manual replay | Compare source vs projection, emit a report, repair mismatches safely. |

Bootstrap order for the first implementation:
1. `regions`
2. `touchpoints`
3. `channels`
4. `families`
5. `family_variants`
6. `product_models`
7. `product_variants`
8. `products`
9. `channel_values` / base prices

## 6. Incremental signal contract

### 6.1 Webhook fast path
Use Directus Flows with HTTP actions as the low-latency signal path.

Supported triggers:
- `items.create`
- `items.update`
- `items.delete`
- `flows.trigger` for manual or scheduled backfills

Rules:
- Every webhook payload MUST include a stable business identity when available, not only the record PK. Examples: `region.code`, `channel.code`, `product_model.id`, `sku`.
- Webhooks are the fast path, not the only source of truth.
- Webhook dedupe MUST be based on `event_id` if provided, plus source entity identity and version.
- Content-change flows and publication-state flows SHOULD be separate.

### 6.2 Revision cursor catch-up
Use `directus_revisions` as the durable pull signal.

Rules:
- Poll revisions by monotonically increasing `id`.
- Checkpoint the last successfully processed revision ID.
- Revisions are the authoritative catch-up mechanism after downtime, missed webhooks, or bulk edits.
- Revisions MUST be replayable by time window or cursor window.
- Prefer revision ID over `updated_at` as the primary incremental cursor.

Reference pattern:
```text
GET /revisions?filter[id][gt]=<last_seen>&sort=id
```

### 6.3 Publication-state gate
Publication is a first-class integration contract.

Rules:
- React to `status` transitions explicitly, not only generic field edits.
- Use a dedicated Directus Flow for publication transitions.
- `draft -> published` creates or activates the Medusa projection.
- `published -> archived` or `active -> inactive` deactivates the Medusa projection without breaking historical orders.
- Hard deletes MUST not remove historical commerce records.

### 6.4 Auxiliary timestamps
`date_updated` / `updated_at` may be used for diagnostics, reporting, or secondary filters, but they are not the primary replay cursor.

## 7. Projection mapping

### 7.1 Commerce topology
- Directus `regions` map 1:1 to Medusa regions.
- Directus `touchpoints` define the selling surface vocabulary and should be stored in the custom `directus` module, and optionally mirrored into Medusa sales-channel metadata.
- Directus `channels` map 1:1 to Medusa sales channels.
- `channels` are generated from the `touchpoint × region` matrix and should be treated as deterministic derived records.
- Region-owned values such as currency, countries, locales, and measurement system MUST come from Directus `regions`, not from per-channel overrides.
- Stock-location sync is not part of release 1.

### 7.2 Catalog projection
- `families` and `family_variants` shape authoring and mapping rules. They do not need 1:1 Medusa entities.
- `product_models` map to Medusa `Product`.
- Directus `products` map to Medusa `ProductVariant`.
- Directus `product_variants` is the L1 grouping for 2-axis families. It should remain an authoring concern unless Medusa option projection is later required.
- `sku` is immutable and is the canonical cross-boundary identity for the Medusa variant.
- Keep option richness in Directus when Medusa only needs SKU identity and commerce-relevant runtime fields.
- Media and localized alt text remain authoritative in Directus.
- Current working assumption: a SKU is export-eligible only when its parent projection chain is publishable and `product_models.enabled = true`.

### 7.3 Pricing projection
- Base prices are authored per `SKU × channel`.
- Because a channel is `touchpoint × region`, price lookup MUST include both regional context and channel context.
- Project prices onto Medusa variant price sets.
- Medusa-owned promotions and discount rules remain separate and MUST not be overwritten by upstream base-price sync.
- Shipping-cost absorption and marketplace-fee adjustments belong in the upstream base-price authoring model, not in Medusa promotions.

## 8. Implementation pattern

### 8.1 Target structure
```text
src/modules/directus/
  contracts/v1/
  models/
  mappers/
  service.ts

src/workflows/directus/
  sync-commerce-topology.ts
  sync-catalog-projection.ts
  sync-pricing-projection.ts
  reconcile-topology.ts
  reconcile-catalog.ts
  reconcile-pricing.ts

src/api/internal/directus/webhooks/
src/jobs/
src/scripts/
```

### 8.2 Medusa usage rules
- The custom `directus` module owns contracts, source adapters, mapping state, revision checkpoints, and dedupe receipts.
- Workflows own orchestration.
- Cross-module reads use Medusa query facilities.
- Cross-module associations use Medusa links.
- Reuse Medusa core workflows inside custom workflows whenever possible.
- API routes, jobs, and operational scripts MUST call workflows; they must not contain business logic.

### 8.3 Standard envelope
```json
{
  "source": "directus",
  "contract_version": "v1",
  "mode": "bootstrap | incremental | reconcile",
  "entity_type": "region | touchpoint | channel | product_model | product_variant_group | sku | base_price",
  "event_id": "string | null",
  "revision_id": "number | null",
  "external_id": "string",
  "business_key": "code | slug | sku",
  "occurred_at": "ISO-8601",
  "status": "draft | published | archived | active | inactive | null",
  "payload": {}
}
```

Rules:
- Reject unknown contract versions.
- Deduplicate by `source + entity_type + external_id + revision_id` when revision is available.
- Deduplicate webhook deliveries separately by `event_id`.
- Skip no-op updates when the source hash or effective version is unchanged.

## 9. Workflow pattern
```text
webhook | revision poller | scheduled reconcile | CLI/admin trigger
  -> workflow(mode, scope)
  -> validate contract
  -> acquire lock
  -> load current Directus canonical state if needed
  -> load current Medusa projection
  -> map source -> Medusa projection
  -> create/update/link via Medusa workflows and services
  -> write audit state, revision checkpoint, and report
  -> emit structured result
```

Rules:
- Use workflows for operations that require consistency and compensation.
- Use subscribers only for asynchronous side effects after the main transaction completes.
- Locks MUST protect entity-level writes for topology, catalog, and pricing upserts.
- The same business logic MUST be callable from webhook, revision poller, scheduled job, and operational replay.

## 10. Operational state and reliability
The `directus` module MUST persist at least:
- `external_refs`: source ID to Medusa ID mappings.
- `processed_messages`: webhook receipts and idempotency records.
- `revision_checkpoints`: last processed revision cursor.
- `sync_runs`: bootstrap, replay, and reconcile execution records.
- `reconcile_reports`: drift and repair summaries.

Each record MUST capture enough metadata to answer:
- which source payload changed the record,
- when it was applied,
- which release handled it,
- which revision or event produced it,
- whether it is safe to replay.

Reliability requirements:
- per-entity or per-batch locking,
- retries with backoff,
- poison-message or failed-item handling,
- structured logs,
- correlation IDs on every run,
- replay by entity, batch, cursor range, or time window.

## 11. Delivery and testing criteria
A sync domain is not complete until it supports:
1. bootstrap,
2. incremental,
3. reconcile,
4. replay by entity,
5. replay by batch, revision range, or time window,
6. reconciliation reporting,
7. failed-item handling,
8. automated tests for duplicate events, missed webhooks, out-of-order revisions, status transitions, and safe re-runs.

Recommended dependency order:
1. Commerce topology
2. Catalog projection
3. Pricing projection

## 12. Open questions
1. Please share the actual `channel_values` collection YAML so pricing field names, identities, and delete semantics can be made explicit.
2. Confirm the exact publication eligibility matrix across `product_models`, `product_variants`, `products`, and `channel_values`.
3. Confirm how hard deletes from Directus should be handled for each domain. Current assumption: deactivate or archive in Medusa unless a record never became commerce-visible.
4. Confirm whether every touchpoint should map to a Medusa sales channel, or only transactional touchpoints such as the storefront and marketplaces with checkout implications.
5. Please share example webhook payloads for:
   - content update,
   - publish / unpublish,
   - delete,
   - manual backfill via `flows.trigger`.
