---
name: amazon-reorder-planning
description: Planejamento de reabastecimento FBA via MCP Titanos — ask_selling_partner_report_analyst + get_inventory. Use para reorder point, safety stock, PO draft e coordenação com Ads (ask_ads_report_analyst).
---

# Amazon Reorder Planning — MCP Titanos

Decisões de recompra a partir de warehouse SP + inventário ao vivo. **Seller Central FBA** (BR). Vendor: ver nota no skill `amazon-seller-central`.

> Entregável: tabelas/texto (não dashboard). Para visão interativa de risco de estoque, use skill `fba-inventory-risk-dashboard`.

## Tools

| Tool | Use for |
|---|---|
| `ask_selling_partner_report_analyst` | Headline workhorse. Run rate, days of supply, sell-through, vendor sales, vendor forecast — anything backed by report data. |
| `get_inventory` | Snapshot FBA (`fulfillable`, inbound working/shipped/receiving, reserved) |
| `get_listings` | SKUs do seller |
| `list_seller_connections` / `list_selling_partner_integrations` | Passo 1: `connection_id` + marketplace (BR default). Nunca misturar marketplaces. |

## Inputs the user must supply

Reorder math is not derivable from Amazon data alone. Before producing a recommendation, ask the user for:

1. **Lead time (days)** — from PO placement to FBA check-in (or to Amazon receipt for Vendor). Often 30-90 days for international supply chains, 7-30 for domestic / 3PL.
2. **Target service level / safety stock approach** — most users want one of:
   - "I want X days of safety stock" (simple buffer)
   - "I want a 95% / 97.5% / 99% in-stock probability" (statistical, needs demand variability)
3. **Reorder horizon** — "what should I order *this month*" vs "what's my reorder schedule for the next 90 days"
4. **MOQ / case pack** (optional) — minimum order quantity from the supplier and the case-pack size. Round recommendations to the nearest MOQ.
5. **Cost per PO** (optional) — only relevant if you're doing EOQ-style optimization to balance ordering cost vs holding cost. Most users don't have this; skip unless asked.

If the user hasn't provided any of these, ask up front. Don't fabricate defaults — wrong defaults waste their cash on the wrong order size.

## The math (keep it transparent)

Show the math in your output so the user can audit it. The base reorder model:

```
lead-time demand = avg daily units * lead time (days)
safety stock     = (user's safety target, in units or days)
reorder point    = lead-time demand + safety stock
order quantity   = (target days of supply * avg daily units) - on-hand - inbound
```

Where:
- `avg daily units` comes from `ask_selling_partner_report_analyst` — typically a trailing 30-day average; use 60 or 90 days for stable lines, 7-14 for hot/seasonal SKUs. Be explicit about which window you used.
- `on-hand` = `fulfillable` de `get_inventory` (FBA).
- `inbound` = somar **working + shipped + receiving** do mesmo tool. Omitir `inbound_working` superestima risco de ruptura.

State the formulas in the response so the user can sanity-check. A reorder recommendation without visible math is a black box they can't trust with real money.

## Common workflows

### "What should I reorder this month?" (FBA)

> "For account <X>, list SKUs where (`fulfillable + inbound_working + inbound_shipped + inbound_receiving`) divided by trailing-30-day avg daily units is less than `lead_time_days + safety_stock_days`. For each, return seller SKU, ASIN, fulfillable, working/shipped/receiving inbound (sum these into a `total_inbound` column), avg daily (30d), days of cover, and recommended order quantity to bring days of cover up to `(lead time + safety stock + target horizon)`. Sort by days of cover ascending."

Substitute the user's actual `lead_time_days`, `safety_stock_days`, and `target horizon`. Lead with the SKU most at risk.

### Vendor Central forecast vs on-hand

For Vendor Central, the relevant question is "does Amazon's forecast exceed what they're holding?" Amazon places POs based on its forecast — the gap is the reorder signal:

> "Using `sp_vendor_forecasting` and `sp_vendor_inventory`, find ASINs where Amazon's mean forecast for the next 4 weeks exceeds `sellableOnHandUnits + openPurchaseOrderUnits`. Return asin, mean forecast (4w), `sellableOnHandUnits`, `openPurchaseOrderUnits`, gap, and `weeksOfCover`."

Vendor reorders are pulled by Amazon, not pushed by you — but knowing the gap tells you when to expect the next PO and whether you can fulfill it.

### Replenishment + ad coordination

This is the workflow that justifies pairing this skill with the Amazon Ads skills. Don't push spend toward SKUs that are about to stockout — and don't dial back ads on healthy SKUs:

> `ask_ads_report_analyst` (spend por ASIN, 7d) + `get_inventory` / analyst FBA para days of supply. Tabelas: **Reduzir ads** (estoque baixo + spend alto) e **Aumentar ads** (estoque saudável + spend baixo).

Output should be two short tables: "**Pull back / Pause ads**" and "**Push more ads**", each with the supporting numbers.

### PO draft

When the user wants a draft PO (typically off the back of a reorder plan):

Render a simple table:

| SKU | ASIN | Cases | Units | Notes |
|-----|------|-------|-------|-------|
| ... | ... | ... | ... | "Round-up to MOQ of 200" / "Tight — consider air freight" |

Include a one-line summary of total units and total cases. Don't generate prices unless the user supplied a price list — guessing wholesale cost from retail price is unreliable.

### Vendor replenishment timing

Vendor sellers can't push reorders, but they can warn Amazon's planners. If a forecast gap is large enough:

> "Show me ASINs where the 4-week forecast exceeds total available inventory by more than 50%. These are candidates for outreach to Amazon's vendor manager — production or shipping needs to ramp before the next PO comes in."

## Pitfalls

- **Combining marketplaces.** Reorder planning is per-marketplace (US != MX != UK). Always confirm which marketplace the user wants. Inventory pooling across marketplaces is rare and the user has to opt in to it.
- **Using a single demand window for all SKUs.** Stable lines (electronics, household goods) read well at 60-90 days. Hot or seasonal SKUs (gifting, summer goods) need 7-14 day windows so you don't under-order during a ramp.
- **Ignoring inbound.** A SKU at 5 days of fulfillable supply but 30 days of inbound on the way is fine — don't recommend an emergency reorder. Always net out inbound when computing days of cover.
- **Reorder math without lead time.** Without a lead time, there's no reorder point. Don't proceed with a default — ask.
- **Glossing over the math.** Reorder recommendations involve real cash. Show the formulas and the inputs so the user can audit and adjust.
- **Assuming Vendor and Seller workflows are the same.** Vendor inventory is Amazon's; you don't push reorders, you respond to forecast gaps. Seller inventory is yours; you do push.

## Tips

- For new SKUs without 30 days of sales history, lean on the catalog's category run-rate or ask the user for a planning estimate. Don't extrapolate from 5 days of data — you'll either over-order on a launch spike or under-order on a slow start.
- Days of supply lower than lead time means you're already late. Flag these as "express / air freight candidates" so the user knows the standard reorder won't catch the gap.
- Pair with the FBA Inventory Risk Dashboard skill when the user wants to *see* the at-risk picture before deciding what to reorder. This skill answers "what to do"; that one answers "what's the situation".
- Ad spend coordination is a high-leverage cross-skill move that most agencies miss. If the user has both ads and inventory data flowing, the replenishment + ad coordination workflow consistently surfaces wasted spend and missed budget headroom.
