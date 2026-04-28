---
status: WIRED
type: client_interview_followup
project: wms-laura
source: Leo (Fast Action)
medium: WhatsApp voice messages
date: 2026-04-28
parent_doc: 2026-04-28-leo-operations-interview.md
wires_into:
  - wms-laura/intel/2026-04-28-leo-operations-interview.md (parent doc)
  - app-v2/docs/shopee-integration.md (Up Seller findings)
  - wms-laura/CLAUDE.md (warehouse map: 353 positions)
priority: HIGH
tags: [upseller, nfe, fill-batch, sku, marketplace, promo-calendar]
---

# Leo Follow-up — Respostas às 6 perguntas pendentes

> Sequência da [primeira entrevista](2026-04-28-leo-operations-interview.md) — respostas dadas no mesmo dia, à tarde.


## 1. ✅ Up Seller — o que é

**Resposta verbatim:**
> "O UpSeller é uma plataforma, um sistema de ERP que a gente utiliza aqui. Gratuito, focado em e-commerce. A gente usa mais aqui para gerenciamento geral de pedidos, produtos e emissão das nossas fiscais e também das etiquetas."

**Link enviado pelo Leo:** https://www.upseller.com/pt/

**Descobertas próprias (research 2026-04-28):**
- Empresa chinesa (CDN em `cdn.upseller.cn`) com produto branded para LATAM
- 10+ anos de SaaS, 200K+ lojas em LATAM, 100M+ pedidos/mês processados
- Free tier real (não trial)
- Integra com 15+ marketplaces: Shopee, ML, Amazon, Shein, TikTok Shop, Shopify, Nuvemshop, Temu, Walmart, Falabella, AliExpress, Magalu, Kwai Shop, Americanas
- Features: NF-e auto, etiquetas em massa (1000 por vez), wave picking, scan para checkout, sync de estoque entre lojas, marketing/promoções multi-loja
- **Páginas testadas para API:**
  - `open.upseller.com` → 000 (não resolve)
  - `api.upseller.com` → 000 (não resolve)
  - `developer.upseller.com` → 000 (não resolve)
  - `docs.upseller.com` → 000 (não resolve)
  - `upseller.com/pt/openapi` → 200 mas é redirect pra homepage (não tem doc real de API pública)

**🔴 CONCLUSÃO:** Up Seller **não tem API pública aparente** para integração third-party. É uma plataforma fechada. Categoria SaaS chinês orientado a usuário final, não a integradores.

**Implicação estratégica:**
- ❌ Plano B descartado: "integrar com Up Seller para resolver multi-canal de uma vez"
- ✅ Plano A confirmado: integração direta com cada marketplace (Shopee feito, ML/TikTok/Amazon próximos)
- ⚠️ Risco oculto: Up Seller faz **sync de estoque** entre marketplaces. Se o WMS deduz estoque, mas Up Seller também sincroniza, podemos ter conflito → **precisa investigar com Leo se Up Seller atualmente está controlando estoque ou só etiquetas**


## 2. ⏳ NF-e XML — pendente confirmação com Lucas

**Resposta verbatim:**
> "Eu vou confirmar com o Lucas. (...) Eu acredito que tem sim quem envia via e-mail, mas não sei se seria nesse formato exato aí de XML."

**Status:** Leo vai checar com Lucas e mandar exemplo se houver.

**Próxima ação:** aguardar resposta do Lucas. Se confirmar XML → priorizar feature de import.


## 3. 📅 Calendário de promoções

**Resposta verbatim:**
> "Normalmente, na Shopee, é todo o dia do número do mês. Dia 2 do mês 2, dia 3 do mês 3. Em março especificamente teve dia 15 também, que foi o Dia do Consumidor. Mas continua: dia 4 do mês 4, dia 5 do mês 5, dia 6 do mês 6 e por aí vai."

**Calendário extraído:**

