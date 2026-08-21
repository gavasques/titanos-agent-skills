---
name: amazon-ads-write-recovery
description: Use when an AI agent is applying Amazon Ads mutations through Titanos MCP and must use dry-run, idempotency, readback, partial-write recovery, duplicate prevention, or backend error troubleshooting.
license: MIT
---

# Amazon Ads Write Recovery — Titanos MCP

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Is applying Ads mutations and a write failed mid-way
- Needs dry-run, idempotency, or readback before/after writes
- Wants to avoid duplicate campaigns or recover a partial write

## Objetivo

Evitar que agentes dupliquem campanhas, keywords, targets ou product ads quando um write via Titanos MCP falha no meio.

Use esta skill para qualquer mutação Amazon Ads com `create_resources`, `update_resources` ou `delete_resources`.

## Regra principal

**Depois de erro em write, nunca reexecute o mesmo lote às cegas.**

Faça readback primeiro. Erro 502/timeout pode acontecer depois de a Amazon ter criado parte dos recursos.

## Antes do write

1. Confirmar conta:
   - `connection_id`
   - `profile_id`
   - marketplace
2. Confirmar escopo/schemas:
   - `list_resource_types`
   - `tools/list` se disponível
3. Rodar `dry_run: true`.
4. Mostrar payload e impacto ao usuário.
5. Receber confirmação explícita.
6. Executar `dry_run: false` em lotes pequenos.

## Lote seguro

Use no máximo 5 itens por batch.

Exemplo de idempotency keys:

```text
bkza-mocho-20260705-campaigns-1
bkza-mocho-20260705-adgroups-1
bkza-mocho-20260705-productads-1
bkza-mocho-20260705-keywords-1
bkza-mocho-20260705-targets-1
bkza-mocho-20260705-negatives-1
```

Não reutilize a mesma `idempotency_key` com payload diferente. Se precisar mudar payload, use nova chave e faça readback antes.

## Sequência de criação SP recuperável

| Etapa | Write | Readback |
|---|---|---|
| 1 | `sp_campaigns` | `get_campaigns` por nome/ID |
| 2 | `sp_ad_groups` | `get_ad_groups` por campaign/ad group |
| 3 | `sp_product_ads` | `get_product_ads` por campaign/ad group |
| 4 | `sp_keywords` | `get_targets` / target listing por campaign/ad group |
| 5 | `sp_targets` | `get_targets` por campaign/ad group |
| 6 | negativos | `get_negatives` |

## Como lidar com erro

### Caso 1: `INTERNAL_ERROR (502)` ou timeout

Ação:

1. Pare.
2. Faça readback por nome/ID exato.
3. Compare o plano com o que já existe.
4. Continue apenas os recursos faltantes.
5. Registre o erro real para o dev.

Nunca diga "não criou nada" sem readback.

### Caso 2: `IDEMPOTENCY_CONFLICT (409)`

Causa:

- mesma `idempotency_key` usada com payload diferente.

Ação:

1. Readback do estado atual.
2. Gerar nova `idempotency_key` para o payload corrigido.
3. Criar só o que falta.

### Caso 3: `VALIDATION_ERROR (400)`

Ação:

1. Ler erro de campo/caminho.
2. Comparar com schema do `list_resource_types`.
3. Confirmar se payload do MCP precisa ser flat ou nested.

**IMPORTANTE — `update_resources` exige wrapper `changes`:**

O payload de update deve ter a estrutura:

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

NÃO passe campos soltos (`bid`, `state`) no nível do resource — eles devem estar dentro de `changes`. Passar `bid` solto causa `VALIDATION_ERROR (400)`.

O mesmo padrão vale para `sp_campaigns` (`changes: { daily_budget, state, name }`) e `sp_targets` (`changes: { bid, state }`).

Exemplos reais de formato correto:

```text
update_resources sp_campaigns:
  { campaign_id: "<campaign_id>", changes: { daily_budget: 65 } }     → ✅
  { campaign_id: "<campaign_id>", changes: { state: "PAUSED" } }    → ✅
  { campaign_id: "<campaign_id>", changes: { name: "TEST" } }       → ✅

update_resources sp_keywords:
  { keyword_id: "<keyword_id>", changes: { bid: 1.55 } }            → ✅
  { keyword_id: "<keyword_id>", changes: { state: "PAUSED" } }     → ✅

update_resources sp_targets:
  { target_id: "<target_id>", changes: { bid: 1.55 } }             → ✅
  { target_id: "<target_id>", changes: { state: "PAUSED" } }      → ✅
```

Outros exemplos de validação:

- `sp_campaigns`: `start_date` precisa ISO `YYYY-MM-DD`.
- `sp_product_ads`: Amazon exige `sku`; ASIN sozinho pode falhar.
- `sp_keywords`: use `keyword_text` + `match_type`, não expression de target.
- `sp_targets`: use `expression` com `ASIN_SAME_AS`, não keyword.

### Caso 4: `RESOURCE_NOT_FOUND (404)`

