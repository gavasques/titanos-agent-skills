---
description: Pesquisa de mercado Amazon BR via MCP Titanos — find_categories, search_products, get_market_overview, assess_market_entry, get_price_bands, get_market_pulse. Use para nicho, validação, scoring GO/CAUTION/AVOID e monitoramento.
---

# Mineração Amazon — MCP Titanos

Pesquisa de mercado com **dados de mercado** (provider interno Titanos). Complementa Seller Central e Ads — não substitui conta conectada.

**Pré-requisito:** skill `titanos-mcp-setup` (scope `mining:search` + opcional `amazon:lookup` + `mining:keywords_rank:read`).

## Tools — camada base

| Tool | Scope | Créditos (aprox.) |
|------|-------|-------------------|
| `find_categories` | `mining:search` | **0** |
| `search_products` | `mining:search` | 200 / 500 com detalhes |
| `get_best_sellers` | `mining:search` | 100 / 400 |
| `find_private_label_opportunities` | `mining:search` | 300 / 600 |
| `lookup_amazon_product` | `amazon:lookup` | 100 (cache miss) |

## Tools — inteligência de mercado (novo)

| Tool | Créditos | Uso |
|------|----------|-----|
| `get_market_overview` | 250 | Agregados + scoring |
| `assess_market_entry` | 400 | Veredito GO/CAUTION/AVOID |
| `get_price_bands` | 250 | Faixas de preço |
| `get_pricing_signals` | 200 | RAISE/HOLD/LOWER |
| `get_market_pulse` | 100 | Radar concorrentes |
| `analyze_reviews_structured` | reviews API | Pain points |
| `expand_keywords` / `reverse_asin_keywords` | 0 | Keywords ABA |
| `get_keyword_serp` | 200 | Proxy SERP |

Todas as respostas podem incluir `orientacao_agente` — **leia antes de interpretar vendas**.

## Labels de confiança

| Label | Significado |
|-------|-------------|
| 📊 | Dado direto da amostra |
| 🔍 | Inferido (ex.: BSR → vendas) |
| 💡 | Recomendação estratégica |

## Ordem recomendada (fluxo padrão)

```text
find_categories (opcional)
  → assess_market_entry OU search_products
  → get_price_bands
  → lookup_amazon_product (1–3 finalistas)
  → analyze_reviews_structured
  → super-listing-power
```

## Fluxo — validação pré-lançamento

1. `assess_market_entry` com budget/experiencia
2. Se GO: `lookup_amazon_product` top 3
3. `analyze_reviews_structured` no líder
4. Oferecer `invoke_listing_power`

## Fluxo — monitoramento

1. `get_market_pulse` com ASINs próprios + concorrentes
2. Agendar daily no agente
3. Alertas RED → `get_pricing_signals` + `Amazon-Ads`

## Presets `search_products`

| Preset | Perfil |
|--------|--------|
| `hot_picks` | Alto giro, reviews, faixa média |
| `low_competition` | Poucos sellers |
| `rising_stars` | Crescimento 90d |
| `amazon_absent` | Sem Amazon na buy box |

## Pitfalls

- `incluir_detalhes: true` com `limite > 30` — rejeitado.
- Nunca trate `estimativa_bsr` como fato.
- Sigilo: use "dados de mercado" / Titanos — nunca codinome upstream.

## Skills relacionadas (download)

- `amazon-market-entry` — viabilidade de nicho
- `amazon-pricing-signals` — precificação
- `amazon-keyword-intelligence` — keywords ABA
- `amazon-competitor-radar` — monitoramento
