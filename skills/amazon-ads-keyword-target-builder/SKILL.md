---
name: amazon-ads-keyword-target-builder
description: Use when an AI agent needs to choose Amazon Ads keywords, product targets, match types, bids, and negative seeds for Sponsored Products campaigns using Titanos MCP data and safe launch logic.
---

# Amazon Ads Keyword & Target Builder — Titanos MCP

## Objetivo

Selecionar keywords e product targets bons o suficiente para uma campanha Sponsored Products sem depender de chute.

Use esta skill junto com `amazon-sp-campaign-launcher` quando precisar montar a parte de targeting: exact, phrase, broad controlado, ASIN targeting e negativos iniciais.

## Fontes de dados, em ordem

Priorize dados reais. Não invente demanda.

1. Histórico Ads da própria conta:
   - `ask_ads_report_analyst`
   - `get_search_term_report`
   - `get_ads_performance`
   - `get_advertised_product_report`
2. Recomendações Amazon:
   - `list_ads_recommendations`
   - `get_bid_guidance`
3. Inteligência Titanos/ABA:
   - `expand_keywords`
   - `reverse_asin_keywords`
   - `thanos_rank_keywords`
   - `get_keyword_serp`
4. Catálogo/concorrência:
   - `lookup_amazon_product`
   - Product Finder / SERP / reverse ASIN.
5. Lógica semântica conservadora, só quando dados forem insuficientes.

## Janela de análise

Padrão:

- 60 dias completos para decisão.
- Se há pouco volume, expandir para 120 dias.
- Para lançamento sem histórico, usar mercado/ABA/concorrentes e marcar como teste.

Nunca trate 1 pedido como prova. Para este seller, termo ou ASIN vencedor precisa de **3+ pedidos/conversões**.

## Classificação de keywords

| Classe | Exemplo | Uso | Match |
|---|---|---|---|
| Head core | `cadeira mocho` | volume principal | EXACT + PHRASE |
| Long-tail compra | `cadeira mocho sem encosto` | intenção alta | EXACT |
| Profissão/uso | `mocho manicure`, `mocho esteticista` | público comprador | EXACT + PHRASE |
| Atributo | `mocho giratorio`, `mocho preto` | variação/atributo | PHRASE |
| Problema/ocasião | `cadeira para salão de beleza` | descoberta | PHRASE/BROAD controlado |
| Irrelevante | `rodinha cadeira`, `capa cadeira` | negativo | NEGATIVE EXACT/PHRASE |

## Regras de seleção

### Exact

Use para:

- termos principais do produto,
- termos com 3+ pedidos,
- termos óbvios de alta intenção,
- termos de SKU/produto com forte fit.

Bid: 100% do bid calculado.

### Phrase

Use para:

- descoberta controlada,
- variações de termos core,
- termos com intenção boa mas volume incerto.

Bid: 60–80% do exact.

### Broad

Use com cuidado.

Use apenas quando:

- conta precisa de descoberta,
- orçamento separado,
- negativos iniciais estão claros,
- usuário aceita teste.

Bid: 40–60% do exact.

### Product targeting / PAT

Use para:

- ASINs concorrentes diretos,
- produtos substitutos,
- produtos com preço/rating/página pior que o seu,
- próprios ASINs para defesa/cross-sell.

Bid: geralmente abaixo do exact, salvo concorrente validado.

## Como montar lista inicial

### 1. Seeds humanas

Extraia do título/listing:

- substantivo principal,
- uso,
- atributo,
- público,
- material/cor/tamanho,
- problema que resolve.

### 2. Expansão Titanos

Use `expand_keywords` com 2–5 seeds principais.

Exemplo:

```json
{ "seed": "cadeira mocho", "limit": 50 }
```

### 3. Reverse ASIN

Para concorrentes importantes:

```json
{ "asin": "B0XXXXXXXXX", "limit": 30 }
```

Classifique apenas termos relevantes ao produto.

### 4. SERP/concorrentes

Use `get_keyword_serp` para termos core e capture ASINs com:

- mesmo tipo de produto,
- preço parecido,
- reviews medianos/baixos,
- ranking/BSR relevante,
- anúncio/posição orgânica forte.

Evite ASINs que não são substitutos reais.

## Bids iniciais

Fórmula:

```text
max_cpc = preço × CVR estimado × target_ACOS
```

Se não tiver CVR histórico:

| Tipo | CVR assumido inicial |
|---|---:|
| Exact core | 1.5–2.5% |
| Phrase | 1.0–1.5% |
| Broad | 0.7–1.2% |
| PAT concorrente | 0.8–1.5% |

Ajuste para baixo quando:

- listing novo,
- poucas reviews,
- estoque baixo,
- Buy Box incerta,
- produto caro/consideração longa.

## Negativos iniciais

Crie negativos para:

- acessórios que o produto não vende,
- peças/reposição,
- produto de categoria errada,
- intenção gratuita/download/manual,
- marcas incompatíveis quando não for conquest.

Exemplo para cadeira/mocho:

```text
capa para cadeira
rodinha cadeira
pistão cadeira
cadeira gamer
cadeira presidente
kit cadeira
```

Use negativo exact quando o termo isolado é ruim. Use phrase quando toda a família semântica é ruim.

## Payload de keyword

`sp_keywords` usa payload flat:

```json
{
  "campaign_id": "<campaign_id>",
  "ad_group_id": "<ad_group_id>",
  "keyword_text": "cadeira mocho",
  "match_type": "EXACT",
  "bid": 1.5,
  "state": "ENABLED"
}
```

Valores comuns de `match_type`:

```text
EXACT
PHRASE
BROAD
```

## Payload de target

`sp_targets` usa expression array:

```json
{
  "campaign_id": "<campaign_id>",
  "ad_group_id": "<ad_group_id>",
  "expression": [
    { "type": "ASIN_SAME_AS", "value": "B0XXXXXXXXX" }
  ],
  "bid": 0.85,
  "state": "ENABLED"
}
```

Tipos comuns:

```text
ASIN_SAME_AS
ASIN_CATEGORY_SAME_AS
```

## Score de priorização

Antes de criar, classifique cada item:

| Score | Critério |
|---|---|
| 5 | 3+ pedidos ou termo core óbvio com alto fit |
| 4 | long-tail altamente específico |
| 3 | termo relevante mas sem prova |
| 2 | termo amplo/ambíguo |
| 1 | alto risco de desperdício |

Crie primeiro scores 4–5. Scores 2–3 entram em discovery com bid menor. Score 1 vira negativo ou fica fora.

## Saída para campaign launcher

Entregue uma tabela final:

| target | tipo | match/expression | bid | motivo | confiança |
|---|---|---|---:|---|---:|
| cadeira mocho | keyword | EXACT | 1.50 | head core | 5 |
| mocho manicure | keyword | PHRASE | 0.95 | uso profissional | 4 |
| B0XXXXXXXXX | product target | ASIN_SAME_AS | 0.75 | concorrente direto | 4 |

## Erros comuns

- Misturar keyword com target: keyword vai em `sp_keywords`; ASIN vai em `sp_targets`.
- Enviar `KEYWORD_EXACT` para `sp_targets`; isso é errado.
- Criar broad junto com exact no mesmo ad group sem orçamento separado.
- Usar ASIN concorrente sem validar se é substituto real.
- Escalar termo com 1 pedido só.
- Definir bid sem preço, CVR ou target ACOS.
