---
name: super-listing-power
description: Gera listing Amazon completo via MCP Titanos — invoke_listing_power, polling get_result. Texto otimizado, imagens e A+ opcionais. Use após mineração ou com URL/manual; antes de publicar use listing-create-publish.
---

# Super Listing Power — MCP Titanos

Agente **Super Listing Power**: gera título, bullets, descrição, keywords e (opcional) **imagens de listing** + **módulos A+** via job assíncrono.

**Pré-requisitos:**

- Skill `titanos-mcp-setup`
- Scopes: `agents:listing_power:invoke` + `agents:read`
- Créditos suficientes na conta Titanos
- URLs de imagens públicas se `gerar_imagens` ou `gerar_aplus` = true

## Tools

| Tool | Scope | Uso |
|------|-------|-----|
| `invoke_listing_power` | `agents:listing_power:invoke` | Inicia job |
| `get_result` | `agents:read` | Polling de status/resultado |
| `wait_for_result` | `agents:read` | (Cliente MCP) espera até concluir |
| `lookup_amazon_product` | `amazon:lookup` | Contexto prévio (opcional) |

## Quando usar vs outras skills

| Situação | Skill |
|----------|-------|
| Pesquisar nicho / ASINs concorrentes | `amazon-mining-research` |
| **Gerar copy + criativos IA** | **Esta skill** |
| Auditar listing existente | `amazon-listing-audit` |
| Publicar/patch na conta Seller | `listing-create-publish` |

## Fontes de entrada (`source`)

### `manual`

Produto descrito pelo usuário:

```json
{
  "source": "manual",
  "produto": {
    "nome": "Nome do produto",
    "descricao": "Descrição detalhada, materiais, público, diferenciais..."
  },
  "pais": "BR",
  "gerar_imagens": false,
  "gerar_aplus": false
}
```

### `amazon_url`

Referência de concorrente ou produto similar:

```json
{
  "source": "amazon_url",
  "amazon_url": "https://www.amazon.com.br/dp/B0XXXXXXXXX",
  "pais": "BR",
  "gerar_imagens": true,
  "gerar_aplus": true,
  "imagens": [
    "https://cdn.exemplo.com/ref1.jpg",
    "https://cdn.exemplo.com/ref2.jpg"
  ]
}
```

URL deve conter `/dp/` ou `/gp/` + ASIN válido (10 caracteres).

## Opções importantes

| Campo | Default | Notas |
|-------|---------|-------|
| `gerar_imagens` | false | Imagens de listing (KIE) — exige `imagens[]` (máx 4 URLs) |
| `gerar_aplus` | false | Módulos A+ — exige `imagens[]` |
| `gerar_mineracao` | false | Enriquece com dados de mercado no pipeline |
| `gerar_mercado_livre` | false | Variante ML (não Amazon) |
| `modelo_texto` | default | `gemini_pro`, `claude_sonnet`, `gpt_5_5`, … |
| `modelo_imagem` | default | `fal_gpt_image_2`, `fal_nano_banana_2`, … |
| `aplus_quality` | — | `low` / `medium` / `high` |
| `aplus_resolution` | — | `2K` / `4K` |
| `aplus_num_imagens` | — | 1–4 |
| `idempotency_key` | — | Evita cobrança duplicada em retry (1–100 chars) |

**Regra:** se `gerar_imagens` ou `gerar_aplus` = true → `imagens` obrigatório (array de URLs HTTPS, máx 4).

## Workflow completo

### 1. Preparar contexto (recomendado)

- `whoami` — créditos e plano
- Opcional: `search_products` + `lookup_amazon_product` (skill mineração)
- Opcional: `audit_listings` se for **relist** de ASIN próprio

### 2. Invocar

```
invoke_listing_power(body)
```

Resposta típica:

- `request_id` — UUID para polling
- `status` — `accepted` / `in_progress`
- `credits_used` — débito inicial

Envie header `Idempotency-Key` se o cliente MCP suportar (mesmo valor que `idempotency_key` no body).

### 3. Polling

```
get_result({ request_id })
```

Estados até `completed` ou `failed`:

- `in_progress` — aguarde (jobs podem levar vários minutos com imagens/A+)
- `completed` — `data` com texto, URLs de imagens geradas, estrutura A+
- `failed` — erro; créditos podem ter refund conforme política Titanos

**Limites:** concurrency MCP — máx 2 jobs `processando` por API key; não dispare invokes paralelos excessivos.

### 4. Entregar ao usuário

- Apresente título, bullets, descrição, backend keywords em markdown
- Links de imagens/A+ quando gerados
- Sugira `preview_listing` → `update_listing` (skill publicar) se conta Seller conectada

## Créditos (ordem de grandeza)

Custo **dinâmico** (texto + flags de imagem/A+). A resposta do invoke e do histórico Titanos traz `credits_used`.

Referência web (UI interna, pode variar):

- Base texto ~600
- Imagens ~1000
- A+ ~800

Sempre confirme saldo com o usuário antes de `gerar_imagens` + `gerar_aplus` juntos.

## Fluxos por persona

### Private label (manual)

1. Usuário descreve produto + envia fotos (URLs)
2. `invoke_listing_power` manual + `gerar_imagens: true`
3. Revisão humana do copy
4. `listing-create-publish` para subir na conta

### Melhorar concorrente (amazon_url)

1. `lookup_amazon_product` no ASIN referência
2. `invoke_listing_power` com `amazon_url` + diferenciais no chat
3. Comparar bullets gerados vs referência

### Relist otimizado

1. `get_listing` + `audit_listings` no SKU existente
2. `invoke_listing_power` manual incorporando correções da auditoria
3. `preview_listing` / `update_listing` (não publicar sem confirmação)

## Erros comuns

| Erro | Ação |
|------|------|
| `SCOPE_MISSING` | Key sem `agents:listing_power:invoke` |
| Validação `imagens` | Adicionar URLs se geração visual ativa |
| Job lento | Normal com A+; continuar polling |
| `failed` | Ler `error` no result; não reinvocar sem idempotency se já debitou |

## Pitfalls

- **Publicar direto sem preview** — Amazon rejeita atributos inválidos; use skill `listing-create-publish`.
- **Imagens não-HTTPS ou inacessíveis** — falha no pipeline de imagem.
- **Confundir com Listing Creative** — `invoke_listing_creative` é **admin-only** Titanos; não documentar para usuário final.

## Dicas

- `gerar_mineracao: true` quando o usuário já fez mineração no mesmo nicho — reforça dados de mercado no prompt.
- Para só texto rápido: `gerar_imagens: false`, `gerar_aplus: false` — menor custo e latência.
- Após sucesso, ofereça skill `amazon-listing-audit` no ASIN publicado (pós-go-live).