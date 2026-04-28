---
status: WIRED
type: code_review_synthesis
project: wms-laura
date: 2026-04-28
sources:
  - ce-security-sentinel (Shopee security)
  - ce-performance-oracle (mapa API + render)
  - ce-data-integrity-guardian (schema + migrations)
  - ce-architecture-strategist (overall architecture)
  - ce-testing-reviewer (test coverage)
wires_into:
  - wms-laura/CLAUDE.md (action items in backlog)
  - wms-laura/docs/PROJECT-BRIEFING-FOR-REVIEW.md (closes the review loop)
  - intel/INDEX.md
priority: HIGH
tags: [audit, shopee, security, performance, schema, tests, deployment]
---

# WMS Laura — 5-Agent Adversarial Review Synthesis

> **Audit performed:** 2026-04-28 by 5 parallel specialized critics on `feature/shopee-integration` branch + main prod state.
> **Bottom line:** System is solid for current load. Three independent agents converged on the **same 5 risks** that fire on either (a) merging Shopee branch, or (b) the 6/6 promo (~5 weeks away). Fix list is concrete, ~2 days of work to safely ship.

## 0. Cross-agent convergence — fix these first

When 3+ specialists flag the same thing, it's not opinion, it's signal. Five issues converged:

| # | Risk | Flagged by | Fires when |
|---|---|---|---|
| **A** | **Concurrent webhook DEDUCT race** | Security + Performance + Data Integrity | 6/6 promo + Shopee live |
| **B** | **Idempotency on Shopee orderSn is unsafe for status transitions** | Security + Data Integrity + Testing | Shopee live + retries/partial fulfillment |
| **C** | **Stock can change without StockMovement (audit gap)** | Data Integrity + Architecture | Already happening (phantom occupation) |
| **D** | **Webhook = synchronous DB write under HTTP timeout** | Security + Performance + Data Integrity | 6/6 burst (100 events/sec) |
| **E** | **Deploy is fragile and silently failed today** | Architecture + Performance | Any future deploy |

These five are the **must-fix before merging Shopee branch** list. Everything else is improvement, not blocker.

## 1. P0 — Block before Shopee merge OR before 6/6 (whichever comes first)

### Security & data integrity

**P0-1. `_test_items` backdoor in production code path**
- File: [src/lib/shopee/webhook.ts:157-163](../app-v2/src/lib/shopee/webhook.ts)
- Risk: signed webhook with `_test_items` field skips Shopee API, deducts arbitrary stock
- Fix: gate on `NODE_ENV !== 'production' && cfg.env === 'sandbox'` (3 lines)
- Effort: 5min

**P0-2. Webhook timestamp window missing → unbounded replay**
- Files: [src/lib/shopee/webhook.ts](../app-v2/src/lib/shopee/webhook.ts), [src/lib/shopee/crypto.ts](../app-v2/src/lib/shopee/crypto.ts)
- Risk: captured signed webhook replayable forever; idempotency saves SAME order, not future codes
- Fix: reject events where `payload.timestamp` < now-5min OR > now+1min
- Effort: 10min

**P0-3. Unbounded body size + IP-less webhook = DB DoS**
- File: [src/lib/shopee/webhook.ts:67-76](../app-v2/src/lib/shopee/webhook.ts)
- Risk: anyone on internet can flood `ShopeeWebhookEvent` with 1MB payloads → SQLite explodes, disk fills
- Fix: cap `request.text()` at 64KB. Drop signature-invalid events instead of persisting (or persist metadata-only with TTL)
- Effort: 10min

**P0-4. Host-header trusted for HMAC URL → defense-in-depth gap**
- File: [src/app/api/shopee/webhook/route.ts:65-73](../app-v2/src/app/api/shopee/webhook/route.ts)
- Risk: if `partner_key` ever leaks, attacker can use `Host: evil.com` header to forge any URL
- Fix: require `SHOPEE_WEBHOOK_PUBLIC_URL` env var in prod, never trust headers
- Effort: 10min

