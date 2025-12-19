

Solution Diagram
![[Medusa Headless Commerce-2025-12-19-105501.svg]]

```
---

config:

layout: fixed

---

flowchart LR

subgraph Cloud["Vercel / Public Cloud"]

N["Next.js Storefront<br>(Medusa Next.js Starter)"]

PDB[("Prisma Postgres<br>via Vercel Marketplace")]

end

subgraph OnPrem["On‑Prem"]

K["Keystone Server<br>(Admin UI + GraphQL)"]

A["Akeneo PIM"]

M["Medusa Backend"]

S["Sync Worker<br>(Akeneo events -&gt; Upserts)"]

WD["Wiredoor<br>Ingress-as-a-Service"]

end

N -- Prisma Client (Keystone context API)<br>read Pages/Blocks/ProductEditorial --> PDB

K -- Prisma Client --> PDB

N -- /store REST calls --> WD

WD --> M

A -- Events API Webhooks --> S

S -- Akeneo REST pull for full payload --> A

S -- Upsert ProductRef + locale snapshots --> K

S -- Sync catalog/refs as needed --> M

  

N:::svc

PDB:::db

K:::svc

A:::svc

M:::svc

S:::svc

WD:::infra

classDef svc fill:#e8f4ff,stroke:#2563eb,color:#0b1220

classDef infra fill:#ecfeff,stroke:#0891b2,color:#0b1220
```

# Elements & responsibilities (what lives where)



### Keystone (on‑prem server)

- **Editorial & experience content**: pages, blocks, hero cards, navigation, campaign landing pages.
- **Product editorial overlays** (PDP enhancements): “marketing headline”, “feature bullets”, “hero images override”, “buying guide”, etc.
- Keystone’s schema is defined in **Lists** (fields, UI config, hooks, access control), which drives both GraphQL and Admin UI.

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
    - MailiSearch

### Medusa (on‑prem)

- Source of truth for commerce behavior.
- Exposes REST endpoints (Store APIs).
- You can create custom endpoints via **API Routes** under `src/api/.../route.ts`.

### Akeneo (on‑prem)

- Source of truth for product enrichment and localization.
- For near real-time sync: use Events API/webhooks (Akeneo 5.0+/SaaS).

### Wiredoor

- Exposes your on‑prem Medusa endpoints (and optionally other internal services) through a secure reverse WireGuard tunnel and NGINX proxy.

---

# 3) Integration flows (how data moves)

## 3.1 Website content authoring (Blocks / Pages / Hero cards)

**Where is content written?**  
✅ In **Keystone Admin UI** (on‑prem Keystone server).

**How is it consumed?**

- Keystone writes content to **Prisma Postgres**.
- Next.js storefront (Vercel) reads the same tables via Prisma Client using the Keystone schema/types (the “use Keystone in Next.js” pattern). Keystone’s blog describes importing Keystone’s context API into Next.js to CRUD/query without making HTTP calls.

### Recommended Keystone content model

- `Page` (slug, locale, seo, status, blocks[])
- `Block` types:
    - `HeroBlock`
    - `CardGridBlock`
    - `PromoBannerBlock`
    - `RichTextBlock`
- `Navigation` (header/footer)
- `ProductEditorial` (editor-owned PDP enhancements)
- `ProductRef` (Akeneo-synced product reference snapshot)

This lets you do: “Editors compose pages from blocks” **in Keystone**, while the storefront renders blocks and pulls product pricing/availability from Medusa.

---

## 3.2 Commerce flow

Storefront uses Medusa Store APIs:

- `/store/*` endpoints for product, cart, checkout.
- Medusa also supports custom API routes under `src/api` if you need a facade endpoint (e.g., to return “page + products” in one call).

Traffic path: **Vercel Storefront → Wiredoor → On‑prem Medusa**

---

# 4) Deep dive: How Keystone gets updated from Akeneo

The clean approach: **Akeneo updates do not rewrite editorial content**.

## 4.1 Use Akeneo Events API → Sync worker → Upsert `ProductRef`

