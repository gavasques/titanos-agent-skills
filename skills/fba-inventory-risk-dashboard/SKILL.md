---
description: Dashboard de risco de estoque FBA via MCP Titanos — ask_selling_partner_report_analyst (warehouse FBA) + get_inventory. Artifact React estilo editorial. Triggers: days of supply, stockout, reorder urgency, inventário FBA saudável.
---

# FBA Inventory Risk Dashboard — MCP Titanos

Visão interativa de **risco de ruptura** FBA (Seller Central BR). Para **decisão de compra/PO**, use skill `amazon-reorder-planning`.

## Tools

| Tool | Uso |
|------|-----|
| `list_seller_connections` | `connection_id` SP-API |
| `get_inventory` | Quantidades ao vivo (fulfillable + inbound) |
| `ask_selling_partner_report_analyst` | Days of supply, sell-through, tendência de vendas (warehouse FBA + sales) |
| `get_seller_central_guide` | Fluxos e report types |

## Workflow

### 1. Conta e marketplace

`list_seller_connections` → um `connection_id` por dashboard (BR default).

### 2. Query de risco (analyst)

Exemplo:

> "Para cada SKU com afn_fulfillable_quantity > 0: calcular média diária de unidades vendidas (últimos 30 dias) e dias de cobertura = fulfillable / média diária. Incluir SKU, ASIN, fulfillable, inbound total (working+shipped+receiving), dias de cobertura. Ordenar por dias de cobertura ascendente. Top 40 SKUs em maior risco."

Complementar com `get_inventory` para validar inbound em tempo real.

### 3. Triage (antes do artifact)

- SKUs com cobertura &lt; 14 dias → **crítico**
- SKUs com vendas acelerando + estoque baixo → **urgente**
- Overstock (&gt; 90 dias cobertura) → informativo

### 4. Artifact

Entregar tabela ou artifact React (se template disponível) com tiers de risco. Paleta alinhada aos dashboards Ads (cream + serif + mono) se o cliente suportar artifacts.

### 5. Próximos passos

- Risco alto → skill `amazon-reorder-planning` (pedir lead time do usuário)
- SKU com ads alto + estoque baixo → `ask_ads_report_analyst` + reduzir spend

## Pitfalls

- Não misturar FBM-only sem estoque FBA
- Inbound incompleto superestima ruptura
- Analyst exige reports ingeridos — se vazio, `request_sp_report` (FBA inventory / sales traffic) e aguardar ingestão

## Gap vs AdPros

Skill original AdPros assumia warehouse + template JSX bundled. Titanos: datasets FBA/sales no analyst estão ativos; **template JSX** pode ser copiado do pacote AdPros ou criado no produto Titanos — popule `RAW` com colunas do analyst.