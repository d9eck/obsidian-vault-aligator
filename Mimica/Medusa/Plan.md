## 12. Phase-by-phase delivery strategy

  

This section converts your implementation roadmap into release-safe slices.

  

### Phase 1 — Foundation

  

**Scope**

- Bootstrap Medusa v2 project

- PostgreSQL and Redis wiring

- local developer environment

- IaC baseline

- CI pipeline baseline

- runtime topology for server + worker

- base observability

- backup/restore baseline

  

**Required outputs**

- `medusa-config.ts` environment-driven configuration

- local Docker-based developer stack

- IaC for non-local infrastructure

- deployment descriptors

- controlled migration step

- health and readiness checks

- smoke tests

  

**Promotion gate**

- Dev deployment is automated

- Staging can run the same artifact and migrations

- Logs/metrics exist before Phase 2 begins

  

### Phase 2 — Channels & Regions

  

**Scope**

- sync/integration layer for markets, currencies, countries, sales channels, stock locations

- encode multi-region selling model

  

**Required outputs**

- versioned contracts

- idempotent upsert workflows

- stable external IDs

- reconciliation job and report

- environment-specific provider sandbox configuration

  

**Promotion gate**

- full re-sync is replay-safe

- region/channel/location data can be rebuilt from source without manual DB edits

  

### Phase 3 — Catalog Projection

  

**Scope**

- Directus → Medusa product family / variant / base price projection

- SKU-centered identity across systems

  

**Required outputs**

- projection workflows

- contract tests using fixture payloads

- checksum/hash or version tracking per inbound item

- deterministic upsert behavior

- parity report between source and projection

  

**Promotion gate**

- repeated imports do not duplicate or corrupt data

- checkout-relevant projection is available in Medusa without Directus at runtime

  

### Phase 4 — Pricing & Promotions

  

**Scope**

- Medusa price lists per market/currency

- Medusa-owned discounts/promotions

  

**Required outputs**

- pricing config as code

- feature flags or controlled rollout for promotion logic changes

- test matrix by market/currency

- smoke tests for computed prices

  

**Promotion gate**

- staging validates representative regional pricing scenarios

- promotion rules do not require production-only manual configuration

  

### Phase 5 — Cart & Checkout

  

**Scope**

- cart → checkout → order flow

- explicit market/currency/country context

- PSP and shipping/fulfillment integration

  

**Required outputs**

- end-to-end transactional workflows

- idempotent webhook handling

- concurrency/locking around critical payment and inventory paths

- operational kill switches / safe degradation paths

- synthetic checkout tests

  

**Promotion gate**

- successful staging checkout with sandbox providers

- payment callback and fulfillment callback replay tested

- business-critical alerts enabled

  

### Phase 6 — Customer Accounts

  

**Scope**

- registration

- login

- address management

- order history

  

**Required outputs**

- auth/security review

- data privacy review

- customer workflow tests

- account-related observability

  

**Promotion gate**

- regression suite covers registration, login, address CRUD, order retrieval

- secrets and auth config fully externalized

  

### Phase 7 — Storefront Integration

  

**Scope**

- Next.js storefront consumes Medusa for prices, inventory, cart, checkout

- Directus remains content/catalog read owner

  

**Required outputs**

- clear API contracts

- CORS/auth config per environment

- rate limiting / abuse considerations

- synthetic user journeys

  

**Promotion gate**

- staging storefront works against staging Medusa with no environment-specific code changes

- release notes document any API contract additions or changes

  

### Phase 8 — Ops & Hardening

  

**Scope**

- inventory operations

- fulfillment workflows

- webhook reliability

- observability

- resilience testing

- testing without Directus runtime dependency

  

**Required outputs**

- runbooks

- alert tuning

- replay tooling

- chaos/failure drills

- load/performance checks on critical paths

  

**Promotion gate**

- checkout resilience demonstrated when Directus is unavailable

- failed jobs can be replayed safely

- on-call team has release and recovery documentation

  

---