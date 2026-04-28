---
status: WIRED
type: client_interview
project: wms-laura
source: Leo (Leonardo, gerente de operações Fast Action)
medium: WhatsApp voice messages
date: 2026-04-28
ingested_by: Claude (Rafael)
wires_into:
  - wms-laura/CLAUDE.md (operational reality update)
  - wms-laura/intel/INDEX.md (intel registry)
  - wms-laura/SEED-CONCEPT.md (productization signals)
  - app-v2/docs/shopee-integration.md (multi-channel context)
priority: HIGH
tags: [operations, multi-channel, shopee, mercadolivre, tiktok, sku, picking, receiving, volume]
---

# Fast Action — Entrevista de Operações com Leo

> **Data:** 2026-04-28
> **Quem respondeu:** Leonardo (Leo), gerente de conta / suporte ao Edmilson (gerente de operações física)
> **Empresa:** Fast Action (pet shop supply digital, Brasil)
> **Cliente WMS:** Laura (sobrinha do Rafael)
> **Status do WMS:** LIVE em [estoquefast.log.br](https://estoquefast.log.br) com 222 posições reconfiguradas hoje

---

## Quadro operacional rápido

| Métrica | Valor |
|---|---|
| Pedidos/dia normal | 1.500–2.000 |
| Pedidos/dia em promo | 2.000+ |
| SKUs distintos ativos | ~50–60 (variações contam separadamente) |
| Anúncios ativos por loja | 45–75 |
| Canais ativos | Shopee, Mercado Livre, TikTok Shop |
| Canais planejados | Amazon Brasil (próximo), Temu (futuro) |
| Vendas físicas | **Zero** — 100% digital |
| Posições no armazém | 222 (Rua 1: 102, Rua 2: 60, Rua 3: 60) |
| Estoque atual | Cheio — poucos espaços vagos |

---

## 1. Canais de venda

**Resposta verbatim:**
> "A Shopee não é o nosso único canal de venda. (...) Shopee em primeiro lugar, Mercado Livre em segundo, TikTok Shop em terceiro. Estamos com o plano de abrir Amazon Brasil, talvez Temu. Não temos nenhum meio de venda física."

**Implicação para o WMS:**
- A integração Shopee que construímos cobre ~60% do volume (estimativa)
- ML, TikTok Shop, Amazon precisarão de integrações análogas
- **Continua válido o pattern Shopee:** webhook → fila → processOrder → DEDUCT — só muda o adapter por marketplace

---

## 2. Fluxo de pedido (do clique ao envio)

**Resposta verbatim:**
> "Esse pedido cai aqui para o nosso Up Seller que é uma plataforma aqui de impressão de etiquetas. (...) Edmilson faz a impressão de todas as milhares de etiquetas que a gente tem aqui por dia, divide, passa para o pessoal, o pessoal vai olhando as etiquetas e conforme o SKU eles vão separando os produtos, fazendo a embalagem, colam a etiqueta e jogam para a expedição."

**Mapa do fluxo:**
```
[Marketplace] → [Up Seller] → [Etiqueta impressa: SKU + endereço + cliente]
                                          ↓
                                    [Picker lê SKU]
                                          ↓
                                    [Separa produto]
                                          ↓
                                    [Embala + cola etiqueta]
                                          ↓
                                    [Expedição]
```

**🔴 INSIGHT CRÍTICO — Up Seller:**
- Up Seller já agrega pedidos de TODOS os canais (Shopee, ML, TikTok)
- Já é a fonte única da verdade operacional de pedidos
- **Pergunta para investigar:** Up Seller tem API? Se sim, integrar com Up Seller resolve multi-canal de uma vez só.
- Possíveis identidades: Bling, Tiny, Plugg.to, Magis5, NuvemShop, ou ferramenta proprietária regional. Precisa confirmar com Leo.

**Implicação para WMS:**
- Hoje o WMS recebe Shopee diretamente → cria StockMovement
- Se integrarmos com Up Seller, recebemos TODOS os pedidos de uma fonte só
- Mas perdemos o "tempo real" do webhook Shopee (Up Seller pode ter delay)
- **Decisão pendente:** integração paralela (Shopee direto + Up Seller para o resto) vs migrar tudo para Up Seller

---

## 3. Convenção de SKU

**Resposta verbatim:**
> "Não tá nos melhores padrões. Não tá padronizado. Mas todos têm SKU. (...) Por exemplo, copo suco, é 4. Então é o kit 4, 4 copos suco 400. 6 copos suco 400. Se for 450, é 6 copos suco 450. (...) A gente classifica de um jeito que vai ficar entendível pra operação."

**Padrão observado (descritivo, não estruturado):**
- `[QTD] [PRODUTO] [VARIAÇÃO]`
- Exemplos:
  - `4 copos suco 400` = kit 4 unidades de copo suco 400ml
  - `6 copos suco 450` = kit 6 unidades de copo suco 450ml

**Implicação para WMS:**
- O `Product.sku` no WMS precisa bater **exatamente** com o SKU configurado em cada marketplace
- A tabela `ShopeeSkuMapping` que já construímos cobre divergências — vale para todos os canais
- **Recomendação para Leo (longo prazo):** padronizar para `[CATEGORIA]-[VARIAÇÃO]-[QTD]`, ex: `COPO-SUCO-400-4UN` — mas hoje a operação já funciona, não é urgente
- **Recomendação para o WMS (curto prazo):** UI para bulk import de mapping SKU marketplace → SKU WMS via CSV

---

## 4. Código de barras (scanner)

**Resposta verbatim:**
> "A maioria tem sim, viu. Mas não são todos os produtos que têm individualmente esse código no produto. Às vezes tem na caixa. Tipo copo de vidro, a caixa vem com 24, tem o código de barras na caixa. Mas não dá pra garantir [todos]."

**Estado real:**
- Cobertura de barcode individual: **parcial** (~50-70% estimado)
- Cobertura de barcode na caixa-mãe: **alta**
- Picker hoje **não scaneia produto a produto** — ele lê o SKU da etiqueta visualmente e separa

**Implicação para WMS:**
- O módulo Scanner que existe no app-v2 só serve para 1) recebimento (caixa-mãe), 2) verificação de produtos premium
- **Não vale insistir em scanner para picking** — não funciona com 30-50% dos itens
- Picking continua sendo SKU-driven (visual leitura da etiqueta)
- **Possível evolução:** scanner mode para entrada de mercadoria, validando contra NF-e

