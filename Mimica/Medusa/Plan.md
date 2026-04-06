**Phase 1 — Foundation** Set up the Medusa project, configure the database.

**Phase 2 — Channels & Regions ** Build the sync/integration layer that receives your core commerce entities: markets (with currencies, countries, tax/payment/shipping configs), sales channels, and stock locations. This is where you encode your multi-region selling model.

**Phase 3 — Catalog Projection** Build the sync/integration layer that receives product families, variants (by SKU), and base prices from Directus and upserts them into Medusa as derived commerce projections. This should be idempotent and contract-driven. Variant options stay in Directus — Medusa only needs the SKU as the cross-boundary identity.

**Phase 4 — Pricing & Promotions** Configure Medusa's price lists per market/currency using the base prices synced from Directus. Then layer on Medusa-owned promotions and discount rules on top.

**Phase 5 — Cart & Checkout** Wire up the cart → checkout → order flow, ensuring market context (locale, currency, country) is explicit at every step. Integrate your chosen PSP(s) and shipping/fulfillment providers.

**Phase 6 — Customer Accounts** Enable customer registration, login, address management, and order history — all Medusa-owned.

**Phase 7 — Storefront Integration** Expose the Medusa API to your Next.js storefront for computed prices, inventory availability, cart operations, and checkout. Directus handles content/catalog reads; Medusa handles everything transactional.

**Phase 8 — Ops & Hardening** Inventory management, fulfillment workflows, webhook/event reliability, observability, and testing (especially around the sync boundary and checkout resilience without Directus).