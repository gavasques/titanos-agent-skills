---
name: amazon-dsp
description: Amazon DSP via MCP Titanos — leitura get_dsp_*, write create/update/delete_dsp_resources. Performance histórica via ask_ads_report_analyst ainda não cobre DSP no warehouse; use API live + skill Amazon-Ads para SP.
---

# Amazon DSP — MCP Titanos

DSP (programmatic) no Titanos: leitura/escrita via Advertising API em perfis **agency** (`account_type` agency no perfil Ads).

## Pré-requisitos

1. OAuth Amazon Ads ativo + scope `ads:campaigns:read` / `write`
2. `list_amazon_ads_integration_accounts` — filtrar perfil agency
3. `connection_id` + `profile_id` em todas as chamadas
4. Identificar `advertiser_id` DSP antes de campanhas/line items

## Hierarquia (API Titanos)

```
profile (agency)
  → get_dsp_campaigns        (filters.advertiser_id)
  → get_dsp_ad_groups        (advertiser_id + campaign_id opcional)
  → get_dsp_line_items       (advertiser_id + ad_group_id)
  → get_dsp_creatives        (advertiser_id — nível advertiser)
```

Targets DSP: não há tool `dsp_targets` separada no MCP v1 — use line items e creatives.

## Tools

| Ação | Tool |
|------|------|
| Listar campanhas / ad groups / line items / creatives | `get_dsp_*` |
| Atualizar entidades | `update_dsp_resources` (`dsp_campaigns`, `dsp_ad_groups`, `dsp_line_items`, `dsp_creatives`) |
| Criar | `create_dsp_resources` |
| Desassociar creative↔ad group | `delete_dsp_resources` (tipo `dsp_creatives`) |

Sempre `dry_run: true` antes de write.

## Performance e relatórios — gap importante

A skill AdPros original usa `ask_report_analyst` com report types `dsp_campaign_performance`, `dsp_products`, etc.

**No Titanos hoje:** `ask_ads_report_analyst` cobre apenas datasets **Sponsored Products** (`ads_campaign_performance`, `ads_search_terms`, `ads_placement`, `ads_advertised_product`). **Não** inclui warehouse DSP.

**Alternativas:**

- Métricas agregadas recentes: combinar respostas de `get_dsp_campaigns` / line items (campos de status/budget quando expostos pela API)
- Para análise NL profunda DSP: aguardar ingestão warehouse ou export manual + análise offline
- Não inventar NTB%, halo ou geography sem dados reais

Quando o warehouse DSP existir, atualizar esta skill para usar `ask_ads_report_analyst` com os novos datasets.

## Fluxos (estrutura + operação)

### Health check estrutural

1. Resolver agency profile
2. `get_dsp_campaigns` com `advertiser_id`
3. Resumir campanhas ENABLED vs PAUSED

### Browse line items de uma campanha

1. `get_dsp_ad_groups` (campaign_id)
2. `get_dsp_line_items` (ad_group_id)

### Creatives do advertiser

`get_dsp_creatives` — biblioteca no nível advertiser; cruzar com ad groups manualmente.

## Pitfalls Titanos

- Não misturar advertisers numa mesma query
- DSP write ≠ pausar Sponsored Products — especificar plataforma
- Lag 1–3 dias em relatórios Amazon (quando warehouse existir)
- "Order" no relatório DSP ≈ campanha em outras interfaces

## Dicas

- `list_brands` para contexto multi-conta
- PPC memory: `get_entity_history` inclui `dsp_campaigns` em mutações registradas
- SP performance: skill `Amazon-Ads` + `ask_ads_report_analyst`