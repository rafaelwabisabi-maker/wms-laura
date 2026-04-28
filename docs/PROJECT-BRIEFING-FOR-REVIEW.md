# WMS Laura — Project Briefing for External Code Review

> **Audience:** OpenCode (or any external reviewer)
> **Author:** Rafael (project owner) + Claude (implementation)
> **Date:** 2026-04-28
> **Status:** LIVE in production at https://estoquefast.log.br
> **Repository:** Local — `/Users/apple/Desktop/PROJECTS/wms-laura/`
> **Goal of this review:** suggest architectural, code-quality, security, and product improvements across the entire project.

## 1. What this project is

A small-but-real **Warehouse Management System** for a Brazilian pet-shop supply e-commerce business called **Fast Action**. Built by Rafael (uncle) for **Laura** (niece, business owner) and Leo (operations lead). Live, processing real inventory.

**The problem it solves:** Fast Action was running 1.500–2.000 orders/day across Shopee/ML/TikTok using paper notes + Up Seller (third-party label printer) + manual stock tracking. They needed a system that:
- Maps the physical warehouse (3 ruas of porta-paletes + floor zones, 353 positions)
- Tracks stock per position
- Auto-deducts on sale (CSV import today, Shopee API integration in branch)
- Logs every movement (audit trail)
- Works on mobile (warehouse floor)

**Commercial framing:** internal family project today, but documented as **SOL Business #5** in [SEED-CONCEPT.md](../SEED-CONCEPT.md) — potential B2B SaaS for SMB Brazilian e-commerce sellers (€300-500/mo target).

## 2. Stack

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js 16.1.6** (App Router) | Server components, single repo, fast iteration |
| Language | **TypeScript** | Type safety on a small team |
| Database | **SQLite** via `better-sqlite3` | Single-file DB on VPS volume, fast, no separate service |
| ORM | **Prisma** | Schema-first, generates client into `src/generated/prisma` |
| Auth | **NextAuth v5** (Edge-safe split) | `auth.config.ts` (Edge) + `auth.ts` (full Node) |
| UI | **shadcn/ui + Tailwind CSS** | Owned components, no npm bloat |
| Hosting | **Docker on VPS Semeadora** (`159.69.202.5`) | Hetzner Cloud, port 3001 mapped to container 3000 |
| Domain | `estoquefast.log.br` (Cloudflare → VPS) | Tier-2 DNS provider |
| Deploy | Manual: `scp` + `docker build` + `docker run` | No CI/CD yet — `railway-init.js` runs migrations on startup |

**Runtime model:** single Docker container per project. SQLite DB in `/opt/wms-data/wms.db` (volume-mounted). Backups via Guardian system on Mac (`agent-infra/guardian/`).

## 3. Repository layout

