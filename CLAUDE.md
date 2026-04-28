# WMS Laura — Mini Sistema de Controle de Estoque

## Contexto
- **Cliente:** Laura (sobrinha do Rafael)
- **Empresa:** Fast Action (pet shop supply)
- **Tipo:** Projeto pessoal/família — sem cobrança inicial, monetização futura possível (ver SEED-CONCEPT.md)
- **Idioma:** PT-BR para tudo (cliente + docs)
- **URL produção:** https://estoquefast.log.br
- **Hosting:** VPS Semeadora (159.69.202.5) — Docker direto no VPS (Railway foi descartado)
- **Status:** 🟢 LIVE em produção

## O Problema
Laura tem um armazém com 3 ruas de porta-paletes com endereços.
- Gerente (Edmilson) não tem tempo de registrar mercadoria na entrada
- Não sabe onde cada produto está
- Quer baixa automática quando bipa produto na separação
- WMS comercial é muito caro (R$800-3000/mês)

## Realidade Operacional (atualizado 2026-04-28 via intel/Leo)
- **Volume:** 1.500–2.000 pedidos/dia, picos de 2.000+ em promoções
- **SKUs ativos:** ~50–60 produtos distintos (variações contam separadas)
- **Canais:** Shopee (#1), Mercado Livre (#2), TikTok Shop (#3) — 100% digital
- **Planejado:** Amazon Brasil, talvez Temu
- **Hub central:** todos os pedidos passam pelo **Up Seller** (plataforma de etiquetas) — possível candidato a integração única
- **Picking:** SKU-driven (lê etiqueta visualmente, não escaneia produto)
- **Recebimento:** Lucas digita NF de papel manualmente — gargalo conhecido
- **Curva A:** ração/porta-ração ocupa ruas inteiras — precisa "fill rua" no WMS

📂 Intel completa: [intel/2026-04-28-leo-operations-interview.md](intel/2026-04-28-leo-operations-interview.md)

## Fluxo Operacional
1. Pedido cai no Up Seller (todos os canais) → etiqueta impressa com SKU + endereço
2. Edmilson divide etiquetas → pickers separam por SKU → embalam → cola etiqueta → expedição
3. Mercadoria nova chega → Edmilson recebe na doca → aloca no armazém
4. Lucas digita nota no sistema (manual)

## Warehouse Real (V2 — esquema do Leo, 2026-04-28)
- 3 ruas, 2 lados (E=Esquerda / D=Direita), níveis 1-3, **222 posições**
- **Rua 1:** colunas A-Q (17 cols × 2 lados × 3 níveis = 102 posições)
- **Rua 2:** colunas A-J (10 cols × 2 lados × 3 níveis = 60 posições)
- **Rua 3:** colunas A-J (10 cols × 2 lados × 3 níveis = 60 posições)
- **Formato de endereço:** `[RUA][COLUNA][NÍVEL][E/D]` — ex: `1B2E` = Rua 1, Coluna B, Nível 2, Esquerda
- Codes definidos pelo Leo + Edmilson em 2026-04-28 (substituiu o esquema legado `AA MM.NN.PPP`)
- **Pendente:** posições adicionais de chão entre Rua 1-2 e em frente da Rua 3 (Leo manda mapa atualizado)

## Usuários do Sistema
| Nome | Email | PIN | Role |
|------|-------|-----|------|
| Laura | laura@fastaction.com | 1001 | ADMIN |
| Edmilson | edmilson@fastaction.com | 1002 | OPERATOR |
| Lucas | lucas@fastaction.com | 1003 | OPERATOR |
| Thiago | thiago@fastaction.com | 1004 | OPERATOR |
| Leonardo | leonardo@fastaction.com | 1005 | OPERATOR |
| Henrique | henrique@fastaction.com | 1006 | OPERATOR |

## Stack (definitivo)
- Next.js + TypeScript + Prisma ORM + SQLite (better-sqlite3)
- NextAuth v5 (Edge-safe auth.config.ts + full auth.ts)
- shadcn/ui + Tailwind CSS
- Docker + VPS Semeadora (159.69.202.5)
- Domínio: estoquefast.log.br (DNS A record → 159.69.202.5)

## App V1 vs V2
- **app/** — V1: order management, driver management, SMS/email updates
- **app-v2/** — V2 ATUAL em produção: WMS completo com todos os módulos abaixo

## Módulos (app-v2 — PRODUÇÃO)
| Módulo | Status |
|--------|--------|
| Login (email + PIN rápido) | ✅ |
| Dashboard | ✅ |
| Mapa do armazém | ✅ |
| Produtos | ✅ |
| Recebimento | ✅ |
| Separação (picking) | ✅ |
| Scanner (câmera) | ✅ |
| Transferência | ✅ |
| Ajuste de estoque | ✅ |
| Contagem | ✅ |
| Histórico | ✅ |
| Baixa de Vendas (CSV import) | ✅ v2.1 |
| Configurações | ✅ |

## Versões / Changelog
- **v2.1** — Módulo "Baixa de Vendas": import de planilha CSV para baixa automática no estoque
- **v2.2** — Campo `unitsPerPack` no Product (unidades por pack/caixa)
- Migrations idempotentes via `railway-init.js` (nome legado — roda no startup do Docker no VPS)

## Arquivos
- `docs/questionario.md` — perguntas iniciais para Laura
- `docs/analise-viabilidade.md` — análise completa (custos, tempo, hardware)
- `docs/proposta-integracao-shopee-*.pdf` — integração Shopee (v1–v4)
- `DESIGN.md` — design system e guidelines visuais
- `SEED-CONCEPT.md` — plano de productização como SaaS (SOL Business #5)
- `app-v2/.claude/napkin.md` — erros conhecidos + padrões que funcionam
- `app-v2/scripts/railway-init.js` — init do servidor (migrations + seed users; nome legado, roda no VPS)