| Data | Evento | Status |
|---|---|---|
| 2/2 | Promo Shopee 2.2 | passou |
| 3/3 | Promo Shopee 3.3 | passou |
| 15/3 | Dia do Consumidor | passou |
| 4/4 | Promo Shopee 4.4 | passou |
| **5/5** | Promo Shopee 5.5 | **passou (3 dias atrás)** |
| **6/6** | Promo Shopee 6.6 | **PRÓXIMO — em ~5 semanas** |
| 7/7 | Promo Shopee 7.7 | T+10 semanas |
| 8/8 | Promo Shopee 8.8 | T+14 semanas |
| 9/9 | Promo Shopee 9.9 | T+19 semanas |
| 10/10 | Promo Shopee 10.10 | T+23 semanas |
| 11/11 | Promo Shopee 11.11 (BIG) | T+28 semanas |
| 12/12 | Promo Shopee 12.12 | T+33 semanas |

**Implicação para WMS:**
- Próxima promo grande: **6/6** (~5 semanas) → janela para validar Shopee integration sob carga real
- Promo "X.X" tipicamente dobra o volume diário (1.500 → 3.000+ pedidos)
- **Recomendação:** rodar piloto Shopee paralelo ao CSV até semana antes do 6/6, depois desligar o CSV


## 4. ✅ Fill Batch — Leo APROVOU e quer urgentemente

**Resposta verbatim:**
> "Muito boa essa pergunta! Sim, é interessante a gente ter a possibilidade de preencher uma rua inteira ou então, nesse novo mapa que eu te passei, uma posição de chão inteira ao invés de ter que ir preenchendo um por um. Como se fosse um ctrl-c, ctrl-v. Quando a gente recebe algumas cargas dos nossos produtos curva A, então por exemplo porta-ração, ração de cachorro, ração de gato, normalmente a gente entope uma rua inteira com isso. Na verdade, a gente costuma entupir a posição de chão dessa rua, entre a rua, o espaço de manobra ali onde a gente levanta a carga e coloca nas posições. Quando chega um caminhão cheio desse, e nosso estoque já tem quase todas as posições preenchidas, a gente coloca a rua inteira ali entre, por exemplo, de ração ou então de porta-ração."

**🔴 PRIORIDADE MÁXIMA — Feature: Bulk Fill**

**Comportamento esperado:**
1. Usuário (Edmilson) seleciona 1 produto + 1 quantidade
2. Seleciona um conjunto de posições:
   - Opção A: range na rua (ex: "todas as posições da Rua 2")
   - Opção B: zona inteira (ex: "todo o Centro 1")
   - Opção C: ctrl+click múltiplo no mapa
3. Sistema cria N entradas de Stock + N StockMovements (type=INBOUND) em batch transacional
4. Movimentos referenciados como `BULK:[id]` para audit + reversão fácil

**Use case real (cenário-tipo):**
- Caminhão chega com 30 paletes de ração
- Edmilson clica "Fill Centro 1 com Ração X 15kg, 1 palete por posição"
- Sistema preenche C1P1...C1P18 + R1CH1...R1CH12 = 30 posições com 1 palete cada
- Tudo em 1 transação. 1 movimentação por posição (audit trail completa).


## 5. 📝 SKU mapping — quem cria

**Resposta verbatim:**
> "O SKU a gente atrela ao produto na hora de criar o anúncio. (...) Varia normalmente de quem está criando o anúncio. Às vezes pode ser eu, às vezes pode ser a Laura, às vezes pode ser o Tiago, pode ser o Henrique. Normalmente não é o Lucas que estabelece e nem o Edmilson quem estabelece os SKUs. Com o SKU atrelado ao anúncio, quando sai um pedido, essas informações vão para o pedido da Shopee e também vão para o Up Seller, para a gente poder emitir a nota e a etiqueta."

**Implicação:**
- SKU é definido upstream (Shopee/marketplace) por 4 pessoas: **Leo, Laura, Thiago, Henrique**
- Lucas e Edmilson recebem SKU pronto da etiqueta
- Não há padronização entre as 4 pessoas que criam → SKUs heterogêneos
- O WMS precisa fazer match exato (SKU marketplace = SKU produto cadastrado)

