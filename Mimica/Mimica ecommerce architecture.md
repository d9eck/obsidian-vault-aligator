

### Next.js Storefront (Vercel)

- Runs the Medusa starter and renders:
    - commerce flows (cart/checkout)
    - content-driven pages using Keystone content
- Links
	- https://docs.medusajs.com/resources/nextjs-starter
	- https://github.com/medusajs/nextjs-starter-medusa
	- https://vercel.com/templates/ecommerce/medusa
	- https://next.medusajs.com/
	- https://docs.medusajs.com/resources/storefront-development
	- https://github.com/keystonejs/keystone/tree/main/examples/framework-nextjs-pages-directory
- Storefront talks to:
    - **Medusa** via Store APIs (`/store/*`)
    - **KeyStone via direct Prisma Postgres** directly for CMS content (via Prisma Client) [[prisma.io]](https://www.prisma.io/docs/postgres/integrations/vercel), [[prisma.io]](https://www.prisma.io/docs/orm/prisma-client/deployment/serverless/deploy-to-vercel)
    - MeiliSearch

### Stripe & Squareup Payment Gateways
https://squareup.com/es/es/payments/tap-to-pay

### Medusa js - ecommerce backend app

- Source of truth for commerce behavior.
- Exposes REST endpoints (Store APIs).
- You can create custom endpoints via **API Routes** under `src/api/.../route.ts`.
- Install: https://docs.medusajs.com/learn/installation/docker
- User guide: https://docs.medusajs.com/user-guide

### Directus - CMS PIM & DAM

- Source of truth for product enrichment and localization.

### MeiliSearch
https://www.meilisearch.com/
https://docs.medusajs.com/v1/plugins/search/meilisearch
https://github.com/meilisearch/nextjs-starter-meilisearch-table
---

