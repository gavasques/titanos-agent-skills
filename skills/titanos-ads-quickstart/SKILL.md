---
name: titanos-ads-quickstart
description: Use when an AI agent needs to start using Titanos MCP for Amazon Ads quickly, identify the right account/profile, inspect available Ads tools, validate scopes, and avoid common setup/discovery mistakes.
license: MIT
---

# Titanos Ads Quickstart

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Quer começar Amazon Ads via Titanos MCP sem descobrir endpoints na tentativa e erro
- Precisa identificar a conta/perfil Ads certo e validar scopes
- Pergunta quais tools Ads existem ou como evitar erros comuns de setup

## Objetivo

Dar ao agente o caminho mais curto para operar Amazon Ads via Titanos MCP sem ficar descobrindo endpoints por tentativa e erro.

Use esta skill antes de qualquer análise, criação, otimização ou auditoria de campanhas Amazon Ads.

## Princípio

Sempre descubra **quem**, **qual conexão**, **qual profile** e **quais recursos estão disponíveis** antes de consultar ou escrever dados.

Não invente métricas, IDs, campanhas, bids, budgets ou escopos.

## Primeiro minuto obrigatório

Execute nesta ordem:

1. `whoami`
   - confirma usuário, organização, plano, scopes e conexões ativas.
2. `list_ads_connections`
   - encontra conexões Amazon Ads.
3. `list_ads_profiles`
   - encontra o `profile_id` correto por marketplace/conta.
4. `list_resource_types`
   - descobre recursos Ads mutáveis e schemas.
5. `get_ads_report_metadata`
   - confirma cobertura/freshness dos relatórios antes de análise histórica.

Guarde sempre:

```text
connection_id = UUID da conexão ads_api
profile_id = ID numérico do perfil Amazon Ads
marketplace/country = país do profile
currency = moeda do profile
```

## Ferramentas essenciais (MCP 1.42.2 — testado 2026-07-06)

### ✅ Funcionando

| Necessidade | Tool Titanos MCP | Notas |
|---|---|---|
| Identidade/scopes | `whoami` | ✅ |
| Conexões Ads | `list_ads_connections` | ✅ |
| Perfis Ads | `list_ads_profiles` | ✅ |
| Tipos e schemas write | `list_resource_types` | ✅ |
| Campanhas SP | `get_campaigns` | ✅ evitar filtro `state` com contas grandes |
| Ad groups SP | `get_ad_groups` | ✅ |
| Product ads SP | `get_product_ads` | ✅ |
| Keywords/targets SP | `get_targets` | ✅ retorna `keywords[]` + `targets[]` |
| Negativos SP | `get_negatives` | ✅ |
| Criar recursos | `create_resources` | ✅ ver pitfalls abaixo |
| Atualizar recursos | `update_resources` | ✅ **corrigido 2026-07-06** — ver formato do payload abaixo |
| Deletar recursos | `delete_resources` | ✅ `sp_keywords` confirmado |
| Budget changes | `get_budget_changes` | ✅ |
| Entity history | `get_entity_history` | ✅ |
| Optimization drift | `get_optimization_drift` | ✅ |
| Optimization guide | `get_optimization_guide` | ✅ |
| Ads changelog | `list_ads_changelog` | ✅ |
| SB campaigns/ads | `get_sb_campaigns` etc | ✅ |
| SD campaigns/ads | `get_sd_campaigns` etc | ✅ |
| Report metadata | `get_ads_report_metadata` | ✅ |
| Report analyst (sem scope) | `ask_ads_report_analyst` | ✅ **não passar** `connection_id`/`profile_id` |
| Performance (com datas) | `get_ads_performance` | ✅ usar `start_date`/`end_date` |
| Search terms (com datas) | `get_search_term_report` | ✅ usar `start_date`/`end_date` |
| Placement (com datas) | `get_placement_report` | ✅ usar `start_date`/`end_date` |
| Advertised prod (com datas) | `get_advertised_product_report` | ✅ usar `start_date`/`end_date` |
| Recomendações | `list_ads_recommendations` | ✅ **corrigido 2026-07-06** — retorna vazio graceful se Amazon 403 |
| Bid guidance | `get_bid_guidance` | ✅ **corrigido 2026-07-06** — campo `strategy` default `CONVERSION_OPPORTUNITIES` |

### ⚠️ Limitações conhecidas (não bugs de código)

| Necessidade | Tool | Comportamento | Workaround |
|---|---|---|---|
| Recomendações (contas BR) | `list_ads_recommendations` | Retorna `[]` com nota (Amazon 403 no endpoint) | Limitação de permissão Amazon, não bug |
| Bid guidance sem dados | `get_bid_guidance` | `suggested_bid: null` para keywords sem histórico | Testar com keywords que têm tráfego |
| DSP (conta seller) | `get_dsp_campaigns` etc | 502 — DSP requer conta agency | Usar apenas em contas agency |
| PAT targets readback | `get_targets` | `targets[]` pode retornar vazio para campanhas PAT | Bug no filtro `adGroupIdFilter` da Amazon |

