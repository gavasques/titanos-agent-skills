---
description: Criar e publicar listings na conta Seller via MCP Titanos — product type schema, validate_listing, preview_listing, create_listing, update_listing, changelog. Use após Super Listing Power ou auditoria; sempre dry-run/preview antes de write.
---

# Criar e publicar listing — MCP Titanos

Fluxo **write** Seller Central (SP-API) pela conta conectada do usuário. Complementa `amazon-listing-audit` (diagnóstico) e `super-listing-power` (geração IA).

**Pré-requisitos:**

- Skill `titanos-mcp-setup`
- OAuth Seller Central **ativo**
- Scopes: `seller:listings:read` + `seller:listings:write`
- `connection_id` de `list_seller_connections`

## Tools

| Tool | Scope | Uso |
|------|-------|-----|
| `list_seller_connections` | `seller:connections:read` | Obter `connection_id` |
| `get_listings` / `get_listing` | read | SKU/ASIN existente |
| `search_selling_partner_catalog` | read | Buscar ASIN no catálogo |
| `get_catalog_item` | read | Atributos de ASIN |
| `search_product_types` | read | Achar `productType` |
| `get_product_type_schema` | read | Schema obrigatório de atributos |
| `get_browse_node_recommendations` | read | Categorias sugeridas |
| `validate_listing` | read | Valida atributos sem publicar |
| `preview_listing` | write* | Simula patch/create |
| `update_listing` | write | Patch em SKU existente |
| `create_listing` | write | Novo SKU/listing |
| `get_listing_restrictions` | read | Bloqueios de venda |
| `list_agent_changelog` / `get_changeset` | read | Histórico de mutações MCP |
| `get_seller_central_guide` | read | Fluxo topic `listings_quality` |

\* `preview_listing` usa scope write mas não publica.

Guia embutido: `get_seller_central_guide({ topic: "listings_quality" })` ou resource `titanos://seller/flows` Fluxo B.

## Parâmetros comuns

Quase todas as mutações exigem:

```json
{
  "connection_id": "uuid-da-conexao-sp",
  "marketplace_id": "A2Q3Y263D00KWC",
  "sku": "SEU-SKU-001"
}
```

Brasil: `marketplace_id` = `A2Q3Y263D00KWC` (confirmar em `list_seller_connections` se multi-marketplace).

## Fluxo A — Novo listing (create)

```text
search_product_types → get_product_type_schema
    → montar attributes (título, bullets, preço, etc.)
    → validate_listing
    → preview_listing
    → create_listing (após confirmação explícita do usuário)
    → list_agent_changelog
```

### 1. Product type

```
search_product_types({ connection_id, keywords: "garrafa termica" })
get_product_type_schema({ connection_id, product_type: "WATER_BOTTLE", marketplace_id })
```

O schema define atributos obrigatórios e enums — **não invente campos** fora do schema.

### 2. Validar

```
validate_listing({
  connection_id,
  marketplace_id,
  sku,
  product_type,
  attributes: { ... }
})
```

Corrija `issues[]` antes de seguir.

### 3. Preview (obrigatório antes de write)

```
preview_listing({
  connection_id,
  marketplace_id,
  sku,
  product_type,
  attributes: { ... }
})
```

Mostre ao usuário diff/resumo do preview.

### 4. Create

Somente após **confirmação explícita**:

```
create_listing({
  connection_id,
  marketplace_id,
  sku,
  product_type,
  attributes: { ... }
})
```

Listings podem levar **horas** para refletir na Amazon.

## Fluxo B — Atualizar listing existente

```text
get_listings → get_listing
    → validate_listing (atributos novos)
    → preview_listing
    → update_listing (dry_run opcional — ver abaixo)
```

### `update_listing` seguro

- Use `preview_listing` sempre antes
- Campo `dry_run: true` quando disponível no payload para simular sem aplicar
- Patch apenas atributos necessários (título, bullets, preço, estoque conforme schema)

### Restrições

```
get_listing_restrictions({ connection_id, asin, marketplace_id })
```

Se restrito, explique ao usuário antes de tentar write.

## Fluxo C — Da geração IA para a conta

1. Usuário concluiu `invoke_listing_power` (skill `super-listing-power`)
2. Mapear output IA → atributos do `get_product_type_schema`
3. `validate_listing` → corrigir issues de compliance (skill `amazon-listing-audit` ajuda)
4. `preview_listing` → usuário aprova
5. `create_listing` ou `update_listing`

**Não** publique bullets com claims proibidos — cruzar com `audit_listings` se ASIN já existia.

## Changelog e auditoria

Após mutação MCP:

```
list_agent_changelog({ connection_id, filters: { sku } })
get_changeset({ connection_id, mutation_id })
```

Útil para suporte e rollback manual (Amazon não tem undo automático).

## Integração com outras skills

| Antes do write | Skill |
|----------------|-------|
| Copy/imagens IA | `super-listing-power` |
| Compliance / RUFUS | `amazon-listing-audit` |
| Pesquisa de nicho | `amazon-mining-research` |
| Ads no ASIN após live | `Amazon-Ads` |

## Perguntas típicas

- “Publica este título e bullets no SKU X” → Fluxo B com preview
- “Cria listing novo para este produto” → Fluxo A
- “Por que Amazon rejeitou?” → `validate_listing` + `get_listing` issues
- “O que mudou ontem?” → `get_listing_change_history` (read; skill auditoria)

## Pitfalls

- **Write sem confirmação** — regra de segurança; listings afetam receita e compliance.
- **Atributos fora do schema** — rejeição silenciosa ou erro SP-API.
- **SKU duplicado** — `create_listing` falha; use `update_listing`.
- **Confundir catálogo vs listing** — `get_catalog_item` é catálogo Amazon; listing do seller é `get_listing`.
- **Sem OAuth** — `AMAZON_CONNECTION_NOT_FOUND`; voltar ao setup.

## Erros comuns

| Código / situação | Ação |
|-------------------|------|
| `VALIDATION_ERROR` | Releia issues do `validate_listing` |
| `AMAZON_API_ERROR` | Rate limit ou payload; reduzir batch |
| Product type mismatch | `search_product_types` de novo |
| Restrictions ativas | `get_listing_restrictions` |

## Dicas

- `get_browse_node_recommendations` ajuda categoria quando usuário não sabe browse node.
- Para só leitura de qualidade, não use esta skill — use `amazon-listing-audit`.
- Após publicar, sugira `request_sp_report` (sales/traffic) após ingestão para medir conversão.
- Mutações MCP seller read de changelog: 0 créditos; writes seguem política de créditos seller se aplicável.