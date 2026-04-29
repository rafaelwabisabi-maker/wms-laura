---
status: WIRED
type: phased_execution_plan
project: wms-laura
date: 2026-04-29
sources:
  - intel/2026-04-28-code-review-synthesis.md (5-agent review)
  - Mythos (strategic product lens — anti-over-engineering)
  - Codex Max (pragmatic simplicity — TRUE/PREMATURE/OVERKILL classification)
  - Original promise to Laura/Leo (briefing + operations interview)
wires_into:
  - wms-laura/CLAUDE.md
  - intel/INDEX.md
priority: HIGH (action plan)
tags: [phases, execution, anti-overengineering, shopee-deferred, bulk-fill]
---

# WMS Laura — Phased Execution Plan

> **Filosofia:** "Não criar problemas instalando coisas complicadas demais. Temos um sistema bom, NÃO QUEBRA."
> **Verdict de Mythos + Codex Max + Rafael:** dos 17 P0s do review original, **só 7 fazem sentido AGORA** (~2-6h total). Os outros 10 são reais SE a Shopee mergear ou SE o sistema dobrar de escala — não fazem mal serem deferidos.

## Princípios desta fase

1. **Sequência > volume.** Fazer 7 fixes certos é melhor que 17 fixes paralelos.
2. **Não merge Shopee sem motivo de negócio.** A 6/6 (5 semanas) é o teste de carga grátis do CSV — boring, comprovado, funciona há meses.
3. **Bulk Fill é a ÚNICA feature pedida pelo Leo** — vale priorizar. Tudo o resto saiu do Rafael.
4. **Hardening do que tá no ar > construir mais.** Laura usa hoje. Proteger primeiro.
5. **Backup automatizado é não-negociável.** Já feito (GitHub) + a fazer (VPS-local).


## FASE 0 — STABILIZE (esta semana, ~2-3h)

> **Quando rodar:** já. **Quando termina:** sistema seguro, sem footguns, com backup local + remoto. Pronto para 6/6 sem stress.

| # | Item | Esforço | Por que |
|---|---|---|---|
| 0.1 | ✅ Push pro GitHub (parent + app-v2) | DONE | Backup remoto. Reverter qualquer coisa = `git checkout`. |
| 0.2 | Remover bloco `RESET_DATA` do `railway-init.js` | 5min | Catastrophic footgun. Um typo no env wipa tudo. Só remover. |
| 0.3 | Cap de body 64KB no webhook Shopee + cap por método | 10min | Mesmo a branch não merged: protege se alguém apontar prod. Custo zero. |
| 0.4 | Gate `_test_items` em `NODE_ENV !== 'production'` | 5min | Backdoor mesmo em código não live. 3 linhas. |
| 0.5 | Verificar/setar WAL mode no `railway-init.js` | 30min | 5-10× write throughput grátis se faltar. Insurance pro 6/6. |
| 0.6 | Cron VPS-local: sqlite `.backup` a cada 2h, 24 rolling | 15min | Guardian sync = 6h stale. Local = 2h. Proteção real. |
| 0.7 | Helper `applyStockDelta(tx, ...)` LITE — 3 callers, sem trigger ainda | 1h | A causa raiz do "31% phantom" foi essa. Helper único = futuras mutações forçadas a auditar. |

**Total: ~2h 15min.** Roda em 1 sessão. Zero risco de quebrar. Tudo reversível via Git.

**Não está nessa fase (justificativa):**
- ❌ Worker async webhook → Shopee não tá live, 1500 pedidos/dia ≠ 100/sec
- ❌ `deploy.sh` + rollback orquestrado → bash de 3 linhas resolve, 1 dev, 1 deploy/semana
- ❌ Atomic conditional updateMany → SQLite single-writer + import CSV em batch (não rajada de webhooks) = race effectively zero hoje
- ❌ Baseline migration formal → "fresh deploy" é hipotético; container atual roda
- ❌ Tests → código em prod há meses sem bug; tests são insurance, não fogo


## FASE 1 — ENTREGA O QUE LEO PEDIU (semana 2, ~6-8h)

> **Quando rodar:** depois da Fase 0. **Quando termina:** Bulk Fill no ar, mapa com filtros úteis, Curva A visível.