---

## 5. Volume

**Resposta verbatim:**
> "A gente recebe entre 1500 a 2000 pedidos por dia. Quando é dia de promoções, a gente gira ali entre 1500 e 2000 pedidos dia. (...) 50, 60 produtos diferentes."

**Carga sobre o WMS:**
- 1500-2000 webhooks Shopee por dia (em pico de promo)
- ~1 webhook a cada 50 segundos em média
- Em pico (digamos 200 pedidos/hora em rajada): 1 a cada 18s
- **Conclusão:** o worker async + Shopee 1s ack que construímos absorve isso de boa
- Burst risk: se 50 pedidos chegam em 30s (promo flash), webhook engole tudo, processador limpa em ~3-5 min

---

## 6. Recebimento de mercadoria

**Resposta verbatim:**
> "Edmilson faz o recebimento. (...) Hoje o nosso estoque está bem cheio, a gente tem poucos espaços vagos dentro do estoque. (...) Em algumas ruas a gente preenche também com produtos e vai movimentando conforme vai saindo. Isso a gente faz mais com nossos produtos de giro alto — rações de cachorro, de gato, porta-ração, que é um dos produtos curva A. (...) A nota chega praticamente sempre em papel. (...) Lucas recebe, coloca tudo que chegou no estoque manualmente."

**🔴 INSIGHT CRÍTICO — Curva A:**
- Produtos de alto giro (rações, porta-ração) **ocupam ruas inteiras** sequencialmente
- O WMS precisa permitir "fill rua" — operação batch de 1 produto X 30+ posições
- Hoje o WMS exige 1 entrada por posição → **muito atrito para curva A**

**🔴 INSIGHT CRÍTICO — Notas em papel:**
- Lucas digita TODA nota manualmente
- 1500-2000 pedidos saindo por dia → entradas de mercadoria várias vezes por semana
- Qualquer coisa que automatize NF-e (XML) elimina horas/semana de trabalho do Lucas
- Brasil tem padrão SEFAZ NF-e XML — fornecedores grandes mandam por email automaticamente