```
wms-laura/
├── CLAUDE.md                          # Project rules + operational reality
├── README.md
├── DESIGN.md                          # Visual design guidelines
├── SEED-CONCEPT.md                    # Productization plan (B2B SaaS angle)
├── docs/
│   ├── PROJECT-BRIEFING-FOR-REVIEW.md  ← THIS FILE
│   ├── shopee-integration.md           # Shopee API integration plan
│   ├── proposta-integracao-shopee-*.pdf
│   ├── analise-viabilidade.md          # Initial feasibility study
│   └── questionario.md                 # Initial discovery questions
├── intel/                              # Client interview intel (NEW — 2026-04-28)
│   ├── INDEX.md
│   ├── 2026-04-28-leo-operations-interview.md     # 7 questions answered
│   └── 2026-04-28-leo-followup-answers.md         # 6 follow-ups + Up Seller research
├── app/                                # V1 (legacy, archived)
│   └── ...                             # Order management, drivers, SMS — not in active use
└── app-v2/                             # ← V2 (PRODUCTION)
    ├── prisma/
    │   ├── schema.prisma               # All models
    │   └── migrations/
    │       └── 20260425150000_add_shopee_integration/migration.sql
    ├── scripts/
    │   ├── railway-init.js             # Startup script: migrations + seed users (idempotent)
    │   └── seed-users.js
    ├── src/
    │   ├── app/                        # Next.js App Router
    │   │   ├── (authenticated)/        # Auth-required pages
    │   │   │   ├── mapa/page.tsx       # Warehouse map (main UI)
    │   │   │   ├── dashboard/
    │   │   │   ├── recebimento/        # Receive goods
    │   │   │   ├── separacao/          # Picking
    │   │   │   ├── transferencia/      # Move stock
    │   │   │   ├── ajuste/             # Stock adjust
    │   │   │   ├── contagem/           # Inventory count
    │   │   │   ├── scanner/            # Camera barcode scanner
    │   │   │   ├── produtos/           # Product CRUD
    │   │   │   ├── baixa-vendas/       # CSV sales import
    │   │   │   ├── historico/          # Movement history
    │   │   │   ├── manual/             # User manual
    │   │   │   └── configuracoes/      # Admin config
    │   │   ├── login/page.tsx
    │   │   └── api/
    │   │       ├── auth/[...nextauth]/route.ts
    │   │       ├── mapa/route.ts                  # GET full warehouse
    │   │       ├── estoque/
    │   │       │   ├── liberar-vazios/route.ts    # POST: reset phantom-occupied
    │   │       │   └── movimentar/route.ts
    │   │       ├── stocks/[id]/liberar/route.ts
    │   │       ├── produtos/{,bulk,import,[id]}/route.ts
    │   │       ├── vendas/{importar,preview,historico}/route.ts
    │   │       ├── historico/route.ts
    │   │       ├── files/[filename]/route.ts
    │   │       ├── upload/route.ts
    │   │       └── shopee/                         # Shopee Open Platform v2 boilerplate
    │   │           ├── auth/route.ts               # OAuth init (CSRF state cookie)
    │   │           ├── callback/route.ts           # OAuth callback (timing-safe state)
    │   │           ├── webhook/route.ts            # HMAC-verified webhook intake
    │   │           ├── sync/route.ts               # Process pending events
    │   │           ├── sync/[orderSn]/route.ts     # Re-sync 1 order
    │   │           └── test/route.ts               # MOCK webhook (dev only)
    │   ├── components/
    │   │   ├── ui/                                 # shadcn primitives
    │   │   ├── layout/app-shell.tsx
    │   │   ├── products/product-search.tsx
    │   │   └── warehouse/
    │   │       ├── warehouse-map.tsx               # 4-zone map render (UPDATED today)
    │   │       └── liberar-vazios-button.tsx
    │   ├── lib/
    │   │   ├── auth.ts / auth.config.ts            # NextAuth split
    │   │   ├── db.ts                               # Prisma singleton
    │   │   ├── constants.ts                        # LOCATION_*, MOVEMENT_*, etc.
    │   │   ├── utils.ts / validations.ts
    │   │   ├── warehouse.ts
    │   │   ├── sale-import.ts                      # CSV deduction logic
    │   │   └── shopee/
    │   │       ├── auth.ts                         # OAuth + token refresh
    │   │       ├── client.ts                       # API client with retry/backoff
    │   │       ├── config.ts                       # Env loading + validation
    │   │       ├── crypto.ts                       # AES-256-GCM, HMAC-SHA256, OAuth state
    │   │       ├── orders.ts                       # get_order_detail, decideOrderAction
    │   │       ├── webhook.ts                      # Signature verify + ingest
    │   │       ├── processor.ts                    # Order → DEDUCT/RESTORE/NOOP transaction
    │   │       └── mock.ts                         # Dev mock generator
    │   └── generated/prisma/                       # Auto-generated, DO NOT EDIT
    └── package.json
```

**Code volume:** 78 TypeScript files in `src/` (excluding generated Prisma).

## 4. Database schema (full)

Single SQLite DB. All tables. See `prisma/schema.prisma` (267 lines) for source of truth.

### Core tables (LIVE in production):

```prisma
WarehouseConfig  // singleton: ruaCount=3, moduloCount=17, nivelCount=3, posicaoCount=2
Location         // 353 rows: code (unique), rua, side, modulo, nivel, posicao, type, status
Product          // sku (unique), barcode, name, unit, minStock, unitsPerPack, active
Stock            // unique(productId, locationId), quantity, lastCountAt
StockMovement    // immutable audit: type (INBOUND/OUTBOUND/TRANSFER/SALE/RETURN/...), reference, userId
User             // email (unique), pin (unique 4-digit), role (ADMIN/OPERATOR), passwordHash
SaleImport       // CSV import batch log
```

### Shopee integration (boilerplate, branch `feature/shopee-integration`):

