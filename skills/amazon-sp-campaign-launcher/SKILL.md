---
name: amazon-sp-campaign-launcher
description: Use when an AI agent needs to create Sponsored Products campaigns through Titanos MCP, especially new product launches, AUTO/KW/PAT structures, ASIN/SKU validation, dry-run/write execution, and safe partial recovery.
---

# Amazon SP Campaign Launcher — Titanos MCP

## Objetivo

Criar campanhas Sponsored Products do zero com segurança via Titanos MCP.

Esta skill transforma um pedido como “crie campanhas para estes ASINs” em uma operação controlada: valida produto, monta estrutura, calcula bids, roda dry-run, pede aprovação, cria em ordem e verifica o resultado.

## Quando usar

Use quando o usuário pedir:

- Criar campanha Sponsored Products.
- Lançar campanhas para um ou mais ASINs/SKUs.
- Criar estrutura AUTO + keyword + product targeting.
- Subir campanha de lançamento com budgets e bids.
- Continuar criação parcial que falhou no meio.

Não use para otimização semanal ou dashboards de desperdício; nesses casos use skills de optimization/harvest/waste.

## Inputs mínimos

Antes de escrever qualquer coisa, obtenha:

| Input | Obrigatório? | Como obter |
|---|---:|---|
| `connection_id` | sim | `list_ads_connections` |
| `profile_id` | sim | `list_ads_profiles` |
| ASIN(s) | sim | usuário ou catálogo |
| SKU(s) seller | sim | listing/inventory Seller |
| Preço | sim | listing/catalog |
| Estoque | recomendado | inventory |
| Target ACOS | recomendado | usuário ou default conservador |
| Orçamento diário | sim | usuário ou proposta confirmada |

Se margem/COGS não estiver disponível, deixe claro que bids são de lançamento conservador, não break-even real.

## Validação antes da campanha

Para cada ASIN:

1. Validar que o produto existe.
2. Validar que pertence ao seller/anunciante.
3. Recuperar SKU seller.
4. Confirmar preço.
5. Confirmar buyable/listing ativo.
6. Confirmar estoque suficiente.

Não crie product ads só com ASIN. Em `sp_product_ads`, envie também `sku`.

## Estrutura padrão recomendada

Para lançamento comum:

| Campanha | Objetivo | Budget inicial | Targeting |
|---|---|---:|---|
| `SP | <Marca> | <Produto> | AUTO | Discovery | <YYYYMM>` | descobrir termos e ASINs | 20–30% | AUTO |
| `SP | <Marca> | <Produto> | KW | Core | <YYYYMM>` | termos principais | 40–60% | keywords exact/phrase |
| `SP | <Marca> | <Produto> | PAT | Concorrentes | <YYYYMM>` | ASINs concorrentes | 20–30% | product targeting |

Use `ENABLED` apenas se o usuário aprovar explicitamente. Caso contrário, crie `PAUSED`.

## Cálculo de bid inicial

Fórmula de CPC máximo:

```text
max_cpc = preço_do_produto × CVR_estimado × target_ACOS
```

Exemplo para produto de R$299:

| CVR | Target ACOS 25% | CPC máximo |
|---:|---:|---:|
| 1.0% | 25% | R$0,75 |
| 1.5% | 25% | R$1,12 |
| 2.0% | 25% | R$1,50 |
| 2.5% | 25% | R$1,87 |

Regras práticas:

- AUTO: bid médio conservador.
- Exact core: maior bid.
- Phrase: 60–80% do exact.
- PAT concorrente: menor que exact, salvo concorrente muito validado.
- Termo/ASIN só é “validado” com 3+ pedidos. 1–2 pedidos = watchlist/teste.

## Ordem de criação obrigatória

Nunca crie tudo no escuro em um batch único.

1. `sp_campaigns`
2. Readback com `get_campaigns` por nome/ID.
3. `sp_ad_groups`
4. Readback com `get_ad_groups`.
5. `sp_product_ads`
6. Readback com `get_product_ads`.
7. `sp_keywords` e/ou `sp_targets`.
8. `sp_negative_keywords` / `sp_campaign_negative_keywords`.
9. Readback final.

## Payloads base

### 1. Campaigns

Use `start_date` explícito em ISO `YYYY-MM-DD`.

```json
{
  "resource_type": "sp_campaigns",
  "dry_run": true,
  "resources": [
    {
      "name": "SP | BKZA | Mocho | AUTO | Discovery | 202607",
      "targeting_type": "AUTO",
      "daily_budget": 30,
      "state": "ENABLED",
      "start_date": "2026-07-05"
    }
  ]
}
```

### 2. Ad groups

```json
{
  "resource_type": "sp_ad_groups",
  "dry_run": true,
  "resources": [
    {
      "campaign_id": "<campaign_id>",
      "name": "ADG | Produto | Core KW",
      "default_bid": 1.0,
      "state": "ENABLED"
    }
  ]
}
```

### 3. Product ads

Inclua `sku` junto com `asin`.

```json
{
  "resource_type": "sp_product_ads",
  "dry_run": true,
  "resources": [
    {
      "campaign_id": "<campaign_id>",
      "ad_group_id": "<ad_group_id>",
      "asin": "B0XXXXXXXXX",
      "sku": "SKU123",
      "state": "ENABLED"
    }
  ]
}
```

### 4. Keywords

`sp_keywords` usa payload flat.

```json
{
  "resource_type": "sp_keywords",
  "dry_run": true,
  "resources": [
    {
      "campaign_id": "<campaign_id>",
      "ad_group_id": "<ad_group_id>",
      "keyword_text": "cadeira mocho",
      "match_type": "EXACT",
      "bid": 1.5,
      "state": "ENABLED"
    }
  ]
}
```