| # | Item | Esforço | Origem |
|---|---|---|---|
| 1.1 | **Bulk Fill** — 1 produto × N posições selecionadas em batch | 4-6h | Leo Q4: "muito boa pergunta, urgente!" |
| 1.2 | Filtro "Só posições vazias" no mapa | 30min | Edmilson recebe mercadoria mais rápido |
| 1.3 | `Product.turnoverClass` (A/B/C) + marker visual no mapa | 1h | Operação reconhece curva A na hora |
| 1.4 | Liberar Vazios: trigger no mapa por rua/zona | 30min | Self-service para Leo limpar zonas específicas |

**Total: ~6-8h.** Entrega valor real para Leo (recebimento de Curva A) e Edmilson (visibilidade).

**Como construir Bulk Fill (esqueleto, para evitar over-engineering):**
- UI: modal acionado por shift-click ou botão "Modo Lote" no mapa
- Selector: produto + quantidade
- Range picker: "todas posições da Rua X" / "todo Centro N" / "ctrl+click manual"
- Backend: `POST /api/estoque/bulk-fill` — UMA transação Prisma + `createMany` para movements + loop para upserts (não invente queue/worker)
- Reference: `BULK:[uuid]` em todas movements para audit + reversão fácil
- Erro = rollback total da transação


## FASE 2 — POLISH PRÉ-PROMOÇÃO 6/6 (semana 3-4, ~4h)

> **Quando rodar:** ~2 semanas antes do 6/6. **Quando termina:** sistema medido sob carga, testado, observável.

| # | Item | Esforço | Por que |
|---|---|---|---|
| 2.1 | Trim payload `/api/mapa` (`product: { select: ... }`) + Cache-Control | 30min | Mobile dos pickers carrega instantâneo |
| 2.2 | Memoizar `WarehouseMap` (`useMemo`, `memo(LocationCell)`, Set para highlights) | 30min | 10× menos re-renders, UX melhor no celular |
| 2.3 | `/api/health` endpoint (DB ping + version) | 30min | Insurance para deploy + monitoring futuro |
| 2.4 | `deploy.sh` simples: `git pull && docker build && docker restart && curl /api/health` | 30min | Deploy seguro sem orquestração elaborada |
| 2.5 | Medir CSV import com 3000 linhas (simulação local) | 1h | Decidir se P0-11 (chunking) precisa antes do 6/6 |
| 2.6 | Se 2.5 mostrar > 20s: chunking em batches de 200 | 2h | Decisão data-driven, não preventiva |

**Total: ~4-6h dependendo de 2.5.**


## FASE 3 — DEPOIS DO 6/6 (junho, ~2-3 dias)

> **Quando rodar:** depois da promoção 6/6 com dados reais. **Quando termina:** decisão data-driven sobre Shopee.

| Item | Trigger |
|---|---|
| Análise: CSV chegou ao limite no 6/6? | Métricas reais |
| Se sim: chunking + worker pattern para CSV grandes | Da fase 2.6 |
| Se não: continuar CSV como canal principal | Sem mudança |
| Conversa com Leo: vale a pena Shopee API automation? | Ele decide |


## FASE 4 — SHOPEE GO-LIVE (só se Leo pedir, ~5 dias)

> **Quando rodar:** APENAS se houver business case claro pós-6/6.
> **Pré-requisito:** todos os fixes que o code review pediu PARA SHOPEE.

| # | Item | Esforço | Origem |
|---|---|---|---|
| 4.1 | Worker async pattern (P0-7) | 4-6h | Convergência 3 reviewers |
| 4.2 | Atomic conditional updateMany (P0-6) | 1h+test | Race-proof |
| 4.3 | Timestamp window + host header env (P0-2, P0-4) | 30min | Security defense-in-depth |
| 4.4 | RESTORE idempotency guard (P0-5) | 30min | Sem double-restore |
| 4.5 | `ShopeeOrderItem` per-line idempotency | 4-6h | Para partial fulfillment Shopee BR |
| 4.6 | Tests P0-15, P0-16, P0-17 (crypto + deduct + idempotency) | 5h | Bloqueia merge |
| 4.7 | Token encryption keyVersion column | 30min | Rotação futura |
| 4.8 | Baseline Prisma migration | 1-2h | Fresh deploy reproduzível |

**Total: ~3 dias FOCADOS.** Só depois disso, **merge controlado, atrás de feature flag, em UMA loja teste, monitorado por 2 semanas.**


## FASE 5 — EVOLUÇÃO MULTI-CANAL (Q3 2026 — só se justificar)

> **Quando rodar:** quando Shopee estiver 2 semanas estável + Leo decidir adicionar ML/TikTok.