```prisma
ShopeeAccount       // OAuth tokens encrypted (AES-256-GCM), 1 per shopId
ShopeeWebhookEvent  // Raw webhook log; shopId is soft-link (no FK) for audit-of-junk
ShopeeOrderSync     // Idempotency: unique(shopId, orderSn)
ShopeeSkuMapping    // Shopee item/model → local Product (auto-created or manual)
```

### Location.type enum (NEW today):

| type | Count | Code format | Where |
|---|---|---|---|
| `STORAGE` | 222 | `1B2E` (Rua-Coluna-Nível-E/D) | Porta-paletes (3 ruas) |
| `CENTRO` | 54 | `C1P1` (Centro-Posição) | Corridors between ruas |
| `CHAO_RUA` | 64 | `R1CH1` (Rua-Chão) | Floor in front of each rua |
| `CHAO_PAREDE` | 13 | `CHP1` | Floor against wall |

The Location schema fields (`rua`, `side`, `modulo`, `nivel`, `posicao`) are **mandatory ints** (defaults 0/1). For non-rack types, they're set to placeholder values (rua=0, side="-", etc.) — **this is technical debt** worth flagging.

## 5. What was built TODAY (session 2026-04-28)

In chronological order:

1. **Hardened Shopee integration** (commit `5b7886f`):
   - Added explicit 4xx throw in API client (was silently passing as success)
   - OAuth CSRF: `state` cookie + `timingSafeEqual` comparison
   - 15s `fetchWithTimeout` on token exchange/refresh
   - `ShopeeWebhookEvent.shopId` soft-link (removed FK to allow audit of unknown-shop webhooks)
   - Mock test handles non-numeric shopId (auto-creates placeholder account)
   - Critical fix: `railway-init.js` now creates Shopee tables (was missing — would never deploy in prod)

2. **Liberar Posições Vazias feature** (commit `b27bbf4`):
   - `POST /api/estoque/liberar-vazios` — resets `Location.status` to AVAILABLE for locations with zero Stock rows
   - Button in Configurações > Manutenção (ADMIN only)
   - Triggered by Leo's complaint: 31% phantom occupation from dev seed data
   - Resolution: 66 phantom locations cleared on prod

3. **Warehouse map refactor for Leo's scheme** (commit `2d65b7a`):
   - Migrated 216 generic locations (`AA 01.01.01`) → 222 real positions (`1B2E`)
   - Variable column count per Rua (Rua 1: 17 cols, Rua 2/3: 10 cols)
   - Sides E/D instead of A/B
   - Column letter headers (A...Q) instead of "Mód 1, 2, 3"

4. **Floor zones added** (commit `09d4ec3` + `bfc9edf`):
   - +131 positions across 3 new types (CENTRO, CHAO_RUA, CHAO_PAREDE)
   - `FloorZone` subcomponent with dynamic subtitles computed from data
   - Map API extended to include all 4 types

5. **Intel ingestion** (commits `d2ff5c7`, `d29cecd`, `b2ba9a6`):
   - 2 markdown docs in `intel/` capturing Leo's interview answers (operational reality, multi-channel context, Up Seller research)
   - Found Up Seller has NO public API → confirmed plan A (per-marketplace integration)

**Result:** prod DB has 353 positions across 4 zones. Image rebuilt + container restarted. `Ready in 886ms`.

## 6. Operational reality (from Leo's interview)

This is critical context — see `intel/2026-04-28-leo-operations-interview.md` and `intel/2026-04-28-leo-followup-answers.md` for full verbatim quotes.

