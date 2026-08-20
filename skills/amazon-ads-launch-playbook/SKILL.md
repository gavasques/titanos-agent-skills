---
name: amazon-ads-launch-playbook
description: Use when an AI agent needs a ready-to-run Amazon Ads launch workflow for a new product using Titanos MCP, including account discovery, listing checks, campaign architecture, budget split, launch QA, and first-7-days monitoring.
---

# Amazon Ads Launch Playbook — Titanos MCP

## Objetivo

Dar ao agente um roteiro executivo para lançar Ads de um produto novo na Amazon usando Titanos MCP, sem ficar perguntando ou descobrindo o básico.

Esta skill é mais estratégica que `amazon-sp-campaign-launcher`: ela cobre pré-lançamento, criação, QA e acompanhamento dos primeiros dias.

## Resposta curta para o agente

Para um lançamento padrão de Sponsored Products:

1. Descubra conta/profile.
2. Valide ASIN/SKU/listing/estoque/preço.
3. Monte 3 campanhas: AUTO, KW Core, PAT Concorrentes.
4. Use bids calculados por preço × CVR × target ACOS.
5. Faça dry-run.
6. Peça confirmação do payload.
7. Crie em ordem e faça readback.
8. Monitore 72h e 7 dias.

## Fase 1 — Discovery

Execute:

```text
whoami
list_ads_connections
list_ads_profiles
list_resource_types
get_ads_report_metadata
```

Escolha um único `connection_id` + `profile_id`.

Se houver múltiplos marketplaces, não misture.

## Fase 2 — Validação do produto

Para cada ASIN/SKU:

| Check | Por quê |
|---|---|
| ASIN existe | evita campanha sem produto válido |
| SKU pertence ao seller | product ads exigem SKU correto |
| Buyable | tráfego sem compra desperdiça dinheiro |
| Estoque disponível | evita gastar antes de ruptura |
| Preço atual | base do bid máximo |
| Listing sem issue crítica | CTR/CVR dependem disso |
| Buy Box/elegibilidade | anúncio pode não servir ou converter mal |

Se listing/estoque estiver ruim, recomende corrigir antes de ativar Ads.

## Fase 3 — Arquitetura padrão

### Campanha 1 — AUTO Discovery

Objetivo: capturar termos e ASINs que a Amazon encontra.

```text
SP | <Marca> | <Produto> | AUTO | Discovery | <YYYYMM>
```

Config:

- budget: 20–30% do total.
- bid: conservador.
- bidding: down only.
- product ads: todos os SKUs do produto.
- negativos iniciais: termos claramente irrelevantes.

### Campanha 2 — KW Core

Objetivo: controlar termos principais.

```text
SP | <Marca> | <Produto> | KW | Core | <YYYYMM>
```

Config:

- budget: 40–60% do total.
- exact para termos core.
- phrase para variações seguras.
- broad só se houver verba de descoberta.

### Campanha 3 — PAT Concorrentes

Objetivo: aparecer em páginas de concorrentes/substitutos.

```text
SP | <Marca> | <Produto> | PAT | Concorrentes | <YYYYMM>
```

Config:

- budget: 20–30% do total.
- ASINs concorrentes diretos.
- bids abaixo dos exact, salvo concorrente validado.

## Split de budget

Padrão conservador:

```text
AUTO: 25%
KW Core: 50%
PAT: 25%
```

Exemplo para R$120/dia:

```text
AUTO: R$30/dia
KW: R$60/dia
PAT: R$30/dia
```

Se estoque for baixo, reduza budgets ou crie PAUSED.

## Bids iniciais

Use:

```text
max_cpc = preço × CVR esperado × target_ACOS
```

Sem histórico:

| Campanha | Bid relativo |
|---|---:|
| KW exact core | 100% |
| KW phrase | 60–80% |
| AUTO | 50–70% |
| PAT | 50–70% |

Nunca prometa bid perfeito no lançamento. O objetivo inicial é comprar dados sem queimar caixa.

## Fase 4 — Dry-run obrigatório

Dry-run para:

```text
sp_campaigns
sp_ad_groups
sp_product_ads
sp_keywords
sp_targets
sp_campaign_negative_keywords
```

Verifique se o planned payload tem:

- `start_date` em ISO `YYYY-MM-DD`.
- budget diário correto.
- product ads com `sku` e `asin`.
- keywords com `keyword_text` e `match_type`.
- targets com `expression`.

## Fase 5 — Aprovação do usuário

Mostre antes do write:

```text
Vou criar:
- 3 campanhas SP
- 3 ad groups
- X product ads
- Y keywords
- Z product targets
- N negativos
Budget máximo diário: R$___
Estado: ENABLED/PAUSED
```

Só crie depois de aprovação explícita.

## Fase 6 — Write em ordem

Ordem:

1. campaigns
2. readback campaigns
3. ad groups
4. readback ad groups
5. product ads
6. readback product ads
7. keywords
8. targets
9. negativos
10. readback final

Use idempotency key por lote.

## Fase 7 — QA pós-criação

Confirme:

| Item | Critério |
|---|---|
| Campanhas | estado/budget corretos |
| Ad groups | um por campanha |
| Product ads | todos SKUs em todos ad groups certos |
| Keywords | match/bid corretos |
| Targets | ASIN/expression/bid corretos |
| Negativos | termos de exclusão aplicados |

Se algo falhar, não esconda. Diga o que criou e o que falta.

## Fase 8 — Monitoramento inicial

### Após 24–72h

Checar:

- impressões,
- cliques,
- gasto,
- CPC,
- CTR,
- budget cap,
- se product ads estão servindo.

Não matar termo cedo demais se ainda não houve cliques suficientes.

### Após 7 dias ou 20+ cliques

Ações:

- termos com cliques e zero venda: watch/negativo dependendo de volume.
- termos com venda: migrar para exact se vieram de AUTO/PHRASE/BROAD.
- PAT com gasto sem venda: reduzir bid ou pausar.
- AUTO com bons search terms: harvest.

## Mensagem final ideal

```text
Lançamento criado e verificado.
Budget diário máximo: R$___
Campanhas: ___
Product ads: ___
Keywords: ___
Targets: ___
Próximo check recomendado: D+3 para servir/impressões e D+7 para search terms.
```

## Erros comuns

- Criar KW/PAT sem product ads.
- Criar campanha enabled com listing sem estoque.
- Deixar KW/PAT sem targets e achar que está rodando.
- Misturar ASIN target com keyword target.
- Reexecutar write inteiro após erro parcial.
- Não avisar que dados Ads têm lag.
