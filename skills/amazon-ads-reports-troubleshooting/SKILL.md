---
name: amazon-ads-reports-troubleshooting
description: Use when an AI agent needs to pull Amazon Ads reports through Titanos MCP and encounters errors, timeouts, or needs to choose between report tools, date formats, and scope parameters.
license: MIT
---

# Amazon Ads Reports Troubleshooting — Titanos MCP

## Quando usar

Use quando precisar puxar relatórios de Amazon Ads via Titanos MCP e encontrar erros, timeouts, ou precisar escolher entre ferramentas de relatório, formatos de data e parâmetros de escopo.

## Resumo rápido (MCP 1.42.2 — testado 2026-07-06)

### ✅ Funciona

| Tool | Como usar | Notas |
|---|---|---|
| `get_ads_report_metadata` | `{connection_id, profile_id}` | ✅ rápido, confirma cobertura |
| `ask_ads_report_analyst` | `{question: "..."}` | ✅ **NÃO passar** `connection_id`/`profile_id` |
| `get_ads_performance` | `{connection_id, profile_id, start_date, end_date}` | ✅ assíncrono (PENDING → pronto em ~45-120s) |
| `get_search_term_report` | `{connection_id, profile_id, start_date, end_date}` | ✅ assíncrono |
| `get_placement_report` | `{connection_id, profile_id, start_date, end_date}` | ✅ assíncrono |
| `get_advertised_product_report` | `{connection_id, profile_id, start_date, end_date}` | ✅ assíncrono |
| `get_ads_performance` com `days` | `{connection_id, profile_id, days: 7}` | ✅ **corrigido 2026-07-06** — retorna PENDING |
| `list_ads_recommendations` | `{connection_id, profile_id}` | ✅ **corrigido 2026-07-06** — retorna `[]` com nota se Amazon 403 |
| `get_bid_guidance` | `{connection_id, profile_id, recommendation_type, targeting_expressions, ...}` | ✅ **corrigido 2026-07-06** — campo `strategy` default |

### ⚠️ Limitações conhecidas (não bugs)

| Tool | Comportamento | Workaround |
|---|---|---|
| `ask_ads_report_analyst` com `connection_id` + `profile_id` | `VALIDATION_ERROR (400)` | omitir scope params |
| `list_ads_recommendations` em contas BR | Retorna `[]` com nota (Amazon 403) | limitação de permissão Amazon |
| `get_bid_guidance` para keywords sem histórico | `suggested_bid: null` | testar com keywords ativas |

## Regras de data

### Formato correto

```json
{
  "start_date": "2026-06-01",
  "end_date": "2026-06-30"
}
```

Também aceita `days` (ex: `days: 7`):

```json
{
  "days": 7
}
```

### Cálculo de janela

```python
from datetime import date, timedelta
end = date(2026, 7, 5)
start = end - timedelta(days=7)
# start_date = "2026-06-28"
# end_date = "2026-07-05"
```

### Lag de dados

Amazon Ads tem lag de 2-3 dias. Se hoje é 5 de julho, o último dia com dados completos é 2 ou 3 de julho.

Para relatórios de "últimos 7 dias", use:

```text
end_date = hoje - 3 dias
start_date = end_date - 7 dias
```

## Escolha de ferramenta por necessidade

| Pergunta | Tool | Tempo |
|---|---|---|
| "Total de spend, sales, ACOS?" | `ask_ads_report_analyst` (sem scope) | ~2s |
| "Performance por campanha?" | `get_ads_performance` com datas ou `days` | ~45-120s |
| "Quais search terms converteram?" | `get_search_term_report` com datas ou `days` | ~45-120s |
| "Performance por placement?" | `get_placement_report` com datas ou `days` | ~45-120s |
| "KPIs por ASIN?" | `get_advertised_product_report` com datas ou `days` | ~45-120s |
| "Que dados estão disponíveis?" | `get_ads_report_metadata` | ~1s |
| "Recomendações de otimização?" | `list_ads_recommendations` | ~2s (pode retornar vazio) |
| "Sugestão de bid?" | `get_bid_guidance` | ~2s |

## `ask_ads_report_analyst` — armadilhas

### ✅ Funciona

```json
{
  "question": "Total spend, sales14d, ACOS, impressions, clicks for last 7 days"
}
```

### ❌ Falha

```json
{
  "connection_id": "<ads_connection_uuid>",
  "profile_id": 123456789012345,
  "question": "Total spend, sales14d, ACOS, impressions, clicks for last 7 days"
}
```

O `ask_ads_report_analyst` usa o profile default da organização quando não recebe scope explícito. Passar `connection_id` + `profile_id` causa `VALIDATION_ERROR (400)`.

Se precisar de scope específico e o default não servir, use `get_ads_performance` com datas explícitas.

## Reports assíncronos

Os reports de performance, search terms, placement e advertised product são assíncronos. A chamada retorna imediatamente com `status: "PENDING"` e um `report_id`. O processamento acontece em background na Amazon (~45-120s).

Não é necessário polling — o resultado é retornado na mesma chamada quando pronto. Se retornar PENDING, aguarde e tente novamente.

## Estratégia anti-timeout

1. Use `ask_ads_report_analyst` (sem scope) para análises rápidas — é mais rápido.
2. Para dados determinísticos, use `start_date`/`end_date` ou `days`.
3. Use janelas curtas (7 dias) em vez de 30+ dias quando possível.
4. Se um relatório timeout, reduza a janela para 3 dias e tente novamente.

## Modelo de resposta para usuário

```text
Janela: <start_date> a <end_date> (lag de 2-3 dias)
Conta: <account_name> (profile_id: <pid>)

Spend: R$ X
Sales: R$ Y
ACOS: Z%
ROAS: N.Nx
Impressões: N
Cliques: N
CTR: X%
CPC: R$ X

Fonte: <tool usado> (<tempo>s)
```
