# Análise de Viabilidade — Mini-WMS

> ⚠️ **Versão preliminar** — será atualizada após respostas do questionário da Laura.

## Contexto
- **Cliente:** Laura (sobrinha do Rafael) — projeto família, sem cobrança inicial
- **Desenvolvimento:** Rafael + Claude (AI-assisted coding)
- **Problema:** WMS comercial custa R$800-3.000/mês, inviável pra ela agora

## O que seria o sistema (MVP)

| Módulo | Funcionalidade |
|--------|---------------|
| **Mapa do armazém** | Grid visual: 3 ruas × N níveis × N posições. Mostra o que tem em cada endereço |
| **Recebimento** | Produto chega → escaneia → atribui endereço → registra quantidade |
| **Separação (picking)** | Funcionário recebe pedido → bipa produto → sistema dá baixa automática |
| **Dashboard** | Estoque atual, ocupação por rua, alertas de estoque baixo |
| **Busca** | "Onde está a casinha de cachorro?" → R2-N1-P5 |

## Custo de Infraestrutura (nosso)

| Item | Custo |
|------|-------|
| Hosting (VPS) | ~€10/mês |
| Domínio | ~€10/ano |
| Libs/APIs | Zero — tudo open source |
| Claude Code | Já pago (custo marginal = 0) |
| **Total infra** | **~€10/mês** |

## Valor de Mercado (o que isso VALE)

### Se fosse cobrado como projeto freelance

Taxa freelancer BR (senior full-stack): ~R$200/hora
Taxa freelancer EU: ~€100/hora

| Cenário | Horas mercado | Horas reais (com AI) | Valor BR (R$200/h) | Valor EU (€100/h) |
|---------|:-:|:-:|:-:|:-:|
| **MVP mínimo** | 40h | ~15-20h | R$4.000-8.000 | €2.000-4.000 |
| **MVP completo** | 80h | ~30-40h | R$12.000-16.000 | €6.000-8.000 |
| **Com integração ERP** | 140h | ~50-70h | R$20.000-28.000 | €10.000-14.000 |

> AI-assisted dev (Claude) = 2-3x mais rápido que dev solo tradicional.
> O MVP completo vale ~R$12.000-15.000 no mercado.

### Se fosse cobrado como SaaS (mensalidade)

WMS comercial no Brasil: R$800 a R$3.000+/mês

| Modelo | Valor | O que inclui |
|--------|-------|-------------|
| Básico | R$300-500/mês | Sistema + hosting + suporte WhatsApp |
| Intermediário | R$500-800/mês | + atualizações mensais + treinamento |
| Setup + mensalidade | R$3.000-5.000 (setup) + R$400/mês | Implementação à parte |

## Hardware Necessário

| Item | Necessário? | Custo | Alternativa gratuita |
|------|-------------|-------|---------------------|
| Leitor código de barras USB | Recomendado | R$80-200 | Câmera do celular |
| Tablet ou celular no armazém | Sim (mín. 1) | Provavelmente já têm | — |
| Wi-Fi cobrindo o armazém | Sim | Provavelmente já têm | — |
| Servidor/PC dedicado | Não | — | Roda na nuvem |
| Impressora de etiquetas | Opcional (fase 2) | R$300-800 | Etiquetas manuais |

**Resumo: um celular com câmera + Wi-Fi já resolve.**

## Estimativas de Tempo

| Cenário | Escopo | Tempo (~20h/sem) |
|---------|--------|-----------------|
| MVP mínimo | Endereçamento + busca (sem integração) | 1-2 semanas |
| MVP completo | Endereçamento + picking + dashboard | 3-4 semanas |
| Com integração ERP | MVP completo + conectar ao sistema deles | 5-7 semanas |

## Estratégia Recomendada

Entrega em fases:
- **Fase 1:** Endereçamento + busca (2 semanas) — já começa a usar
- **Fase 2:** Picking com baixa automática (+2 semanas)
- **Fase 3:** Dashboard + relatórios (+1 semana)
- **Fase 4:** Integração ERP se necessário (+2 semanas)