**P0-5. RESTORE has no idempotency guard → replay = 2× stock**
- File: [src/lib/shopee/processor.ts:355-419](../app-v2/src/lib/shopee/processor.ts)
- Risk: action decision is OUTSIDE transaction; 2 concurrent CANCEL webhooks both see `previousAction=DEDUCT`, both restore = double stock added
- Fix: move `decideOrderAction` INSIDE transaction + check existence of `SHOPEE_CANCEL:` movement before re-restoring
- Effort: 30min

**P0-6. Concurrent DEDUCT race (Issue A above)**
- File: [src/lib/shopee/processor.ts:298-352](../app-v2/src/lib/shopee/processor.ts)
- Risk: two webhooks for same product both pass `total >= quantity` check, both deduct → audit ≠ stock, silent oversell
- Fix: replace `findUnique → check → update` pattern with atomic conditional updateMany:
  ```ts
  const result = await tx.stock.updateMany({
    where: { id: stock.id, quantity: { gte: deduct } },
    data: { quantity: { decrement: deduct } }
  })
  if (result.count === 0) throw new Error("race lost")
  ```
- Bonus: in-process p-limit(1) per `productId` for belt-and-suspenders
- Effort: 1h + tests

**P0-7. Webhook synchronous DB processing kills handler under burst (Issue D)**
- Files: webhook handler + processor
- Risk: 100 webhooks/sec during promo → handlers queue on SQLite single-writer → 15s Shopee timeout → endpoint disabled
- Fix: USE the worker pattern (schema already supports it via `ShopeeWebhookEvent.status=PENDING`). Webhook = signature verify + insert event + 200 OK in <50ms. Worker = pulls PENDING in batches, processes serially.
- Effort: 4-6h (the briefing hardening missed this)

**P0-8. `RESET_DATA` env var = catastrophic data loss footgun**
- File: [scripts/railway-init.js:38-51](../app-v2/scripts/railway-init.js)
- Risk: typo or copy-paste of old V1 deploy script wipes StockMovement + Stock + Product on container restart
- Fix: remove the block from prod startup script. Move to `scripts/dev-reset.js` outside Docker image
- Effort: 5min

**P0-9. Schema reproducibility broken — no baseline migration**
- Files: [prisma/schema.prisma](../app-v2/prisma/schema.prisma) + [prisma/migrations/](../app-v2/prisma/migrations/)
- Risk: core tables (Stock/Product/Location/StockMovement) exist only because someone ran `prisma db push` once and committed `dev.db` binary. Fresh deploy = fails or diverges
- Fix: `prisma migrate diff --from-empty --to-schema-datamodel ... > 00000000_baseline/migration.sql` + `prisma migrate resolve --applied baseline` on VPS. Replace `db.exec(CREATE TABLE)` in railway-init.js with `prisma migrate deploy`
- Effort: 1-2h

**P0-10. Stock can mutate without StockMovement (Issue C)**
- Files: [api/estoque/liberar-vazios/route.ts](../app-v2/src/app/api/estoque/liberar-vazios/route.ts), [shopee/processor.ts](../app-v2/src/lib/shopee/processor.ts), various route handlers
- Risk: today's "phantom 31% occupation" was caused by this exact class. Stock and audit drift silently.
- Fix:
  1. Single `applyStockDelta(tx, ...)` helper, write StockMovement first then Stock; refactor 3 callers
  2. SQLite trigger as backstop: `CREATE TRIGGER stock_audit AFTER UPDATE OF quantity ON Stock WHEN NEW.quantity != OLD.quantity ...`
  3. Add `LocationStatusLog` (or include status changes as zero-quantity movements)
- Effort: 4h

### Performance — must do before 6/6

**P0-11. CSV import in one giant transaction → wall hangs at 3000 lines**
- File: [src/app/api/vendas/importar/route.ts:110-224](../app-v2/src/app/api/vendas/importar/route.ts)
- Today's load: 5-15s wall (works). 6/6 (3000 lines): **15-30s, browser may abort, double-deduction risk**
- Fix:
  1. Drop pre-validation loop (cache product/stock lookups in Map)
  2. Chunk transaction in 200-item batches
  3. Use `createMany` for batched StockMovements