## Formato do payload — `update_resources` (IMPORTANTE)

O `update_resources` exige um wrapper `changes` dentro de cada resource. Não passe campos soltos.

### ✅ Formato correto

```json
{
  "resource_type": "sp_keywords",
  "resources": [
    {
      "keyword_id": "<keyword_id>",
      "changes": {
        "bid": 2.0
      }
    }
  ],
  "dry_run": true
}
```

### ❌ Formato errado (causa VALIDATION_ERROR 400)

```json
{
  "resource_type": "sp_keywords",
  "resources": [
    {
      "keyword_id": "<keyword_id>",
      "bid": 2.0
    }
  ]
}
```

O mesmo padrão se aplica a `sp_campaigns` e `sp_targets`:

```json
{
  "resource_type": "sp_campaigns",
  "resources": [
    {
      "campaign_id": "<campaign_id>",
      "changes": {
        "daily_budget": 60.0
      }
    }
  ],
  "dry_run": true
}
```

Campos mutáveis por resource type:

| Resource type | Campos mutáveis |
|---|---|
| `sp_campaigns` | `state`, `name`, `daily_budget`, `end_date`, `portfolio_id` |
| `sp_keywords` | `state`, `bid` |
| `sp_targets` | `state`, `bid` |

## Formato do payload — `get_bid_guidance`

O campo `strategy` é opcional (default `CONVERSION_OPPORTUNITIES`). Para `BIDS_FOR_EXISTING_AD_GROUP`, use `campaign_id` + `ad_group_id`. Para `BIDS_FOR_NEW_AD_GROUP`, use `asins`.

```json
{
  "recommendation_type": "BIDS_FOR_EXISTING_AD_GROUP",
  "campaign_id": "<campaign_id>",
  "ad_group_id": "<ad_group_id>",
  "targeting_expressions": [
    { "type": "KEYWORD_EXACT_MATCH", "value": "termo exemplo" }
  ]
}
```

Tipos válidos para `targeting_expressions`: `KEYWORD_EXACT_MATCH`, `KEYWORD_BROAD_MATCH`, `KEYWORD_PHRASE_MATCH`, `ASIN_SAME_AS`, etc.

## Escopo correto

A maioria das chamadas Ads precisa dos dois campos:

```json
{
  "connection_id": "<ads_connection_uuid>",
  "profile_id": 123456789012345
}
```

Não misture marketplaces. Se houver mais de um profile, escolha um por vez.

## Diagnóstico rápido de conta

Depois do discovery, rode:

```text
ask_ads_report_analyst:
"Últimos 30 dias completos: resumo Sponsored Products com spend, sales, purchases, ACOS, ROAS, impressions, clicks, CTR, CPC e CVR. Retorne uma linha total e top 10 campanhas por spend."
```

Use janela terminando em D-2 ou D-3 porque Amazon Ads pode ter lag de 1–3 dias.

## Regras de segurança

- Leitura/análise: pode executar direto.
- Write real: só após payload claro e confirmação explícita do usuário.
- Sempre faça `dry_run: true` antes de `dry_run: false`.
- Sempre use `idempotency_key` única por lote.
- Sempre faça readback após write.

## Cheatsheet de resource types SP

| Resource type | Cria o quê | Depende de |
|---|---|---|
| `sp_campaigns` | Campanhas Sponsored Products | profile |
| `sp_ad_groups` | Ad groups | campaign_id |
| `sp_product_ads` | Produto/SKU anunciado | campaign_id + ad_group_id + SKU |
| `sp_keywords` | Keywords manuais | campaign_id + ad_group_id |
| `sp_targets` | Product/category targeting | campaign_id + ad_group_id |
| `sp_negative_keywords` | Negativos no ad group | campaign_id + ad_group_id |
| `sp_campaign_negative_keywords` | Negativos na campanha | campaign_id |

## Erros comuns

| Sintoma | Causa provável | Ação |
|---|---|---|
| `SCOPE_MISSING` | API key sem scope | pedir nova key/scopes |
| `Invalid enum value` | pacote MCP antigo/cache npx | usar `@titanos/mcp-agents@1` e limpar cache |
| `VALIDATION_ERROR` no update | payload sem wrapper `changes` | usar `{ keyword_id, changes: { bid } }` não `{ keyword_id, bid }` |
| `VALIDATION_ERROR` no create | payload não bate com schema Amazon | rodar dry-run e revisar campos |
| `INTERNAL_ERROR (502)` | backend/write adapter falhou | readback antes de qualquer retry |
| `list_ads_recommendations` retorna `[]` | Amazon 403 para contas BR | limitação de permissão, não bug |
| `get_bid_guidance` retorna `suggested_bid: null` | keyword sem histórico suficiente | testar com keywords ativas |
| relatórios timeout/502 | query pesada | dividir por métrica/janela/campanha |

## Saída padrão para o usuário

Sempre responda com:

1. Conta/profile usado.
2. Janela de datas.
3. Métrica principal ou status.
4. O que foi feito ou bloqueado.
5. Próxima ação objetiva.