| Item | Por que |
|---|---|
| `Location.zone` field + drop `Location.status` | Schema cleanup, multi-tenant ready |
| Generic `MarketplaceSkuMapping` | Suporte ML, TikTok, Amazon |
| Mercado Livre webhook integration | Clone do pattern Shopee |
| TikTok Shop integration | Volume baixo, lower priority |
| Amazon SP-API | Mais complexo, plans not active |


## FASE 6 — PRODUCTIZAÇÃO (só se SOL Business #5 ativar)

> **Quando rodar:** primeiro cliente pago. NÃO antes.

- Multi-tenant (`tenantId` em todas tabelas)
- i18n via `next-intl`
- Postgres migration (SQLite chega ao limite com 3+ tenants)
- Stripe + billing
- Self-serve onboarding wizard
- Hash PINs (já vira obrigatório com clientes externos)
- SOC2-grade audit trail
- LGPD/GDPR compliance docs


## OVERKILL — não fazer mesmo se sobrar tempo

| Item | Por que NÃO |
|---|---|
| GitHub Actions / CI | 1 dev, 1 deploy/semana, bash basta |
| Coolify / Dokploy | 3 mais peças móveis, sem ROI |
| Postgres migration prematura | SQLite escala bem até 5K orders/dia/tenant |
| NF-e XML import | Aguardar Lucas confirmar formato — pode ser email simples, não XML |
| OCR para nota em papel | Lucas digita 10/dia, não vale o ML |
| Hash PIN agora | 6 usuários internos, baixo risco |
| Multi-tenant antes de cliente pago | Premature abstraction. SaaS é hipotético hoje. |
| Lighthouse / perf budget | Ainda não temos métricas de mobile real, só hipóteses |
| Structured logging / log aggregation | Console.log + docker logs basta para 1 servidor |
| Monitoring (Sentry, Datadog) | $$$ + complexidade. /api/health + cron alert = 80% disso por $0 |


## Mapa rápido — onde tá tudo

```
wms-laura/
├── intel/                                    ← TODA a inteligência operacional
│   ├── INDEX.md                              ← registro
│   ├── 2026-04-28-leo-operations-interview.md ← realidade do Leo
│   ├── 2026-04-28-leo-followup-answers.md   ← 6 follow-ups + Up Seller
│   ├── 2026-04-28-code-review-synthesis.md  ← 5-agent review (17 P0s)
│   └── 2026-04-29-phased-plan.md            ← ESTE arquivo (decisão final)
├── docs/
│   ├── PROJECT-BRIEFING-FOR-REVIEW.md        ← briefing completo do projeto
│   └── shopee-integration.md                 ← integração branch
├── CLAUDE.md                                  ← regras + realidade operacional
└── app-v2/                                    ← V2 PRODUÇÃO
    └── ...                                    ← código no GitHub: rafaelwabisabi-maker/wms-laura-app-v2
```

## GitHub backup status (2026-04-29)

| Repo | Branch | Status |
|---|---|---|
| https://github.com/rafaelwabisabi-maker/wms-laura | main | ✅ pushed (commit eabfbca) |
| https://github.com/rafaelwabisabi-maker/wms-laura-app-v2 | main | ✅ pushed (private repo created) |
| https://github.com/rafaelwabisabi-maker/wms-laura-app-v2 | feature/shopee-integration | ✅ pushed |

**Reverter qualquer fix:** `git revert <commit>` ou `git checkout <prior-commit> -- path/file`. Backup garantido em 2 lugares (Mac + GitHub) + Guardian sync para VPS.


## Decisão estratégica

**Foco esta semana:** Fase 0 (~2-3h) + arrancar Fase 1 (Bulk Fill). 
**Defer ativamente:** Shopee branch (mantém parada e segura), todas as P0s da Shopee, baseline migration, tests, deploy.sh elaborado. 
**Resultado em 2 semanas:** sistema mais seguro, Bulk Fill no ar, 6/6 sem stress, com backup garantido.

**O perigo era:** virar engenheiro defensivo gastando 2 semanas em fixes preventivos enquanto Laura fica sem Bulk Fill, enquanto Lucas continua digitando NF manualmente, enquanto a 6/6 chega e o que importava era estabilidade do CSV.

**O caminho é:** consertar os 7 que importam (~2-3h), entregar o que Leo pediu (~6-8h), polir antes da promo (~4-6h). Total ~12-17h em 3 semanas. Sobra tempo para outros projetos.

→ **NEXT:** Fase 0, item 0.2 — remover bloco `RESET_DATA` do `app-v2/scripts/railway-init.js` (5min, zero risco, reversível via Git).