- Effort: 2-3h

**P0-12. Verify SQLite WAL mode is actually persisting**
- File: [src/lib/db.ts:14-17](../app-v2/src/lib/db.ts) + [scripts/railway-init.js](../app-v2/scripts/railway-init.js)
- Risk: WAL pragma set on a temp connection that closes — Prisma adapter's actual connection may not have WAL. Free 5-10× write throughput if missing.
- Fix: verify with `PRAGMA journal_mode;` from running app. If not set, add to railway-init.js: `db.exec("PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA busy_timeout=5000; PRAGMA wal_autocheckpoint=1000;")`
- Effort: 30min

### Operations

**P0-13. Deploy script + health endpoint (Issue E)**
- Failed silently today. Will fail again.
- Fix:
  1. `/api/health` route: DB ping + version
  2. `deploy.sh`: git push → ssh VPS → git pull → docker build → run health check pre-swap → swap containers → re-check, rollback on fail
  3. Skip CI/CD for now (one dev = bash script + curl is sufficient)
- Effort: 4-6h

**P0-14. VPS-local backups (no current safety net)**
- Guardian Mac sync = up to 6h stale. If VPS dies between syncs, lose all paper-NF entries from Lucas.
- Fix: VPS crontab `0 */2 * * * sqlite3 /opt/wms-data/wms.db ".backup /opt/wms-data/backups/wms-$(date +\%Y\%m\%d-\%H).db"` + retain 24 rolling files
- Effort: 15min

### Tests (block Shopee merge on these 3)

**P0-15. Test 1 — `crypto.test.ts`** (45min)
- AES-256-GCM round-trip + tamper detection + wrong key throws
- HMAC signature: known-vector pass, tampered fail, re-stringified body fail (catches the most likely future regression), wrong-length sig fail without throw

**P0-16. Test 2 — `processor.deductStock.test.ts`** (3h)
- Single row drained → row deleted, location AVAILABLE
- Two rows (10, 5), need 12 → 10 from biggest, 2 from smaller
- Insufficient stock → rollback, no movements, no mutations
- Boundary: exactly equal

**P0-17. Test 3 — `processor.idempotency.test.ts`** (1h)
- Same orderSn twice → NOOP second time, deducted once
- Cancel after deduct → RESTORE; second cancel → NOOP, not double-restored

## 2. P1 — Within 2 weeks

**P1-1. Centralized auth wrapper** — `withAuth(handler, {role?})` in `lib/api-handler.ts`. Removes 25× copy-paste of `auth()` checks. **2-3h**

**P1-2. Standardized error contract** — `{ ok: false, error: { code, message } }` JSON shape; combine with P1-1. **3-4h**

**P1-3. Trim `/api/mapa` payload** — `product: { select: { id, name, sku, unit } }` saves ~40%; add `Cache-Control` + ETag from MAX(updatedAt). **30min**

**P1-4. Memoize WarehouseMap** — `useMemo(grouped)`, `memo(LocationCell)`, `useCallback(handleSelect)`, `Set` for highlightIds. Drops re-render cost ~10×. **30min**

**P1-5. Schema cleanup — `Location.zone` field**
- Rename `type → zone`. Make `rua/side/modulo/nivel/posicao` nullable. Drop multi-column unique, keep `code` unique.
- Migration: SQLite needs table-rebuild. Run during Leo's lunch (low-write window). Rollback = restore from backup.
- **1-2 days** but unblocks SaaS multi-tenancy

**P1-6. Drop `Location.status`** — compute it from stocks count in `/api/mapa`. Removes phantom-occupation bug class entirely. Combine with P1-5. **1h**

**P1-7. Per-line-item Shopee idempotency** — `ShopeeOrderItem` table for partial fulfillment + multi-shipment Shopee orders. **4-6h**

**P1-8. Add CHECK constraint `quantity >= 0`** on Stock — defense in depth. Migration. **30min**

**P1-9. Hash `User.pin`** — currently 4-digit cleartext, sequential 1001-1006, in CLAUDE.md. Bcrypt + rate limit on PIN login. **2h**

