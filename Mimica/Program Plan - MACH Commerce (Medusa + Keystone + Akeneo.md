Perfect — thanks, Daniel. These three answers materially change the plan:

1. **“Prisma”** → important clarification: **Prisma is not the database**. Prisma is the **ORM / schema & migrations layer** used by Keystone (and often other Node stacks). Keystone still needs an actual database, typically **PostgreSQL** (or MySQL/SQLite). Keystone’s config explicitly supports DB providers (Postgres/MySQL/SQLite) via Prisma providers. [[wiredoor.net]](https://www.wiredoor.net/documentation), [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/)
2. **Akeneo Community Edition** → you should assume **no Akeneo Events API / webhooks**, as Akeneo’s Events API feature is documented as available since **5.0 / SaaS**. So we plan **polling / scheduled delta sync** instead. [[vercel.com]](https://vercel.com/templates/next.js/payload-website-starter), [[docs.medusajs.com]](https://docs.medusajs.com/)
3. **~10 locales** → we’ll design content + product reference models and storefront routing for ~10 locales for MVP.

Below is the **updated Obsidian-ready implementation plan** with a Kanban board, plus a detailed **Sprint 0 + Sprint 1** backlog that reflects **polling** and a **shared Postgres** (not “Prisma”).

---

# 1) Updated architecture assumptions (locked for MVP)

## Shared database (what we will actually implement)

- **DB = PostgreSQL** (managed Postgres reachable from both Vercel and on‑prem).
- **Prisma = ORM/migrations**, used by Keystone (and optionally by other services). Keystone explicitly configures DB providers and connection URL in its config. [[wiredoor.net]](https://www.wiredoor.net/documentation), [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/)

✅ **Decision for MVP:** “Managed Postgres” (vendor TBD: could be Vercel Postgres, Neon, RDS, etc.) with:

- TLS enforced
- IP allowlisting (or private connectivity)
- separate DB users/roles for storefront read vs Keystone admin vs worker

## Akeneo integration mode

- **Akeneo CE** → implement **polling/delta export**.
- Events API/webhooks are described as available since **5.0/SaaS** → not relied upon. [[vercel.com]](https://vercel.com/templates/next.js/payload-website-starter), [[docs.medusajs.com]](https://docs.medusajs.com/)

## Locales

- MVP supports **~10 locales**
- Default modeling:
    - **Page per locale**: `Page(slug, locale)` unique
    - **ProductRef per locale**: `(akeneo_identifier, locale)` unique

## Storefront + Commerce

- Storefront is a separate Next.js app consuming Medusa Store API routes (typical `/store/*` pattern) and can be deployed independently (Vercel). [[buzzclan.com]](https://buzzclan.com/digital-transformation/mach-architecture/), [[macharchitecture.com]](https://macharchitecture.com/articles/what-is-mach-architecture)
- Medusa allows custom API routes in `src/api` for extension. [[github.com]](https://github.com/wiredoor/wiredoor), [[macharchitecture.com]](https://macharchitecture.com/articles/what-is-mach-architecture)

---

# 2) Delivery strategy (Program + Scrum)

- **2-week sprints**
- Program milestones remain, but **Akeneo work shifts** from event-driven to polling/delta.
- Keystone is deployed on‑prem as a standalone server for Admin UI and GraphQL; Next.js uses Keystone “data layer” approach + shared DB (per your decision).

---

# 3) Obsidian Kanban Board (UPDATED)

> Paste this block into Obsidian as-is.

## Backlog

- [ ] EPIC: Platform Foundations (repos, CI/CD, environments, secrets)

- [ ] EPIC: Shared Database (Postgres provisioning, roles, networking)

- [ ] EPIC: Secure Connectivity (Wiredoor ingress to on-prem Medusa; optional Keystone API)

- [ ] EPIC: Keystone CMS (schema, Admin UI, roles, page builder)

- [ ] EPIC: Storefront (Medusa Next.js Starter + Keystone data-layer reads)

- [ ] EPIC: Medusa Backend (commerce config, payments, CORS, Store API readiness)

- [ ] EPIC: Akeneo Sync (POLLING) -> Keystone/Medusa

- [ ] EPIC: Observability & Security (logging, monitoring, hardening)

- [ ] EPIC: Performance & SEO (caching/ISR, locale SEO)

- [ ] EPIC: UAT & Go-Live (training, cutover, hypercare)

  

## Ready

- [ ] STORY: Confirm DB choice (Managed Postgres) + network allowlist plan 🔺

- [ ] STORY: Define DB roles (storefront-read, keystone-admin, worker, medusa) 🔺

- [ ] STORY: Define locale list for MVP (~10 locales) + fallback locale 🔺

- [ ] STORY: Define Keystone content model v1 (Page/Blocks/Nav/ProductEditorial/ProductRef) 🔺

- [ ] STORY: Define Akeneo polling strategy (delta method + frequency + limits) 🔺

- [ ] STORY: Define SSOT boundaries (Akeneo vs Keystone vs Medusa) 🔺

  

## In Progress

- [ ] TASK: Create mono-repo or multi-repo structure (storefront / keystone / worker / infra)

- [ ] TASK: CI/CD baseline pipelines (lint/test/build/deploy) for each component

- [ ] TASK: Secrets management (env vars, rotation, least privilege)

  

## Review

- [ ] TASK: Architecture pack v1 (diagram + data contracts + endpoints)

- [ ] TASK: Security review v1 (DB exposure, tunnel scope, authN/authZ)

  

## Blocked

- [ ] BLOCKER: On-prem outbound connectivity to managed Postgres not approved

- [ ] BLOCKER: DNS/TLS constraints for Wiredoor- [ ] BLOCKER: DNS/TLS constraints for Wiredoor ingress endpoints

- [ ] BLOCKER: Payment provider sandbox keys not available

  

## Done

---

# 4) Sprint 0 (2 weeks) — “Decisions + Foundations”

## Sprint Goal

Establish deployable skeleton for **storefront + keystone + medusa**, plus the hard constraints:

- Managed Postgres connectivity
- Locale strategy
- Akeneo polling strategy

### Sprint 0 backlog (Ready → Done)

#### Story S0-1: Shared DB provisioned & reachable from Vercel + on‑prem

**Acceptance Criteria**

- Postgres instance exists
- Connectivity validated from:
    - Keystone server on‑prem
    - Next.js runtime on Vercel (or local simulated environment)
- TLS enabled
- DB roles created

**Notes**

- Keystone config requires DB provider and URL (Prisma-managed). [[wiredoor.net]](https://www.wiredoor.net/documentation), [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/)

#### Story S0-2: Repo structure + pipelines bootstrap

**Acceptance Criteria**

- Repos created (or mono-repo)
- CI runs lint/test/build for:
    - storefront
    - keystone
    - worker
- Secret handling documented

#### Story S0-3: Keystone server deployed on‑prem (connected to shared DB)

**Acceptance Criteria**

- Keystone server starts and serves:
    - Admin UI internally
    - GraphQL endpoint internally
- Uses shared DB provider configuration supported by Keystone (Postgres preferred). [[wiredoor.net]](https://www.wiredoor.net/documentation), [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/)

#### Story S0-4: Medusa backend deployed on‑prem + basic Store API reachable

**Acceptance Criteria**

- Medusa runs with at least one region/sales channel
- Store API reachable internally
- CORS/allowed origins documented (for Vercel)
- (Later: secure ingress exposure)

Medusa storefronts are separate apps consuming Store APIs. [[buzzclan.com]](https://buzzclan.com/digital-transformation/mach-architecture/), [[macharchitecture.com]](https://macharchitecture.com/articles/what-is-mach-architecture)

#### Story S0-5: Storefront deployed to Vercel (baseline)

**Acceptance Criteria**

- Next.js starter runs on Vercel
- Can call Medusa backend URL (even if temporarily mocked) Medusa Next.js starter is designed for separate hosting and config via env vars. [[coding-cloud.com]](https://coding-cloud.com/software-engineering-resources/mach-architecture), [[buzzclan.com]](https://buzzclan.com/digital-transformation/mach-architecture/)

#### Story S0-6: Akeneo CE polling strategy agreed and documented

**Acceptance Criteria**

- Polling method defined (see section 6)
- Frequency defined (e.g., every 5–10 minutes MVP)
- Delta strategy defined (cursor-based / timestamp-based / export job-based)
- Rate limits and load impact assessed

Events API/webhooks are documented as 5.0+/SaaS — not assumed. [[vercel.com]](https://vercel.com/templates/next.js/payload-website-starter), [[docs.medusajs.com]](https://docs.medusajs.com/)

---

# 5) Sprint 1 (2 weeks) — “CMS v1 + Storefront Rendering”

## Sprint Goal

Editors can author pages/blocks in Keystone Admin UI; storefront renders them using Keystone data layer reads.

### Sprint 1 backlog

#### Story S1-1: Keystone content model v1 implemented (Lists)

**Scope (minimum)**

- `Page` (slug, locale, seoTitle, seoDescription, blocks[])
- `Block` types:
    - HeroBlock
    - CardGridBlock
    - RichTextBlock
- `Navigation` (header/footer)
- `ProductRef` (placeholder model; to be filled by Akeneo polling later)
- `ProductEditorial` (optional in Sprint 1 if time)

Keystone Lists are the primary schema definition mechanism. [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/), [[wiredoor.net]](https://www.wiredoor.net/documentation)

**Acceptance Criteria**

- Editors can create:
    - Home page in 2 locales
    - One landing page with hero + cards
- Admin UI is usable (labels, sorting)

#### Story S1-2: Storefront “CMS page router” + block renderer

**Scope**

- Add route handling:
    - `/{locale}/{slug}` renders Keystone `Page`
- Implement renderers for each block type

Storefront is independent and built for customization. [[buzzclan.com]](https://buzzclan.com/digital-transformation/mach-architecture/), [[coding-cloud.com]](https://coding-cloud.com/software-engineering-resources/mach-architecture)

**Acceptance Criteria**

- `/en/home` (example) renders from Keystone content
- Blocks render reliably (hero, card grid, rich text)
- Locale selection loads correct page variant

#### Story S1-3: Keystone data-layer access in Next.js (per blog approach)

**Scope**

- Import Keystone `getContext` in Next.js server environment and query content directly (DB-based access). Keystone explicitly describes importing context API in Next.js to CRUD/query data. [[backova.com]](https://www.backova.com/platform/payload), [[wiredoor.net]](https://www.wiredoor.net/documentation)

**Acceptance Criteria**

- Next.js can query `Page` list via context API and return correct data
- No Keystone Admin UI exposed publicly
- Production build works with proper env vars

#### Story S1-4: Secure connectivity baseline (Wiredoor) for Medusa Store API

**Acceptance Criteria**

- Vercel storefront can access Medusa Store API over secure ingress
- Only required endpoints exposed Medusa Store API is the standard interface for storefronts. [[macharchitecture.com]](https://macharchitecture.com/articles/what-is-mach-architecture), [[deepwiki.com]](https://deepwiki.com/wiredoor/wiredoor-cli/5.2-service-exposure-commands)

---

# 6) Akeneo CE polling design (MVP approach)

Because you’re on **Community Edition**, we will not rely on Events API/webhooks (documented availability since 5.0/SaaS). [[vercel.com]](https://vercel.com/templates/next.js/payload-website-starter), [[docs.medusajs.com]](https://docs.medusajs.com/)

## Polling strategies (choose 1 for MVP)

### Option P1 (recommended MVP): “Delta by updated timestamp” (cursor-based)

- Maintain a cursor: `lastSuccessfulSyncAt`
- Every N minutes:
    - Fetch products updated since cursor (or use search filters where possible)
    - Upsert `ProductRef` per locale
- Store cursor in DB (`SyncState` singleton list/table)

### Option P2: Scheduled export job + import

- Run export in Akeneo (CSV/JSON)
- Worker pulls export artifact and applies changes

### Option P3: Hybrid

- Frequent “light” poll for changed identifiers
- Nightly reconciliation full scan

**MVP recommendation:** P1 + nightly reconciliation (P3 hybrid) for safety.

### Key acceptance criteria for polling MVP

- Polling job completes within timebox (e.g., <10 min)
- Idempotent (safe to rerun)
- Doesn’t overload Akeneo
- Updates Keystone `ProductRef` for ~10 locales

---

# 7) Updated RAID items (because of DB + polling)

## New Risks

- **R-DB1:** Shared DB exposed to Vercel runtime increases attack surface → mitigate with strict roles, TLS, IP allowlist.
- **R-AKE1:** Polling may miss changes if cursor logic fails → mitigate with nightly reconciliation.
- **R-AKE2:** Poll frequency vs Akeneo performance needs balancing.

## Dependencies

- Network/Security approval for:
    - on‑prem outbound to managed Postgres
    - Vercel outbound to managed Postgres
    - Wiredoor endpoints and DNS/TLS

---

# 8) Next actions (what I will produce next)

With your answers, the next useful artifacts are:

1. **A full prioritized MVP backlog** (Epics → Stories → Tasks) ordered by business value
2. A dedicated **Obsidian Kanban board per sprint** (Sprint 0 and Sprint 1)
3. A **Keystone schema blueprint** (Lists + fields + locale strategy) aligned to ~10 locales
4. A detailed **Akeneo polling runbook** (frequency, cursor model, reconciliation, error handling)

---

## Two micro-clarifications (no more big questions)

To finalize without ambiguity:

1. When you said “Prisma”, do you mean: **“We will use Prisma (via Keystone) with a Postgres DB”**? (That’s the standard and aligns with Keystone’s DB config.) [[wiredoor.net]](https://www.wiredoor.net/documentation), [[elsner.com]](https://www.elsner.com/medusajs-open-source-headless-ecommerce/)
2. Do you prefer the managed Postgres to be **vendor-neutral** (e.g., Neon/RDS) or **Vercel Postgres** specifically?

Reply with those two and I’ll generate **Sprint 0 and Sprint 1 Kanban boards fully expanded** (tasks, owners, DoD checks) ready to paste into Obsidian.