| Dimension | Reality |
|---|---|
| **Order volume** | 1.500–2.000/day normal, 2.000+ on promo days |
| **Sales channels** | Shopee (#1), Mercado Livre (#2), TikTok Shop (#3); planned: Amazon BR, Temu |
| **Order hub** | All marketplaces flow into **Up Seller** (third-party ERP for label printing + NF-e). NO public API. |
| **Picking** | SKU-driven visual reading of label. Scanner not used per-product (only ~70% have individual barcodes). |
| **Receiving** | Edmilson receives, Lucas types **paper NF manually** into system (gargalo). |
| **High-turnover (Curva A)** | Fills entire ruas + floor positions with 1 product (rações) — needs **Bulk Fill** feature. |
| **Promo calendar** | "Day = Month" pattern (5/5, 6/6, 7/7, 11/11). Next big = **6/6** (~5 weeks from review). |
| **SKU creators** | Leo, Laura, Thiago, Henrique (4 people, NO standardization). |
| **Internal product code** | NONE. Uses marketplace SKU directly. |

## 7. Recently active branches

```
main                                 # Stable, deployed
feature/shopee-integration           # Active development branch (last 6 commits here)
```

Last 6 commits on `feature/shopee-integration`:
```
bfc9edf fix(wms): clone arrays before sort + dynamic floor zone subtitles
09d4ec3 feat(wms): map renders 4 zones (rack + centro + chao_rua + chao_parede)
2d65b7a feat(wms): refactor warehouse map for Leo's column-letter scheme (1B2E)
b27bbf4 feat(wms): liberar posições vazias — endpoint + botão em Configurações
5b7886f fix(shopee): hardening pass — 4xx, OAuth CSRF, audit, prod migration
7f68878 feat(shopee): boilerplate Shopee Open Platform v2 integration
```

## 8. Known tech debt / risks

### High priority

1. **Schema mismatch for floor zones**: `Location` has mandatory `rua, side, modulo, nivel, posicao` ints designed for racks. Floor zones use placeholders (rua=0, side="-"). A `zone` enum field would be cleaner — but migration risk on live data.

2. **Hardcoded warehouse layout in DB seed**: The 222 rack codes + 131 floor codes are inserted via ad-hoc Node scripts (`/tmp/migrate-*.js`), not via Prisma migrations or a config file. Re-applying = manual.

3. **No CI/CD**: every deploy is `scp + docker build + docker run` by hand. Easy to skip steps (happened today — first deploy attempt died silently, second worked).

4. **`railway-init.js` is a custom migration runner**: predates Prisma migrations. Idempotent inline `db.exec()` of CREATE TABLE IF NOT EXISTS. Works but bypasses Prisma's tracking.

5. **Shopee branch not yet merged**: live prod runs `main` (no Shopee yet); the branch has 6 commits including the warehouse rebuild. Needs merge planning — Shopee features are gated by env vars (`SHOPEE_PARTNER_ID`) so they're inert without config.

6. **Single SQLite file**: at 1500-2000 orders/day for many months, will grow large. SQLite handles it but no rotation/archival strategy.

### Medium priority

7. **No tests**: zero unit/integration/e2e tests in `src/`. Manual QA only.

8. **Auth secrets in `.env` on VPS**: not in any vault. `.env.local` on Mac is git-ignored but NextAuth secret etc. would be lost if VPS dies without backup.

9. **`generated/prisma` is committed**: 200+ TypeScript files committed to repo. Bloat, regenerate-friendly but creates noise in diffs.

10. **No rate limiting** on any endpoint. Mapa endpoint dumps all 353 locations + stocks + products in a single payload.

11. **Mobile UX not measured**: the map is responsive but no Lighthouse/perf budget. Pickers use phones on the warehouse floor.

12. **No structured logging**: `console.log` everywhere. No log aggregation.

### Low priority / observations

13. `RUA_LETTER` constant was used for "AA/BB/CC" mapping then removed in the rebuild. Some legacy comments may reference the old scheme.

14. Portuguese strings hardcoded throughout — no i18n. Fine for current single-tenant use, problematic for SaaS productization.

15. `WarehouseConfig` table is mostly unused now that the layout is variable per rua. Only `ruaCount` and `nivelCount` are read.

## 9. Backlog (prioritized from Leo's intel)

### P0 — needs to be built next
1. **Bulk Fill** — 1 product × N positions in a single transaction. Leo's #1 ask. Use case: truck of rações fills entire Centro 1 (18 pos) + R1CH1-R1CH18 (18 pos) at once. ~4-6h of work.

### P1 — within 2 weeks
2. **Generic `MarketplaceSkuMapping`** — refactor `ShopeeSkuMapping` to support ML/TikTok/Amazon (preparation, ~2h).
3. **Bulk SKU mapping CSV import** — UI to bulk-load Shopee/marketplace SKU → Product mappings (~2h).
4. **"Show empty positions only" filter** on map (~30min).
5. **Volume dashboard** — orders today vs yesterday vs 7-day avg (~3h).
6. **Confirm with Leo**: does Up Seller currently sync stock? (5-min message — risk that Up Seller may overwrite WMS stock changes via its own marketplace sync).

### P2 — when ready
7. **Mercado Livre webhook integration** (clone Shopee pattern, ~1-2 days).
8. **NF-e XML import** (Lucas types paper notas manually — Brazil has SEFAZ XML standard — ~8-12h).
9. **`Product.turnoverClass` field** (A/B/C) + visual marker on map for Curva A.
10. **TikTok Shop integration** (lower volume, lower priority).

### P3 — productization (SOL Business #5)
11. Rebrand from "Laura" to product name.
12. Self-serve onboarding wizard.
13. Multi-tenant data isolation.
14. Pricing page + Stripe.

## 10. Specific review focus areas

When OpenCode reviews, please prioritize feedback on:

### A. Data model
- Is the Location schema (rua/side/modulo/nivel/posicao + type) fit-for-purpose given 4 zone types? Should we add a `zone` field? Would that break anything?
- Stock + StockMovement design: is it auditable enough for a future SOC2-style review? Are there edge cases where movements could be lost?
- Idempotency on `ShopeeOrderSync` — is `unique(shopId, orderSn)` sufficient? What about partial fulfillment?

### B. Shopee integration (`src/lib/shopee/`)
- HMAC verification correctness (`webhook.ts`)
- AES-256-GCM token encryption (`crypto.ts`)
- OAuth state CSRF protection (`auth.ts`)
- Retry/backoff logic on 429/5xx (`client.ts`)
- Race condition risk in `processor.ts` (concurrent webhooks for same product)
- Should this be merged to main? What's the risk?

### C. Warehouse map UI (`src/components/warehouse/warehouse-map.tsx`)
- Performance with 353 cells rendering on every state change
- Mobile responsiveness (Rua 1 is 17 columns wide — does it scroll cleanly on phones?)
- Accessibility (color-only status indication, no aria labels)
- React anti-patterns (we just fixed a sort-mutation bug)

### D. API surface
- Single big `GET /api/mapa` returns everything. Is this acceptable at 353 rows? At 1000+?
- Auth check pattern (`auth()` at top of every route handler) — could be middleware
- Error responses are inconsistent (sometimes JSON, sometimes redirect, sometimes HTML)

### E. Operational concerns
- Deploy story: how to make this safer / repeatable? GitHub Actions? Coolify? Dokploy?
- Database backups: Guardian runs on Mac, not VPS. If VPS dies, data is on Mac sync but not real-time.
- Monitoring: zero. No alerts if container crashes, if disk fills, if errors spike.

### F. Productization (if you want to opine)
- What's the minimum viable refactor to make this multi-tenant?
- Would you keep SQLite or migrate to Postgres for SaaS?
- Naming + positioning: any thoughts on the `LogiFlow` / `KlarLager` / `EstoqueFast` names in `SEED-CONCEPT.md`?

## 11. How to navigate the code

If you're starting fresh, recommended reading order:

1. `app-v2/prisma/schema.prisma` (understand the data model — 5 min)
2. `app-v2/src/app/(authenticated)/mapa/page.tsx` + `app-v2/src/components/warehouse/warehouse-map.tsx` (the main UI)
3. `app-v2/src/app/api/mapa/route.ts` (the corresponding API)
4. `app-v2/src/lib/sale-import.ts` (the CSV deduction logic — well-tested, in production for months)
5. `app-v2/src/lib/shopee/processor.ts` (the new Shopee deduction — same pattern adapted for webhooks)
6. `app-v2/src/lib/shopee/crypto.ts` + `auth.ts` (security primitives)
7. `app-v2/scripts/railway-init.js` (the custom migration runner)
8. `intel/*.md` (the operational reality)

## 12. Constraints to respect

- **Zero downtime** on prod. Whatever you suggest must be migrate-able with the system live (Leo is using it daily).
- **Portuguese (BR)** UI for end users (Laura, Leo, Edmilson, Lucas, etc.). Code/docs in English OK.
- **No commercial WMS pattern** copying — keep it simple. Laura's not running an Amazon FC.
- **Mobile-first**: pickers use phones on the floor, not desktops.
- **One developer (Rafael) maintaining** — anything that adds operational complexity needs to pay for itself.

## 13. Where to ask questions

- Code: this repo, this file, `intel/`
- Operations: Leo via WhatsApp (Rafael relays)
- Stack/infrastructure: Rafael directly
- Productization: `SEED-CONCEPT.md`

---

**Hand-off note for OpenCode:** the ask is *"sugerir melhorias"* — be opinionated. Tell us what we'd regret in 6 months. Tell us what's over-engineered. Tell us what's under-engineered. Prioritize concretely (P0/P1/P2). If you find a security issue in the Shopee module, flag it as P0 — that branch isn't merged yet, easy to fix now.
