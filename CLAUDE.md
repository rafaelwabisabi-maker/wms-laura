# WMS Laura — Mini Sistema de Controle de Estoque

## Contexto
- **Cliente:** Laura (sobrinha do Rafael)
- **Tipo:** Projeto pessoal/família — sem cobrança inicial, monetização futura possível
- **Idioma:** PT-BR para tudo (cliente + docs)

## O Problema
Laura tem um armazém com 3 ruas de porta-paletes com endereços.
- Gerente não tem tempo de registrar mercadoria na entrada
- Não sabe onde cada produto está
- Quer baixa automática quando bipa produto na separação
- WMS comercial é muito caro (R$800-3000/mês)

## Fluxo Operacional (como funciona hoje)
1. Mercadoria chega → precisa ir pra um endereço (rua/nível/posição)
2. Funcionários recebem pedido → separam na mesa → embalam → expedição → caminhão

## Duas Dores Principais
1. **Endereçamento** — saber ONDE cada produto está (Rua X, Nível Y, Posição Z)
2. **Baixa automática** — bipar produto na separação → sai do estoque automaticamente

## MVP Planejado

| Módulo | Funcionalidade |
|--------|---------------|
| Mapa do armazém | Grid visual: 3 ruas × N níveis × N posições |
| Recebimento | Produto chega → escaneia → atribui endereço → registra qty |
| Separação (picking) | Pedido → bipa produto → baixa automática |
| Dashboard | Estoque atual, ocupação por rua, alertas |
| Busca | "Onde está X?" → R2-N1-P5 |

## Stack (a definir)
- Web app (Next.js ou similar)
- Banco de dados (PostgreSQL ou SQLite)
- Leitor código de barras via câmera do celular (Web API)
- Hosting: VPS ~€5-15/mês

## Hardware Necessário
- Mínimo: celular com câmera + Wi-Fi no armazém
- Opcional: leitor USB de código de barras (~R$80-200)

## Estimativas de Tempo
| Cenário | Escopo | Tempo (~20h/sem) |
|---------|--------|-----------------|
| MVP mínimo | Endereçamento + busca | 1-2 semanas |
| MVP completo | + picking + dashboard | 3-4 semanas |
| Com integração ERP | + conectar sistema deles | 5-7 semanas |

## Status
- [x] Análise inicial de viabilidade
- [x] Questionário de levantamento criado (8 perguntas)
- [ ] Aguardando respostas da Laura
- [ ] Definir stack e arquitetura
- [ ] Implementação

## Arquivos
- `docs/questionario.md` — perguntas para Laura
- `docs/analise-viabilidade.md` — análise completa (custos, tempo, hardware)
