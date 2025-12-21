---

kanban-plugin: board

---

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





%% kanban:settings
```
{"kanban-plugin":"board","list-collapse":[false]}
```
%%