### 5. Product targets

`sp_targets` usa `expression`.

```json
{
  "resource_type": "sp_targets",
  "dry_run": true,
  "resources": [
    {
      "campaign_id": "<campaign_id>",
      "ad_group_id": "<ad_group_id>",
      "expression": [
        { "type": "ASIN_SAME_AS", "value": "B0YYYYYYYY" }
      ],
      "bid": 0.85,
      "state": "ENABLED"
    }
  ]
}
```

## Dry-run e confirmação

Antes de write real:

1. Execute `dry_run: true` para cada resource type.
2. Mostre ao usuário:
   - nomes das campanhas,
   - budgets,
   - estado `ENABLED`/`PAUSED`,
   - ASINs/SKUs,
   - total de keywords/targets,
   - impacto máximo diário.
3. Só execute `dry_run: false` após aprovação explícita.

## Idempotência e lotes

- Use `idempotency_key` única por lote.
- Batch máximo seguro: 5 recursos por chamada.
- Nomeie chaves por projeto e etapa:

```text
<slug>-<yyyymmdd>-campaigns-1
<slug>-<yyyymmdd>-adgroups-1
<slug>-<yyyymmdd>-productads-1
<slug>-<yyyymmdd>-keywords-1
<slug>-<yyyymmdd>-targets-1
```

## Recovery se falhar no meio

Se qualquer write retornar 400/404/409/502:

1. Pare.
2. Faça readback por nomes/IDs exatos.
3. Registre o que já foi criado.
4. Não repita o batch inteiro sem saber o estado.
5. Continue apenas da etapa pendente.

Exemplos de armadilhas reais:

| Erro | Causa | Correção |
|---|---|---|---|
| campaign 502 | `start_date`/budget mal normalizados | usar `start_date: YYYY-MM-DD` e budget diário correto |
| `merchantSku is empty` | product ad sem SKU | enviar `sku` + `asin` |
| idempotency conflict | mesma chave com payload diferente | usar nova chave e readback antes |
| keyword 400/502 | endpoint/payload errado | `sp_keywords` deve ir para keywords com `keyword_text` + `match_type` |
| target 404 | ASIN/target inválido ou endpoint/payload errado | validar ASIN concorrente e usar `expression` |

## Status de bugs (MCP 1.42.2 — atualizado 2026-07-06)

### ✅ `update_resources` — funcionando (corrigido 2026-07-06)

O payload exige wrapper `changes`. Campos soltos causam 400.

```text
update_resources sp_campaigns:
  { campaign_id, changes: { daily_budget: 65 } }     → ✅
  { campaign_id, changes: { state: "PAUSED" } }     → ✅

update_resources sp_keywords:
  { keyword_id, changes: { bid: 1.55 } }             → ✅
  { keyword_id, changes: { state: "PAUSED" } }       → ✅
```

### ✅ Reports com `days` — funcionando (corrigido 2026-07-06)

`days: 7` retorna status `PENDING` e processa em background (~45-120s). Também aceita `start_date`/`end_date`.

### ⚠️ `ask_ads_report_analyst`

- ✅ funciona **sem** `connection_id`/`profile_id`
- ❌ falha **com** scope explícito

### ✅ `list_ads_recommendations` — funcionando (corrigido 2026-07-06)

Retorna `[]` com nota explicativa se Amazon 403 (comum em contas BR). Não é bug — é limitação de permissão.

### ✅ `get_bid_guidance` — funcionando (corrigido 2026-07-06)

Campo `strategy` adicionado com default `CONVERSION_OPPORTUNITIES`. Pode retornar `suggested_bid: null` para keywords sem histórico.

### ❌ DSP

Retorna erro para contas seller. DSP exige perfil `agency`.

### ✅ Bugs já corrigidos

| Bug | Status | Fix |
|---|---|---|
| `sp_campaigns` write 502 | ✅ corrigido | `start_date` ISO + budget nested |
| `sp_product_ads` write 502 | ✅ corrigido | incluir `sku` no payload |
| `sp_keywords` create 502 | ✅ corrigido | rotear para `/sp/keywords/list` |
| `sp_targets` create 400 | ✅ corrigido | rotear para `/sp/targets` |
| `delete_resources sp_keywords` schema | ✅ corrigido | enum atualizado para 1.42.2 |
| `get_targets` readback KW | ✅ corrigido | retorna `keywords[]` + `targets[]` |
| `update_resources` SP 400 | ✅ corrigido 2026-07-06 | `id_field` + `.passthrough()` + wrapper `changes` |
| `list_ads_recommendations` 502 | ✅ corrigido 2026-07-06 | GET→POST + graceful 403 |
| `get_bid_guidance` 502 | ✅ corrigido 2026-07-06 | campo `strategy` adicionado |
| Reports com `days` 502 | ✅ corrigido 2026-07-06 | agora retorna PENDING |
| `get_targets` readback PAT | ⚠️ pendente | `targets[]` ainda retorna vazio mesmo com DUPLICATE_VALUE |

## Readback final obrigatório

Ao terminar, confirme:

- 3 campanhas existentes com estado e budget.
- 1 ad group por campanha.
- product ads para todos os SKUs.
- keywords criadas na campanha KW.
- ASIN targets criados na campanha PAT.
- negativos criados.

## Resposta final ideal

```text
Criado:
- Campanhas: X
- Ad groups: X
- Product ads: X
- Keywords: X
- Product targets: X
- Negativos: X
Budget diário máximo: R$X
Estado: ENABLED/PAUSED
Pendências: nenhuma / listar erros reais
```