**P1-10. Atomic stock updates everywhere** — replace `findUnique → check → update` with conditional `updateMany` in [api/estoque/movimentar](../app-v2/src/app/api/estoque/movimentar/route.ts). **1h**

**P1-11. Token encryption key versioning** — `keyVersion` column on `ShopeeAccount`; enables future rotation without bricking tokens. **30min**

**P1-12. OAuth state bound to user** — sign as HMAC of `userId|nonce|expiry` so multi-ADMIN orgs don't get cross-account auth confusion. **1h**

**P1-13. Move logic to `lib/services/`** — `productService.ts`, `warehouseService.ts`, `stockService.ts`. Routes become thin HTTP adapters. Refactor one domain as template (Products), propagate opportunistically. **4h template + 1h/domain**

**P1-14. Tier-2 tests** — `decideOrderAction` matrix (30min), `parseSaleRows` grouping (45min), `liberar-vazios` filter (30min), SKU resolution fallback chain (1h). **~3h**

## 3. P2 — Worth doing before SaaS, leave alone otherwise

| # | Item | Effort |
|---|---|---|
| P2-1 | `LocationStatusLog` table for status mutations | 2h |
| P2-2 | StockMovement archival (>90d → separate file) | 4h |
| P2-3 | `next/dynamic` import for `xlsx` lib in CSV page | 30min |
| P2-4 | SWR/React Query on map page (instant repaint UX) | 2h |
| P2-5 | `Stock.@@index([productId, quantity])` for fill-rua | 15min migration |
| P2-6 | Dead `WarehouseConfig` table — delete or document deprecated | 30min |
| P2-7 | Rename `railway-init.js` → `scripts/init.js` | 15min |
| P2-8 | Archive `app/` V1 (move to `archive/app-v1/`) | 30min |
| P2-9 | Multi-tenant prep: add nullable `tenantId` to all tables now (no enforcement yet) | 30min |
| P2-10 | LGPD/GDPR: document "WMS does not persist buyer PII" in schema comments | 5min |
| P2-11 | Retention cron for `ShopeeWebhookEvent.signatureValid=false` rows (7d TTL) | 30min |
| P2-12 | Rate limit `/api/shopee/sync` POST (ADMIN can spam) | 30min |
| P2-13 | Mock test endpoint signs with fixed `MOCK_PARTNER_KEY` instead of prod key | 15min |

## 4. Things that are RIGHT — don't change

Validated by multiple agents as correct:
- AES-256-GCM with random 12-byte IV per encryption — no nonce reuse
- Timing-safe equal for state and signature
- Body-as-string preserved before signature check (critical, correct)
- `expire_in` honored, refresh on 5-min window
- Idempotency via `ShopeeOrderSync` unique constraint (caveat: needs P1-7 for partial fulfillment)
- 4xx-no-retry policy (correct anti-ban posture)
- ADMIN gate on `/auth`, `/callback`, `/sync`, `/test`
- Encryption key validation at config load (length + base64)
- Largest-stock-first deduction strategy (sensible for small warehouse)
- shadcn + NextAuth Edge-split architecture choices
- The whole `lib/shopee/` module structure (referenced as **the model** for the rest of the codebase)
- Generated Prisma client gitignored (briefing claim was WRONG — it IS gitignored, verified)
- `(authenticated)/` route group pattern
- `better-sqlite3` + Prisma at this scale

## 5. The agent that disagreed

**Testing reviewer** said skip the concurrent-webhook race test because "SQLite single-writer prevents the race." **Data Integrity reviewer** disagreed: SQLite WAL gives snapshot reads, not row locks; second writer to commit can succeed with stale read view; in-process p-limit(1) needed.

**Resolution:** Data Integrity reviewer is correct for the **lost-update** flavor of race (read stale `total`, decide deduct, both write succeed). Testing reviewer is correct that SQLite serializes the **write phase**. The fix (P0-6: atomic conditional updateMany) sidesteps the disagreement entirely — `updateMany` with `WHERE quantity >= deduct` makes the read+check+update atomic in one SQL statement, no race possible at any DB engine.