---

## 7. Cadastro de produtos

**Resposta verbatim:**
> "Todos os nossos produtos a gente cadastra SKU. (...) Sempre distinto. (...) É uma das formas que a gente usa aqui para o pessoal realmente conseguir fazer a separação dos itens."

**Status:** consistência interna OK. Padronização entre canais OK (mesmos SKUs em Shopee/ML/TikTok). Padronização sintática entre produtos: improvável, mas operacionalmente funcional.

---

## Backlog priorizado para o WMS (sem quebrar nada)

### 🟢 EVOLUÇÕES IMEDIATAS (sem risco, alto valor)

| # | Feature | Esforço | Valor | Quem ganha |
|---|---|---|---|---|
| 1 | **Bulk SKU mapping CSV import** — operador cola CSV de SKU Shopee → SKU WMS | 2h | Alto | Leo, Laura |
| 2 | **Filtro "só posições vazias" no mapa** — receiving rápido | 30min | Alto | Edmilson |
| 3 | **Marker visual de Curva A no mapa** — campo `Product.turnoverClass` | 1h | Médio | Edmilson |
| 4 | **Dashboard de volume** — pedidos hoje vs ontem vs média 7d | 3h | Médio | Laura |
| 5 | **Generic `MarketplaceSkuMapping`** — refatorar `ShopeeSkuMapping` para multi-canal | 2h | Alto (preparação ML/TikTok) | Tech debt zero |

### 🟡 EVOLUÇÕES MÉDIAS (próxima sprint)

| # | Feature | Esforço | Valor |
|---|---|---|---|
| 6 | **"Fill Rua" batch entry** — 1 produto X N posições em 1 click | 6h | Alto (Curva A) |
| 7 | **Up Seller integration discovery** — descobrir o que é, ver se tem API | 2h research | Potencialmente enorme |
| 8 | **NF-e XML import** — Lucas dropa XML, sistema cria entradas | 8-12h | Altíssimo (horas/semana) |
| 9 | **Mercado Livre webhook integration** — clone do pattern Shopee | 1-2 dias | Alto (segundo canal de receita) |

### 🔴 ALTERNATIVAS DE ARQUITETURA (precisam decisão)

| # | Opção | Pros | Cons |
|---|---|---|---|
| A | **Continuar 1 integração por marketplace** (Shopee, ML, TikTok, Amazon separados) | Tempo real, controle granular | 4 integrações para manter |
| B | **Integrar com Up Seller** (single source) | 1 integração serve todos os canais | Depende de API do Up Seller; possível delay de pedido |
| C | **Híbrido: Shopee direto (alto volume) + Up Seller para o resto** | Otimiza o canal #1 sem reescrever tudo | Complexidade de duas fontes para reconciliar |

**Decisão sugerida:** **C (híbrido)**, mas só depois de descobrir o que Up Seller suporta. Próximo passo: perguntar ao Leo "qual a plataforma do Up Seller exatamente?".

---

## Perguntas pendentes para o Leo

1. **Up Seller é qual plataforma?** Bling, Tiny, Plugg.to, Magis5, ou outra? Tem API?
2. **Algum fornecedor manda NF-e por email automaticamente em XML?**
3. **Quando é a próxima promoção grande (Shopee 11.11, BF, etc.)?** Para validar carga antes.
4. **Edmilson quer poder "fill rua" no WMS** ou prefere registrar posição por posição?
5. **Quem responde por mapping de SKU Shopee → produto interno?** Leo, Edmilson, ou Lucas?
6. **Tem etiqueta única no produto interno** (não é o SKU do marketplace)? Se sim, qual o padrão?

---

## Auditoria deste documento

- **Verbatim quotes:** preservadas literalmente nas seções correspondentes
- **Source:** WhatsApp voice messages do Leo, ingeridas via Rafael em 2026-04-28
- **Decisões tomadas com base nesse intel:**
  - 222 posições migradas no banco com esquema do Leo (`1B2E`)
  - Componente warehouse-map refatorado para colunas variáveis
- **Pendente:** atualizar `wms-laura/CLAUDE.md` com volume real (1500-2000/dia) e canais
- **Wire-out:** este doc deve ser citado em qualquer planejamento de integração ML/TikTok/Amazon
