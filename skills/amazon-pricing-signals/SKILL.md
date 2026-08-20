---
description: Sinais de precificacao Amazon BR — get_pricing_signals e get_price_bands. Resposta RAISE/HOLD/LOWER com bandas de oportunidade do nicho.
---

# Amazon Pricing Signals — MCP Titanos

Skill para: *"devo subir ou baixar o preço?"*, *"qual faixa de preço entrar?"*.

**Scopes:** `mining:search`

## Tools

| Tool | Créditos | Output |
|------|----------|--------|
| `get_pricing_signals` | 200 | RAISE / HOLD / LOWER por ASIN |
| `get_price_bands` | 250 | Heatmap de faixas + melhor banda |

## Exemplo — pricing signals

```json
{
  "pais": "BR",
  "asins": ["B0XXXXXXXXX"],
  "palavra_chave": "garrafa termica",
  "categoria_id": 123456
}
```

## Exemplo — price bands

```json
{
  "pais": "BR",
  "palavra_chave": "garrafa termica",
  "faixas": 5,
  "limite": 40
}
```

## Interpretação

| Sinal | Quando |
|-------|--------|
| **RAISE** | Preço abaixo da banda de oportunidade + rating competitivo |
| **HOLD** | Preço na banda ótima |
| **LOWER** | Acima da banda quente + pressão competitiva |

`indice_oportunidade` = vendas_medias / reviews_medios — **maior é melhor** para entrada.

## Fluxo composto

```text
get_price_bands → get_pricing_signals → (Seller) update_listing preco
```

## Pitfalls

- Informe `palavra_chave` + `categoria_id` para bandas precisas.
- Sinais não incluem COGS — validar margem nos simuladores Titanos (Gestão).
- Não confundir com preço Buy Box live sem `lookup_amazon_product`.