**Bottom line:** P0-6 is non-negotiable. Test for it (regression guard).

## 6. Recommended execution plan

### Sprint 1 — "Don't break in 5 weeks" (this week, ~2 days)

Day 1 (P0 mechanical fixes, low-risk):
- [ ] P0-1 `_test_items` gate (5min)
- [ ] P0-2 timestamp window (10min)
- [ ] P0-3 body size cap (10min)
- [ ] P0-4 require `SHOPEE_WEBHOOK_PUBLIC_URL` (10min)
- [ ] P0-8 remove `RESET_DATA` from prod startup (5min)
- [ ] P0-12 verify + fix WAL mode (30min)
- [ ] P0-14 VPS-local sqlite backup cron (15min)
- [ ] P0-13 `/api/health` endpoint + `deploy.sh` (4h)

Day 2 (P0 correctness fixes):
- [ ] P0-6 atomic deduct (1h + test)
- [ ] P0-5 RESTORE idempotency (30min)
- [ ] P0-15 crypto.test.ts (45min)
- [ ] P0-16 deductStock.test.ts (3h)
- [ ] P0-17 idempotency.test.ts (1h)

### Sprint 2 — "Ship Shopee safely" (next week)
- [ ] P0-7 webhook → worker async pattern (4-6h) **← this is the big one**
- [ ] P0-9 baseline migration (1-2h)
- [ ] P0-10 `applyStockDelta` helper + audit trigger (4h)
- [ ] P0-11 CSV chunking (2-3h)
- [ ] P1-7 ShopeeOrderItem (per-line idempotency) (4-6h)
- [ ] **GO/NO-GO call before merging Shopee branch**

### Sprint 3 — "Quality" (week 3)
- [ ] P1-1, P1-2 auth wrapper + error contract (5-7h combined)
- [ ] P1-3, P1-4 mapa perf (1h)
- [ ] P1-8, P1-10 schema CHECK + atomic updates (1.5h)
- [ ] P1-9 hash PINs (2h)
- [ ] P1-14 tier-2 tests (3h)

### Sprint 4 — "SaaS-ready" (when first paying tenant signed)
- [ ] P1-5, P1-6 Location.zone + drop status (2 days)
- [ ] P1-13 services layer (rolling)
- [ ] P2-9 nullable tenantId (30min)

## 7. Files audited (38 source files across all 5 reviews)

Frontend / map:
- `src/components/warehouse/warehouse-map.tsx`
- `src/app/(authenticated)/mapa/page.tsx`
- `src/app/(authenticated)/configuracoes/page.tsx`

API surface:
- `src/app/api/mapa/route.ts`
- `src/app/api/estoque/{liberar-vazios,movimentar}/route.ts`
- `src/app/api/produtos/{,bulk,[id]}/route.ts`
- `src/app/api/vendas/{importar,preview,historico}/route.ts`
- `src/app/api/historico/route.ts`
- `src/app/api/stocks/[id]/liberar/route.ts`
- `src/app/api/shopee/{auth,callback,webhook,sync,test}/route.ts`

Library code:
- `src/lib/{auth,db,constants,sale-import,warehouse}.ts`
- `src/lib/shopee/{auth,client,config,crypto,orders,webhook,processor,mock}.ts`

Schema + migrations:
- `prisma/schema.prisma`
- `prisma/migrations/20260425150000_add_shopee_integration/migration.sql`
- `scripts/{railway-init,seed-users}.js`

Infrastructure:
- `Dockerfile`
- `src/middleware.ts`

## 8. What we'd regret in 6 months without these fixes

The agents converged on a single sentence that captures the regret:
> **"Stock and audit log will silently disagree, you won't notice, then a customer complains about a shipment they never received and you can't reconstruct what happened."**

That's the failure mode P0-6 (race), P0-10 (audit gap), P0-1 (test backdoor), and P0-5 (restore idempotency) all enable in different ways. They're independent paths to the same disaster. Plug all four.
