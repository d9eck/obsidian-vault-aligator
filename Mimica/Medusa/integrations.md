# Integrations (Directus -> Medusa)

## 1. Intent
This document defines the integration strategy for syncing Directus source data into Medusa. It is optimized for AI-driven development: short, prescriptive, and implementation-oriented.

Medusa is the transactional commerce engine. Directus is the source of truth for editorial data and upstream-authored commerce inputs. All integration behavior MUST be implemented as explicit Medusa modules and workflows.

## 2. Source-of-record boundary

| Data | System of record | Read by / replicated to | Notes |
| --- | --- | --- | --- |
| Ecommerce data entities | Directus | Medusa | Entities include regions, currencies, markets. A Directus market projection may also create or update Medusa sales channels and related links if required by the commerce model. |
| Product family structure | Directus | Medusa, Storefront | Canonical authoring lives in Directus. Medusa stores the commerce projection only. |
| Variant definitions and SKUs | Directus | Medusa, Storefront | SKU is immutable and is the cross-boundary identity for variants. |
| Media assets and alt text | Directus | Storefront, optional Medusa refs | Directus DAM is authoritative. Do not sync binaries into Medusa unless there is a proven commerce need. |
| Base prices | Directus | Medusa | Directus authors base prices. Medusa applies market and promotion logic on top. |

## 3. Core design rules
1. Build sync behavior as Medusa modules and workflows, not scattered helpers.
2. Every inbound payload MUST be schema-validated and versioned.
3. Every sync operation MUST be idempotent.
4. Every source entity MUST have a stable external identity.
5. Every applied change MUST be auditable and replay-safe.
6. Medusa catalog objects are commerce projections, not the editorial master model.
7. Do not duplicate Directus-only modeling concerns into Medusa unless commerce execution needs them.
8. Checkout MUST continue when Directus is unavailable because Medusa already holds the required transactional projection.

## 4. Mandatory integration modes
Every sync domain MUST support exactly three modes. The same mapping and upsert logic MUST be reused across all three modes; only the input acquisition changes.

| Mode | Purpose | Typical trigger | Rules |
| --- | --- | --- | --- |
| Bootstrap | Initial or full projection into Medusa | CLI or admin-only operational trigger | Chunked, ordered, resumable, safe to re-run. |
| Incremental | Apply new or changed source entities quickly | Directus webhook or queue consumer | Entity-level idempotent upsert, correlation IDs, retries with backoff. |
| Reconcile | Detect and repair drift between Directus and Medusa | Scheduled job plus manual replay | Compare source vs projection, emit a report, repair mismatches safely. |

## 5. Implementation pattern

### 5.1 Target structure
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

src/api/internal/directus/webhooks/
src/jobs/
src/scripts/
```

### 5.2 Medusa usage rules
- The custom `directus` module owns contracts, source adapters, mapping state, sync runs, and dedupe receipts.
- Workflows own orchestration.
- Cross-module reads use Medusa query facilities.
- Cross-module associations use Medusa links.
- Reuse Medusa core workflows inside custom workflows whenever possible.
- API routes, jobs, and operational scripts MUST call workflows; they must not contain business logic.

### 5.3 Standard sync envelope
```json
{
  "source": "directus",
  "contract_version": "v1",
  "mode": "bootstrap | incremental | reconcile",
  "entity_type": "market | region | currency | product_family | variant | base_price",
  "event_id": "string",
  "external_id": "string",
  "source_version": "updated_at | revision | checksum",
  "correlation_id": "uuid",
  "occurred_at": "ISO-8601",
  "payload": {}
}
```

Rules:
- Reject unknown contract versions.
- Deduplicate by `source + entity_type + external_id + source_version`.
- Record `event_id` separately for webhook dedupe.
- Skip no-op updates when the source hash or version is unchanged.

## 6. Domain strategy

| Domain | Main workflow | Medusa projection | Notes |
| --- | --- | --- | --- |
| Commerce topology | `syncCommerceTopologyWorkflow` | Region, currency/store config, sales channel and related links as needed | Directus markets are business concepts. Their projection may touch multiple Medusa entities. Import order: currencies -> regions -> markets/channels/links. |
| Catalog projection | `syncCatalogProjectionWorkflow` | Product and product variants | Product families project to products. Variants project to Medusa variants. Keep only commerce-relevant fields in Medusa. |
| Pricing projection | `syncPricingProjectionWorkflow` | Pricing module price data | Base prices come from Directus. Promotions and discount rules remain Medusa-owned. |

### 6.1 Commerce topology rules
- Treat a Directus market as the upstream business object.
- The Medusa projection may include region, currency assignment, sales channel creation, and provider or location links depending on the final market model.
- All topology upserts MUST be deterministic.
- Missing or deleted upstream entities MUST follow an explicit deactivate/archive policy; do not hard-delete by default.

### 6.2 Catalog projection rules
- Treat Medusa products and variants as execution-ready projections.
- SKU is immutable and is the canonical variant identity.
- Keep option richness in Directus when Medusa only needs SKU identity and commerce-relevant fields.
- If variant options remain only in Directus, the storefront MUST resolve the desired variant before adding to cart in Medusa.
- Media stays authoritative in Directus; Medusa may store references only if needed for commerce or admin usability.

### 6.3 Pricing rules
- Sync base prices separately from catalog structure.
- Price inputs MUST be keyed by stable source identity plus currency and market context.
- Pricing sync MUST be replay-safe and must not overwrite Medusa-owned promotions.

## 7. Execution pattern
```text
webhook | scheduled job | CLI/admin trigger
  -> workflow(mode, scope)
  -> validate contract
  -> acquire lock
  -> load current Medusa projection
  -> map source -> Medusa projection
  -> create/update/link via Medusa workflows and module services
  -> write audit state and report
  -> emit structured result
```

Rules:
- Use workflows for operations that require consistency and compensation.
- Use subscribers only for asynchronous side effects after the main transaction completes.
- The same business logic MUST be callable from webhook, scheduled job, and operational replay.

## 8. Operational state and reliability
The `directus` module MUST persist at least:
- `external_refs`: source ID to Medusa ID mappings.
- `processed_messages`: webhook receipts and idempotency records.
- `sync_runs`: bootstrap, replay, and reconcile execution records.
- `reconcile_reports`: drift and repair summaries.

Each record MUST capture enough metadata to answer:
- which source payload changed the record,
- when it was applied,
- which release handled it,
- whether it is safe to replay.

Reliability requirements:
- per-entity or per-batch locking,
- retries with backoff,
- poison-message or failed-item handling,
- structured logs,
- correlation IDs on every run,
- replay by entity, batch, or time window.

## 9. Delivery and testing criteria
A sync domain is not complete until it supports:
1. bootstrap,
2. incremental,
3. reconcile,
4. replay by entity,
5. replay by batch or time window,
6. reconciliation reporting,
7. failed-item handling,
8. automated tests for duplicate events, out-of-order changes, and safe re-runs.

Recommended dependency order:
1. Commerce topology
2. Catalog projection
3. Pricing projection

## 10. Open questions

1. Does the storefront resolve variants fully in Directus, or do we want to project option values into Medusa too?
   

2. What is the delete policy for upstream removals: deactivate, archive, unpublish, or hard delete?
3. Which Directus signals are available for incremental sync: webhooks, revision IDs, `updated_at`, publish state, queue, or all of them?
4. 
5. Can you provide example payloads for one create, update, and delete event per domain?
