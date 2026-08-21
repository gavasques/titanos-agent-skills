---
name: amazon-competitor-radar
description: Radar de concorrentes Amazon BR — get_market_pulse com alertas RED/YELLOW/GREEN. Monitora preco, BSR e reviews vs snapshot anterior. Use para monitorar preço, BSR e reviews vs o snapshot anterior.
license: MIT
---

# Amazon Competitor Radar — MCP Titanos

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Pergunta o que mudou no nicho (preço, BSR, reviews)
- Quer alertas de concorrentes RED/YELLOW/GREEN
- Precisa de market pulse vs um snapshot anterior

Skill para monitoramento operacional: *"o que mudou no meu nicho?"*, *"concorrente baixou preço?"*.

**Scope:** `mining:search` · **Requer organização** na API Key.

## Tool principal

| Tool | Créditos | Função |
|------|----------|--------|
| `get_market_pulse` | 100 | Diff vs último snapshot + alertas |

## Setup (primeira execução)

```json
{
  "pais": "BR",
  "asins": ["B0MEUASIN1", "B0CONCORR1", "B0CONCORR2"],
  "label": "watchlist-garrafa-termica",
  "salvar_snapshot": true
}
```

Primeira run = **baseline** (alertas INFO). Segunda run em diante = diff real.

## Alertas

| Nível | Exemplos |
|-------|----------|
| 🔴 RED | Preço -10%; BSR piora >50% |
| 🟡 YELLOW | Preço ±5–10%; BSR ±20–50%; novo player |
| 🟢 GREEN | BSR melhora; aceleração de reviews |

## Automação (agente local)

Agendar daily:

```text
get_market_pulse (mesmos ASINs, salvar_snapshot=true)
→ resumir alertas RED primeiro
→ sugerir get_pricing_signals nos ASINs próprios
```

## Complementos

- `get_market_overview` — contexto de mercado semanal
- `amazon-listing-audit` — gap vs concorrentes na conta seller
- `Amazon-Ads` — reação a movimentos de SERP/keywords

## Pitfalls

- Máximo **20 ASINs** por watchlist.
- Snapshots são por **organização** — keys da mesma org compartilham histórico.
- Não substitui alertas nativos Seller Central ou Ads.
