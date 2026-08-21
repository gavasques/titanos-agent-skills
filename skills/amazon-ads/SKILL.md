---
name: amazon-ads
description: Guia para usar o servidor MCP Titanos (@titanos/mcp-agents) em contas Amazon Advertising — relatórios, campanhas SP/SB/SD, recomendações, mutações e PPC memory. Use para performance, bids, search terms, estrutura de campanhas e otimização via MCP.
license: MIT
---

# Amazon Ads — MCP Titanos

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Pergunta sobre performance, bids ou search terms de Amazon Ads
- Quer operar campanhas SP/SB/SD via Titanos MCP
- Precisa de relatórios, recomendações ou estrutura de campanhas

Você está conectado ao **MCP Titanos** (`@titanos/mcp-agents` → `https://www.titanos.com.br`). Autenticação: API Key `tnk_live_...` com scopes `ads:*` (e OAuth Amazon Ads ativo em Conta → Integrações).

## Setup (sempre)

1. `whoami` — plano, scopes da key, conexões ativas
2. `get_server_guide({ topic: "full" })` — mapa de tools
3. `list_brands` — marcas operacionais e perfis Ads vinculados
4. Se não houver marca: `list_amazon_ads_integrations` → `list_amazon_ads_integration_accounts`

## Conceitos Titanos

| AdPros (legado) | Titanos MCP |
|-----------------|-------------|
| `integration_id` | `connection_id` (UUID da conexão `ads_api`) |
| `account_id` (UUID perfil) | `profile_id` (número do perfil Amazon Ads) |
| `ask_report_analyst` | **`ask_ads_report_analyst`** |
| `list_resources` (leitura) | Tools dedicadas: `get_campaigns`, `get_ad_groups`, … |
| Recomendações granulares AdPros | `list_ads_recommendations` + `get_bid_guidance` |

Escopo quase sempre: `connection_id` + `profile_id` no body. Filtros via `filters: { campaign_id, ad_group_id, state, max_results }`.

## Tool selection

### Performance e relatórios (mais comum)

**`ask_ads_report_analyst`** — perguntas em linguagem natural sobre warehouse Ads (0 créditos). Datasets disponíveis:

| Dataset | Uso |
|---------|-----|
| `ads_campaign_performance` | ACOS, spend, vendas por campanha |
| `ads_search_terms` | Termos de busca, desperdício, harvest |
| `ads_placement` | Top of Search vs Product Pages |
| `ads_advertised_product` | Performance por ASIN |

Colunas no analyst: `spend`, `sales`, `purchases`, `acos`, `impressions`, `clicks` (equivalente a sales14d/purchases14d da API Amazon).

Alternativa determinística (sem LLM): `get_search_term_report`, `get_ads_performance`, `get_placement_report`, `get_advertised_product_report` com `start_date` / `end_date` ou `days`.

**Nota:** Reports com `days` agora funcionam (retornam status `PENDING` e processam em background ~45-120s). `start_date`/`end_date` também funcionam.

Metadados: `get_ads_report_metadata` · paginação: `ads_paged_query_result` / `ads_fetch_full_query_result`.

Exemplos de perguntas:

```
"Últimos 14 dias: top 20 campanhas por sales. Incluir spend, purchases, acos, impressions. Ordenar por sales desc."

"Últimos 30 dias: search terms com spend > 50 e purchases = 0. Incluir campaign_name, search_term, clicks, spend."

"Spend e sales diários nos últimos 30 dias (tendência)."
```

**Escopo:** uma combinação `connection_id` + `profile_id` por chamada. Para multi-perfil, use `list_brands` e confirme o marketplace com o usuário.

**Importante:** `ask_ads_report_analyst` funciona **sem** `connection_id`/`profile_id` (usa default da org). Passar esses campos causa `VALIDATION_ERROR (400)`.

### Estrutura de campanhas (leitura ao vivo)

Não existe `list_resources` unificado. Use:

| Tipo | Tool |
|------|------|
| Campanhas SP | `get_campaigns` |
| Ad groups SP | `get_ad_groups` (exige `campaign_id` em filters) |
| Keywords / product targets SP | `get_targets` |
| Negativos SP | `get_negatives` |
| Product ads SP | `get_product_ads` |
| SB | `get_sb_campaigns`, `get_sb_ad_groups`, `get_sb_ads` |
| SD | `get_sd_campaigns`, `get_sd_ad_groups`, `get_sd_product_ads` |

`list_resource_types` descreve tipos **mutáveis** (write) e aponta o `read_tool` correspondente.

### Recomendações Amazon

| Tool | Uso | Status |
|------|-----|--------|
| `list_ads_recommendations` | Oportunidades consolidadas (bid, budget, targeting) | ✅ Funciona. Retorna `[]` com nota se Amazon 403 (comum em contas BR) |
| `get_bid_guidance` | Bids sugeridos para ad group existente ou novo | ✅ Funciona. Campo `strategy` default `CONVERSION_OPPORTUNITIES` |

**`get_bid_guidance` — formato do payload:**

Para `BIDS_FOR_EXISTING_AD_GROUP`:
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

