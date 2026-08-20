---
description: Avaliacao de viabilidade de entrada em mercado Amazon BR — assess_market_entry, get_market_overview, get_price_bands. Veredito GO/CAUTION/AVOID com scoring 7 dimensoes.
---

# Amazon Market Entry — MCP Titanos

Skill para responder: *"vale a pena entrar neste nicho?"*, *"devo vender X na Amazon?"*.

**Pré-requisito:** `titanos-mcp-setup` com scope `mining:search`.

## Tools principais

| Tool | Créditos | Uso |
|------|----------|-----|
| `assess_market_entry` | 400 | **Workflow composto** — veredito + top oportunidades |
| `get_market_overview` | 250 | Agregados de mercado + scoring |
| `get_price_bands` | 250 | Faixas de preço + índice oportunidade |
| `find_categories` | 0 | Resolver `categoria_id` antes da análise |

## Fluxo recomendado

```text
find_categories (opcional)
  → assess_market_entry { palavra_chave, budget, experiencia }
  → lookup_amazon_product (top 3 ASINs)
  → analyze_reviews_structured (ASIN líder)
```

## Parâmetros `assess_market_entry`

```json
{
  "pais": "BR",
  "palavra_chave": "organizador cozinha",
  "categoria_id": 123456,
  "budget": "medio",
  "experiencia": "intermediario"
}
```

| Campo | Valores |
|-------|---------|
| `budget` | `baixo` \| `medio` \| `alto` |
| `experiencia` | `iniciante` \| `intermediario` \| `avancado` |

## Como ler o veredito

| Veredito | Score | Ação |
|----------|-------|------|
| ✅ GO | 70–100 | Validar ASINs finalistas + listing |
| ⚠️ CAUTION | 40–69 | Exigir diferenciação clara |
| 🔴 AVOID | 0–39 | Buscar subcategoria ou outro nicho |

## Labels de confiança

- 📊 **Data-backed** — métricas da amostra
- 🔍 **Inferido** — estimativa BSR ou tendência
- 💡 **Direcional** — recomendação estratégica

## Pitfalls

- Sempre informe `categoria_id` quando possível (via `find_categories`).
- Não trate `fonte_vendas=estimativa_bsr` como dado real.
- Veredito GO não substitui auditoria operacional (`audit_listings`) nem INPI.

## Próximos passos pós-GO

1. `super-listing-power` — gerar listing
2. `amazon-listing-audit` — gap operacional + competitivo
3. `listing-create-publish` — publicar (Seller conectado)
