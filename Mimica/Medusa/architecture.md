# Architecture (Directus -> Medusa)

## 1. Intent
This document defines the runtime architecture for the backend implementation.

It is optimized for AI-driven development: short, prescriptive, and focused on the data that adds runtime value in Medusa.

## 2. Core decisions
- Directus is the authoring system for catalog, taxonomy, media, localized content, channel topology, and base price inputs.
- Medusa is the commerce runtime for regions, sales channels, product and variant projections, pricing, carts, checkout, orders, customers, and fulfillment.
- Every touchpoint maps to a Medusa sales channel.
- Medusa is the single order and fulfillment management system for every touchpoint.
- Release 1 uses one Medusa-managed stock location.
- Do not project Directus authoring taxonomy into Medusa as first-class entities. Keep only execution-ready commerce projections in Medusa.

## 3. System boundary

| System | Responsibility |
| --- | --- |
| Directus | Authoring, publication state, media, localized content, regional topology, SKU master, and channel-scoped base price inputs |
| Medusa | Commerce runtime: region, sales channel, product, product variant, pricing, cart, checkout, order, customer, fulfillment |
| Storefront | Reads catalog and content from Directus; uses Medusa for price, availability, cart, checkout, customer, and order operations |

## 4. Runtime projection model

| Directus source | Medusa target | Rule |
| --- | --- | --- |
| `regions` | `Region` | 1:1. Region owns countries, locales, currency, and market settings. |
| `touchpoints × regions -> channels` | `SalesChannel` | 1:1. Every channel becomes a Medusa sales channel. |
| `product_models` | `Product` | Design/common parent projected as the Medusa product. |
| `products` | `ProductVariant` | Sellable SKU projected as the Medusa variant. `sku` is immutable and is the cross-boundary identity. |
| `products_channel_values` | Variant `PriceSet` / prices | Base price is projected per `SKU × channel`, in the channel currency. |
| Media and localized catalog content | Not projected by default | Directus remains authoritative. Medusa may store references only if a concrete commerce need appears. |

## 5. What is intentionally not projected
- Authoring taxonomy and axis-definition records.
- Rich variant-selection semantics used only for authoring or storefront browsing.
- Media binaries.
- Stock-location topology beyond the single Medusa-managed location used in release 1.

If a later phase needs Medusa-side taxonomy, category navigation, or Medusa-native option selection, introduce a separate ADR before projecting more upstream structure.

## 6. Integration modes
Every sync domain MUST support the same three modes.

| Mode | Purpose | Trigger |
| --- | --- | --- |
| Bootstrap | Initial or full projection into Medusa | CLI or admin-only operational trigger |
| Incremental | Apply newly changed source entities | Directus Flow HTTP webhook fast path plus revision-cursor catch-up |
| Reconcile | Detect and repair drift | Scheduled compare plus manual replay |

Rules:
- The same mapping and upsert logic MUST be reused across all three modes.
- Webhooks are the fast path, not the only source of truth.
- Revision polling is the recovery path after missed webhooks, downtime, or bulk edits.
- Publication-state transitions MUST create, activate, deactivate, or archive projections safely.

## 7. Implementation mechanism

### 7.1 Medusa module
Use one custom Medusa module for the Directus integration.

It owns:
- source contracts,
- Directus client and adapters,
- external-to-internal ID mappings,
- processed-message receipts,
- revision checkpoints,
- sync runs and reconcile reports.

### 7.2 Workflows
Use Medusa workflows as the only orchestration mechanism.

Required workflow families:
- `syncTopologyWorkflow`
- `syncCatalogWorkflow`
- `syncPricingWorkflow`
- `reconcileTopologyWorkflow`
- `reconcileCatalogWorkflow`
- `reconcilePricingWorkflow`

Wrapper entry points:
- internal webhook route,
- scheduled jobs,
- CLI or admin-only operational triggers.

Wrappers MUST call workflows. They MUST NOT contain business logic.

## 8. Sync rules
- Every inbound payload MUST be schema-validated and versioned.
- Every mutation MUST be idempotent.
- Every entity MUST have a stable business identity.
- Locks MUST guard entity-level writes.
- Hard deletes from Directus MUST deactivate or archive in Medusa unless the record never became commerce-visible.
- Checkout MUST not depend on Directus at request time.
- Medusa-owned promotions and discounts MUST remain separate from upstream base-price sync.

## 9. Availability and pricing rules
- Sales-channel membership controls where a product is sellable.
- Base prices are channel-scoped and SKU-scoped.
- Upstream price shaping for shipping absorption or marketplace fees happens before the value reaches Medusa.
- Medusa stores the runtime price projection and applies native promotions on top.

## 10. Minimal runtime flow
```text
Directus
  -> webhook or revision poller
  -> Medusa workflow
  -> validate + lock + map
  -> upsert topology / catalog / pricing projection
  -> write audit state and checkpoints

Storefront
  -> Directus for content and catalog reads
  -> Medusa for pricing, availability, cart, checkout, and orders
```

## 11. Open questions
1. Is channel availability implied by the presence of a `products_channel_values` row, or is there a separate upstream availability flag we should honor?
2. What will the webhook authentication contract be: shared secret, signed request, network allowlist, or a combination?
3. Do we want to mirror upstream taxonomy codes into Medusa metadata for observability, or keep them completely outside Medusa?