Para `BIDS_FOR_NEW_AD_GROUP`:
```json
{
  "recommendation_type": "BIDS_FOR_NEW_AD_GROUP",
  "asins": ["B0XXXXXXXXX"],
  "targeting_expressions": [
    { "type": "KEYWORD_EXACT_MATCH", "value": "termo exemplo" }
  ]
}
```

Tipos válidos de targeting expression: `KEYWORD_EXACT_MATCH`, `KEYWORD_BROAD_MATCH`, `KEYWORD_PHRASE_MATCH`, `ASIN_SAME_AS`, etc.

Pode retornar `suggested_bid: null` para keywords sem histórico suficiente — normal, não bug.

### PPC memory (histórico de mudanças)

| Tool | Uso |
|------|-----|
| `get_entity_history` | Histórico de campanha/target/negativo |
| `get_budget_changes` | Alterações de budget |
| `get_optimization_drift` | Drift de otimização vs baseline |

### Mutações (scope `ads:campaigns:write`)

| Tool | Uso |
|------|-----|
| `list_resource_types` | Schema dos tipos SP/SB/SD |
| `create_resources` / `update_resources` / `delete_resources` | CRUD em batch (máx. 5 itens) |
| `list_ads_changelog` / `get_ads_changeset` | Auditoria pós-mutação |

**Sempre** `dry_run: true` antes de aplicar. Mutações podem consumir créditos Titanos.

Tipos write SP: `sp_campaigns`, `sp_ad_groups`, `sp_keywords`, `sp_targets`, `sp_negative_keywords`, `sp_campaign_negative_keywords`, `sp_product_ads`. SB/SD: `sb_*`, `sd_*`.

**`update_resources` — formato do payload (IMPORTANTE):**

O payload exige wrapper `changes` dentro de cada resource. Campos soltos causam `VALIDATION_ERROR (400)`.

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

Campos mutáveis:

| Resource type | id_field | Campos mutáveis |
|---|---|---|
| `sp_campaigns` | `campaign_id` | `state`, `name`, `daily_budget`, `end_date`, `portfolio_id` |
| `sp_keywords` | `keyword_id` | `state`, `bid` |
| `sp_targets` | `target_id` | `state`, `bid` |

Limitações vs AdPros:

- Estado de campanha: apenas **ENABLED** ou **PAUSED** (não ARCHIVED via API).
- **Regras nativas** (`sp_budget_rules`, `sp_optimization_rules`, etc.) — **não expostas** no MCP Titanos hoje; use `update_resources` / playbook em `get_optimization_guide`.
- Metadados de produto Ads (`get_amazon_ads_product_metadata`) — use `lookup_amazon_product` (dados de mercado) ou catálogo seller.

### DSP

Leitura: `get_dsp_campaigns`, `get_dsp_ad_groups`, `get_dsp_line_items`, `get_dsp_creatives`. Write: `create_dsp_resources`, `update_dsp_resources`, `delete_dsp_resources`. Ver skill `amazon-dsp`.

**Gap:** DSP requer conta agency. Contas seller recebem erro. `ask_ads_report_analyst` ainda **não** inclui datasets DSP no warehouse — performance DSP via API live + skill DSP, não via analyst NL.

### Marcas e guias

| Tool | Uso |
|------|-----|
| `list_brands` / `create_brand` / `assign_profiles_to_brand` | Agrupamento operacional |
| `get_optimization_guide` | Playbook PPC (tópicos: overview, guardrails, harvest, diagnose) |

## Fluxos comuns

### Saúde da conta

1. `list_brands` → escolher `connection_id` + `profile_id`
2. `ask_ads_report_analyst`: resumo 14d (spend, sales, acos)
3. `list_ads_recommendations`

### Desperdício em search terms

1. `ask_ads_report_analyst` no dataset `ads_search_terms` (ou skill `wasted-ad-spend-dashboard`)
2. Negativos: `create_resources` com `sp_negative_keywords` após confirmação

### Expansão de keywords

1. `get_campaigns` → `get_ad_groups`
2. `list_ads_recommendations` / `get_bid_guidance`
3. `ask_ads_report_analyst`: termos convertendo que ainda não são exact

### Filtro por budget/bid

`get_campaigns` / `get_targets` trazem valores **ao vivo**. Para histórico agregado, use `ask_ads_report_analyst`.

### Ajustar bid de keyword

1. `get_targets` para listar keywords do ad group
2. `update_resources` com `sp_keywords`:
```json
{
  "resource_type": "sp_keywords",
  "resources": [
    { "keyword_id": "...", "changes": { "bid": 2.5 } }
  ],
  "dry_run": true
}
```
3. Confirmar preflight (live vs planned)
4. Executar com `dry_run: false`

## Dicas

- Relatórios Ads têm lag de 1–3 dias; evite "hoje" sem avisar.
- ACOS = spend / sales × 100. ROAS = sales / spend.
- Confirme marketplace quando há vários perfis na mesma marca.
- Conexão: **Conta → Integrações** (aba Amazon, Ads OAuth) + API Key com preset **Amazon Ads completo** (criada em **Conta → Integrar com IA**).
- `update_resources` exige wrapper `changes` — campos soltos causam 400.
- `get_bid_guidance` pode retornar `suggested_bid: null` para keywords sem histórico.
- `list_ads_recommendations` retorna vazio para contas BR (Amazon 403) — limitação de permissão, não bug.
