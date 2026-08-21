---
name: titanos-mcp-setup
description: Instalação e diagnóstico do MCP Titanos (@titanos/mcp-agents) — API Key, OAuth Amazon, scopes, whoami, get_server_guide. Use antes de qualquer outra skill Titanos ou quando o agente não encontra tools, retorna SCOPE_MISSING ou INVALID_API_KEY.
license: MIT
---

# Titanos MCP — Guia de instalação e setup

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Precisa instalar ou diagnosticar o MCP Titanos (API key, OAuth, scopes)
- O agente não encontra tools ou retorna SCOPE_MISSING / INVALID_API_KEY
- Vai usar qualquer outra skill Titanos pela primeira vez

Skill base para conectar **Claude Desktop**, **Claude Code**, **Cursor**, **Codex** ou **Hermes** ao MCP Titanos. Todas as outras skills desta pasta assumem que este setup está concluído.

## O que é

| Peça | Detalhe |
|------|---------|
| Pacote | `@titanos/mcp-agents` (npm, stdio MCP) |
| API | `https://www.titanos.com.br/api/mcp/*` |
| Auth | `Authorization: Bearer tnk_live_...` (API Key) |
| Conta Amazon | OAuth no painel web (tokens no vault Titanos — **nunca** no `.env` do MCP) |

Plano mínimo para API Keys: **Grow+** (`feature.mcp.api_keys`).

## Passo 1 — Conta Titanos

1. Acesse [https://www.titanos.com.br](https://www.titanos.com.br) e faça login.
2. **Conta → Integrações**
   - Conecte **Seller Central** (SP-API) se for operar listings/estoque/relatórios seller.
   - Conecte **Amazon Ads** se for operar campanhas/relatórios PPC.
   - Status deve ficar **ativo** antes de tools seller/ads retornarem dados.
3. **Conta → Integrar com IA**
   - Crie uma key com nome descritivo (ex.: `Hermes Produção`).
   - Copie o token **completo** (`tnk_live_…`, ~64 caracteres). Só aparece uma vez.

## Passo 2 — Scopes da API Key

Escolha um preset ou marque manualmente conforme as skills que o usuário vai instalar:

| Preset (UI) | Quando usar |
|-------------|-------------|
| **Recomendado** | Uso geral (seller + ads + agentes + mineração) |
| **Somente leitura** | Auditoria sem mutação |
| **Seller completo** | Listings, estoque, pedidos, relatórios SP |
| **Amazon Ads completo** | Campanhas + relatórios + write SP/SB/SD |
| **Experimentos MCP** | Testes A/B (skill futura) |

### Matriz rápida (diagnóstico)

| Scope | Tools principais |
|-------|------------------|
| `agents:listing_power:invoke` | `invoke_listing_power` |
| `agents:read` | `get_result`, polling de jobs |
| `amazon:lookup` | `lookup_amazon_product` |
| `mining:search` | `search_products`, `get_best_sellers`, `find_private_label_opportunities`, `find_categories` |
| `seller:connections:read` | `list_seller_connections`, guias |
| `seller:listings:read` | `get_listings`, `audit_listings`, `validate_listing`, … |
| `seller:listings:write` | `update_listing`, `create_listing`, `preview_listing` |
| `seller:inventory:read` | `get_inventory` |
| `seller:orders:read` / `write` | `get_orders`, `request_review` |
| `seller:reports:read` | `request_sp_report`, `ask_selling_partner_report_analyst` |
| `seller:finances:read` | Finanças SP |
| `ads:connections:read` | `list_ads_connections`, `list_ads_profiles` |
| `ads:campaigns:read` | `get_campaigns`, recomendações, PPC memory |
| `ads:reports:read` | `get_ads_performance`, `ask_ads_report_analyst` |
| `ads:campaigns:write` | `create/update/delete_resources` |

Keys antigas **não** ganham scopes novos automaticamente — revogue e recrie após upgrade de plano ou novas features.

## Passo 3 — Configurar o cliente MCP

### Claude Desktop / Claude Code

Arquivo de config MCP do cliente (caminho varia por OS):

```json
{
  "mcpServers": {
    "titanos": {
      "command": "npx",
      "args": ["-y", "@titanos/mcp-agents@1"],
      "env": {
        "TITANOS_API_KEY": "tnk_live_COLE_A_KEY_COMPLETA_AQUI",
        "TITANOS_API_URL": "https://www.titanos.com.br"
      }
    }
  }
}
```

- Use `@1` para sempre resolver a versão estável mais recente da linha 1.x.
- **Nunca** commite a key no git.
- Key truncada → erro `INVALID_API_KEY`.

### Hermes / OpenClaw

Informe o caminho da pasta de skills **ou** cole o conteúdo de `SKILL.md`. Para MCP, use o mesmo bloco JSON acima no config de servidores do Hermes.

### Cursor

Adicione o servidor MCP nas configurações do projeto com o mesmo JSON (`npx` + env).

## Passo 4 — Validar conexão (sempre)

Ordem obrigatória na primeira sessão:

```
1. whoami
2. get_server_guide({ "topic": "full" })
3. (Seller) list_seller_connections
4. (Ads) list_amazon_ads_integrations → list_amazon_ads_integration_accounts
```

### `whoami` — o que checar

- `scopes[]` contém o que a tarefa precisa
- Plano ativo (Grow+)
- Conexões Amazon listadas (seller / ads)

### Guias embutidos (0 créditos)

| Tool | Conteúdo |
|------|----------|
| `get_server_guide` | Mapa geral MCP (tools, scopes, mineração) |
| `get_seller_central_guide` | Fluxos Seller Central |
| `get_optimization_guide` | Playbook PPC |

## Erros comuns

| Sintoma | Causa | Ação |
|---------|-------|------|
| `SCOPE_MISSING` | Key sem permissão | Recriar key com scope ou preset correto |
| `INVALID_API_KEY` | Key cortada ou revogada | Nova key em Conta → Integrar com IA |
| `AMAZON_CONNECTION_NOT_FOUND` | OAuth não conectado | Conectar em Conta → Integrações |
| Tools não aparecem no cliente | Cliente sem reiniciar após config | Reiniciar Claude/Hermes |
| Seller/Ads vazio | Conexão inativa ou perfil errado | Reautorizar OAuth |

## Créditos (visão geral)

| Tipo | Exemplos | Créditos |
|------|----------|----------|
| 0 | `whoami`, guias, reads seller/ads básicos, analyst (warehouse) | 0 |
| Mineração leve | `search_products` sem detalhes | ~200 |
| Mineração full | `incluir_detalhes: true` | ~500 |
| Listing Power | `invoke_listing_power` | Variável (texto + imagens + A+) |

Resposta MCP inclui `credits_used` — reporte ao usuário após invokes pagos.

## Skills relacionadas (instalar depois do setup)

| Pasta | Card na página |
|-------|----------------|
| `amazon-mining-research` | Mineração Amazon |
| `super-listing-power` | Super Listing Power |
| `listing-create-publish` | Publicar listing |
| `Amazon-Ads` | Amazon Ads |
| … | Ver nota Maestri / índice completo |

## Checklist antes de declarar “MCP pronto”

- [ ] API Key criada e colada **inteira** no `env`
- [ ] OAuth Seller e/ou Ads ativo (conforme uso)
- [ ] `whoami` retorna scopes esperados
- [ ] `get_server_guide` responde
- [ ] Pelo menos um teste na área que o usuário vai usar (ex.: `search_products` ou `list_seller_connections`)