Akeneo’s Events API describes an event loop (receive event → ACK → process) and explicitly recommends decoupling processing (queue/worker) when event volume varies.  
It’s available since Akeneo 5.0 / SaaS.

**Flow**

1. Akeneo emits product update/create/delete → your on‑prem webhook receiver.
2. Receiver ACKs quickly, enqueues job.
3. Worker pulls full product from Akeneo REST (so you get all locale/channel values). [[prisma.io]](https://www.prisma.io/docs/postgres/introduction/overview)
4. Worker upserts `ProductRef` records in Prisma Postgres (via Keystone server/context API).

### What goes into `ProductRef` (Akeneo-owned)

Store only what the storefront needs to render and what editors need to reference:

- `akeneoUuid` / `identifier`
- `locale`
- `title` / `displayName`
- `slug` (if derived)
- `primaryImage` reference
- minimal “UX attributes” (brand, color, etc.)
- `lastSyncedAt`

### What stays editor-owned (Keystone-owned)

- `ProductEditorial`:
    - hero headline override
    - storytelling blocks
    - editorial badges
    - buying guides
    - curated cross-sell narrative

This avoids “sync wars”.

## 4.2 Locale strategy (important for many locales)

Best practice in this architecture:

- **One `Page` per locale** (`slug + locale` unique)
- **One `ProductRef` per locale** (`akeneoUuid + locale` unique)

This makes storefront queries and admin filtering straightforward and avoids JSON map complexity.

---

# 5) Critical operational considerations (because DB is shared across Vercel + on‑prem)

This is where the real architecture success/failure happens.

## 5.1 Connectivity & security

- Your on‑prem Keystone server must be able to reach **Prisma Postgres** over the internet (outbound).
- Prisma/Vercel marketplace integration sets `DATABASE_URL` for Vercel automatically. You’ll need to store the same value securely on‑prem for Keystone. [[prisma.io]](https://www.prisma.io/docs/postgres/integrations/vercel), [[vercel.com]](https://vercel.com/marketplace/prisma)

### Strongly recommended

- Use allowlisting where possible.
- If you use **Prisma Accelerate**, it provides managed connection pooling and caching, and it supports **static IP** when you need IP allowlisting. [[prisma.io]](https://www.prisma.io/docs/accelerate), [[deepwiki.com]](https://deepwiki.com/prisma/docs/6.1-prisma-accelerate-documentation)

## 5.2 Connection management (serverless reality)

Vercel functions can create connection pressure on relational DBs. Vercel explicitly recommends connection pooling for relational DBs in function environments. [[vercel.com]](https://vercel.com/kb/guide/connection-pooling-with-functions)

Prisma Postgres (as described in the Vercel marketplace and Vercel changelog) is positioned with **built-in connection pooling** and **global caching**, which helps reduce this risk. [[vercel.com]](https://vercel.com/marketplace/prisma), [[vercel.com]](https://vercel.com/changelog/prisma-joins-the-vercel-marketplace)

## 5.3 Deploy/migration order (must be disciplined)

Because both Keystone server and Next.js app share schema:

1. Update Keystone schema (Lists)
2. Apply migrations (usually from Keystone server runtime or CI)
3. Deploy Keystone server
4. Deploy Next.js storefront

Keystone schema is defined in Lists and config; DB setup is through Keystone config.  
Prisma warns about ensuring Prisma Client is regenerated at build time on Vercel to avoid schema drift (using `prisma generate` in build/postinstall). [[prisma.io]](https://www.prisma.io/docs/orm/prisma-client/deployment/serverless/deploy-to-vercel)

---

# 6) Wiredoor usage (your answer: yes)

Since Medusa and Akeneo are on‑prem and Vercel needs to reach them:

- **Vercel Storefront → Wiredoor → Medusa Store API** is correct.
- Keep Akeneo **not publicly exposed**; only your sync worker should talk to it internally.

Wiredoor is designed to expose internal services through a secure WireGuard reverse tunnel with NGINX routing.