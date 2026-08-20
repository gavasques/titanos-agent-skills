---
description: Inteligencia de keywords Amazon BR via ABA — expand_keywords, reverse_asin_keywords, get_keyword_serp, thanos_rank_keywords. Expansao, reverse ASIN e proxy SERP.
---

# Amazon Keyword Intelligence — MCP Titanos

Skill para pesquisa de keywords: expansão, reverse ASIN, rank ABA e composição de SERP.

**Scopes:** `mining:keywords_rank:read` + `mining:search`

## Tools

| Tool | Scope | Créditos |
|------|-------|----------|
| `thanos_rank_keywords` | keywords_rank | 0 |
| `expand_keywords` | keywords_rank | 0 |
| `reverse_asin_keywords` | keywords_rank | 0 |
| `get_keyword_serp` | mining:search | 200 |

## Fluxos

### Expansão long-tail

```json
{ "seed": "tapete yoga", "limit": 25 }
```

### Reverse ASIN (traffic terms ABA)

```json
{ "asin": "B0XXXXXXXXX", "limit": 20 }
```

### Proxy SERP

```json
{
  "pais": "BR",
  "palavra_chave": "tapete yoga",
  "categoria_id": 123456,
  "limite": 20
}
```

## Ordem recomendada

```text
expand_keywords → get_keyword_serp (top terms)
reverse_asin_keywords (ASIN referência)
thanos_rank_keywords (rank histórico ABA)
```

## Limitações

- Expansão/reverse usam **ABA ingestido** — termos fora do ABA retornam vazio (não é erro).
- `get_keyword_serp` é **proxy** via Product Finder (não snapshot live org/ads).
- Plano precisa incluir área `card.mineracao.aba` para tools ABA.

## Cruzamento com Ads

Keywords validadas → skill `Amazon-Ads` / `search_term_harvest` para campanhas.