Possíveis causas:

- campaign/ad group ID não existe ou ainda não propagou.
- ad group pertence a outra campanha/profile.
- ASIN target inválido para marketplace.
- recurso criado mas readback ainda está atrasado.

Ação:

1. Readback de campaign.
2. Readback de ad group.
3. Aguardar curto intervalo se acabou de criar.
4. Validar ASIN/marketplace.
5. Tentar 1 item isolado com nova idempotency key.

## Checklist de recovery

Quando algo falhar, produza esta tabela:

| Recurso | Planejado | Criado | Faltante | Próxima ação |
|---|---:|---:|---:|---|
| campaigns | 3 | 3 | 0 | ok |
| ad groups | 3 | 3 | 0 | ok |
| product ads | 6 | 6 | 0 | ok |
| keywords | 21 | 0 | 21 | corrigir/reexecutar |
| targets | 6 | 0 | 6 | corrigir/reexecutar |
| negativos | 16 | ? | ? | readback |

## Modelo de mensagem para dev

Inclua sempre:

```text
MCP version:
resource_type:
dry_run:
idempotency_key:
connection_id:
profile_id:
payload mínimo que falhou:
retorno bruto:
readback depois do erro:
```

Exemplo:

```text
create_resources sp_keywords falha no write real.
Dry-run OK.
Payload:
{ ... }
Retorno: INTERNAL_ERROR (502)
Readback: keyword não apareceu no ad group.
```

## Modelo de resposta para usuário

Se parcial:

```text
Parcialmente criado:
- campaigns: 3/3
- ad groups: 3/3
- product ads: 6/6
Pendente por erro backend:
- keywords: 0/21
- targets: 0/6
Ação: enviei payload/erro para dev; vou continuar só dessas etapas quando corrigido.
```

Se concluído:

```text
Concluído e verificado:
- campaigns: 3
- ad groups: 3
- product ads: 6
- keywords: 21
- targets: 6
- negativos: 16
Budget diário máximo: R$120
```

## Anti-padrões

- Repetir batch inteiro depois de 502.
- Usar mesma idempotency key com payload diferente.
- Criar product ads sem SKU.
- Continuar para keywords antes de confirmar product ads.
- Dizer que criou sem readback.
- Misturar profile/marketplace.
- Fazer write sem aprovação explícita.
- Passar campos soltos no `update_resources` sem wrapper `changes`.

## Status por versão (testado 2026-07-06, MCP 1.42.2 pós-fix)

### `update_resources` — ✅ funcionando

Corrigido em 2026-07-06. O bug era duplo:
1. `id_field` no registry estava `target_id` em vez de `keyword_id` para `sp_keywords` — corrigido.
2. Schema Zod estava `.strict()` rejeitando campos extras — corrigido para `.passthrough()`.
3. Payload precisa do wrapper `changes` — isso é by design, não bug.

### `delete_resources` — ✅ funciona para `sp_keywords`

Confirmado em produção: keywords deletadas com sucesso usando:

```json
{ "keyword_id": "<keyword_id>" }
```

### Reports com `days` — ✅ funciona (assíncrono)

Os reports com `days` retornam status `PENDING` e geram o relatório em background (~45-120s). Não é mais 502.

Também funcionam com `start_date`/`end_date` no formato `YYYY-MM-DD`.

### `ask_ads_report_analyst` — ⚠️ scope sensitivity

- ✅ Funciona **sem** `connection_id`/`profile_id` (usa default da org)
- ❌ Falha **com** `connection_id` + `profile_id` (retorna `VALIDATION_ERROR (400)`)

### `list_ads_recommendations` — ✅ funcionando (graceful)

Corrigido em 2026-07-06. Se a Amazon retorna 403 (comum em contas BR), o endpoint retorna `[]` com nota explicativa em vez de 502.

### `get_bid_guidance` — ✅ funcionando

Corrigido em 2026-07-06. A Amazon exigia o campo `strategy` no payload. Agora o campo é enviado com default `CONVERSION_OPPORTUNITIES`.

Pode retornar `suggested_bid: null` para keywords sem histórico suficiente — isso é normal, não bug.

### DSP — ❌ requer conta agency

Todas as funções DSP retornam erro para contas seller. DSP requer perfil agency.

### `get_targets` para PAT — ⚠️

`get_targets` retorna `keywords[]` corretamente para campanhas KW, mas `targets[]` vem vazio para campanhas PAT mesmo quando o create retorna `DUPLICATE_VALUE` (provando que o target existe). O bug pode ser no filtro `adGroupIdFilter` do endpoint `/sp/targets/list`.

### `create_resources sp_product_ads` — formato correto

```text
✅ Funciona:  { campaign_id, ad_group_id, sku, state }
✅ Funciona:  { campaign_id, ad_group_id, asin, state }
✅ Funciona:  { campaign_id, ad_group_id, sku, asin, state }
❌ Falha:     { campaignId, adGroupId, sku, state }  (camelCase)
```

Usar sempre snake_case nos payloads de create.