**Risco identificado:**
- 4 pessoas criando SKU sem padrão = alta chance de divergência entre canais
- Ex: Leo cria "4 copos suco 400" no Shopee; Henrique cria "Copo Suco 400ml 4un" no ML
- Mesmo produto físico, SKUs diferentes nos marketplaces → preciso de mapping em N tabelas

**Recomendação:**
- Tabela `MarketplaceSkuMapping` (já planejada) é OBRIGATÓRIA
- UI deve permitir ver "este produto físico está com SKU X no Shopee, Y no ML, Z no TikTok"


## 6. ❌ Etiqueta interna do produto — não existe

**Resposta verbatim:**
> "Não temos uma etiqueta única do produto interno. A gente utiliza o SKU mesmo do Marketplace. Como eu falei, a gente não tem uma padronização de SKU. (...) A gente tenta deixar fácil de entender. (...) Até então vem funcionando super bem. Mas acredito que futuramente o ideal seja uma padronização mesmo."

**Implicação para WMS:**
- O `Product.sku` no WMS = SKU do marketplace principal (Shopee)
- Não há código interno "neutro" para tracking físico
- Quando produtos têm SKU diferente entre marketplaces, **um deles vira o "principal"** e os outros viram aliases via `MarketplaceSkuMapping`
- **Decisão sugerida:** SKU principal = Shopee (canal #1 em volume)

**Oportunidade futura (não agora):**
- Implementar `Product.internalCode` opcional (ex: `FA-COPO-400-04`)
- Etiqueta física opcional na caixa quando recebe mercadoria
- Pega quando: Leo decidir padronizar (provavelmente depois do 11.11)


## 📊 Atualização do mapa do armazém

Leo enviou novo Excel ([MAPA ESTOQUE (2).xlsx](file:///Users/apple/Downloads/MAPA%20ESTOQUE%20%20%282%29.xlsx)) que adiciona **131 posições novas** ao mapa, totalizando **353 posições**:

| Tipo | Código | Quantidade | Localização física |
|---|---|---|---|
| **STORAGE** (porta-paletes) | `1B2E` | 222 | 3 ruas com racks |
| **CENTRO** (entre ruas) | `C1P1` | 54 | 3 corredores × 18 pos |
| **CHAO_RUA** (em frente de cada rua) | `R1CH1` | 64 | Rua 1: 34 / Rua 2: 24 / Rua 3: 6 |
| **CHAO_PAREDE** (parede) | `CHP1` | 13 | 1 parede com 13 pos |

**Status:** ✅ migrado em produção (2026-04-28, backup em `/data/wms-backup-pre-floor.db`)


## 🎯 Backlog atualizado pós-respostas

### 🟢 PRIORIDADE 1 — pode rodar AGORA (sem bloqueio)

| # | Feature | Esforço | Origem |
|---|---|---|---|
| ✅ | Migrar 131 posições de chão | DONE | Q4 + novo mapa |
| ✅ | Estender warehouse-map para 4 zonas | DONE | Q4 + novo mapa |
| 1 | **Bulk Fill** — 1 produto X N posições selecionadas | 4-6h | Q4 (urgente) |
| 2 | `MarketplaceSkuMapping` genérico | 2h | Q5 |
| 3 | Filtro "só posições vazias" no mapa | 30min | Q5 evidência |

### 🟡 PRIORIDADE 2 — próxima semana

| # | Feature | Esforço | Origem |
|---|---|---|---|
| 4 | Dashboard volume + alerta promo dia X.X | 3h | Q3 |
| 5 | Mercado Livre webhook integration | 1-2 dias | Q1 (Up Seller sem API) |
| 6 | Verificar com Leo se Up Seller controla estoque atualmente | 5min msg | risco oculto Q1 |

### 🔴 BLOQUEADO / AGUARDANDO

| # | Feature | Bloqueio |
|---|---|---|
| 7 | NF-e XML import | Q2 — Leo confirmar com Lucas |
| 8 | Internal product code | Q6 — Leo decidir padronizar (não agora) |


## 🔗 Links úteis

- Up Seller homepage: https://www.upseller.com/pt/
- Shopee promo calendar (extraído): 6/6 próxima grande
- WMS prod: https://estoquefast.log.br
