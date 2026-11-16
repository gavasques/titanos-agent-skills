---
name: titanos-mcp-setup
description: Instalação e diagnóstico do MCP Titanos (OAuth 2.1) — conexão HTTP ou @titanos/mcp-agents, scopes, whoami, get_server_guide. Use antes de qualquer outra skill Titanos ou quando o agente não encontra tools, retorna SCOPE_MISSING ou erro de autorização (401).
license: MIT
---

# Titanos MCP — Guia de instalação e setup

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Precisa instalar ou diagnosticar o MCP Titanos (login OAuth, scopes)
- O agente não encontra tools ou retorna SCOPE_MISSING / 401
- Vai usar qualquer outra skill Titanos pela primeira vez

Skill base para conectar **Claude Desktop**, **Claude Code**, **Cursor**, **Codex** ou **Hermes** ao MCP Titanos. Todas as outras skills desta pasta assumem que este setup está concluído.

## O que é

| Peça | Detalhe |
|------|---------|
| Servidor | `https://www.titanos.com.br/api/mcp` (Streamable HTTP, sem sessão) |
| Auth | **OAuth 2.1** com PKCE: login no navegador, **nenhuma chave para copiar** |
| Alternativa stdio | `@titanos/mcp-agents@1` (versão **1.44.0 ou superior**, que fala OAuth) |
| Conta Amazon | OAuth no painel web (tokens no vault Titanos — **nunca** no `.env` do MCP) |

Plano mínimo: **Grow+**. Guia público completo: <https://www.titanos.com.br/docs/mcp> (Markdown: <https://www.titanos.com.br/docs/mcp.md>). Central de Ajuda pública: <https://www.titanos.com.br/ajuda>.

> Chaves `tnk_*` (API keys antigas) estão **desativadas por padrão**. Não peça nem aceite chave colada em config: use o login OAuth abaixo.

## Passo 1 — Conta Titanos

1. Acesse [https://www.titanos.com.br](https://www.titanos.com.br) e faça login (plano Grow ou superior).
2. **Conta → Integrações**
   - Conecte **Seller Central** (SP-API) se for operar listings/estoque/relatórios seller.
   - Conecte **Amazon Ads** se for operar campanhas/relatórios PPC.
   - Status deve ficar **ativo** antes de tools seller/ads retornarem dados.
3. **Conta → Integrar com IA** mostra as conexões autorizadas e permite revogar.

## Passo 2 — Scopes (tela de consentimento)

Na primeira conexão, o navegador abre: escolha a organização, **revise as permissões (scopes)** e aprove. Aprove só o necessário para as skills que o usuário vai usar. Os modos ajudam a limitar o cliente:

| Modo (`TITANOS_MCP_PROFILE`) | Quando usar |
|------------------------------|-------------|
| `titanos-core` | Uso geral |
| `amazon-seller` | Listings, estoque, pedidos, relatórios SP |
| `amazon-ads` | Campanhas + relatórios PPC |
| `amazon-content` | Conteúdo e listing |
| `titanos-miner` | Pesquisa e mineração |
| `mercado-livre`, `olist`, `bling`, `shopee` | Canais e ERP |

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

Scopes novos (por upgrade de plano ou novas features) não entram sozinhos: reconecte o cliente e aprove de novo na tela de consentimento.

## Passo 3 — Conectar o cliente MCP

URL do servidor: `https://www.titanos.com.br/api/mcp`. Sem credencial na configuração.

```bash
# Claude Code
claude mcp add --transport http titanos https://www.titanos.com.br/api/mcp

# Codex
codex mcp add titanos --url https://www.titanos.com.br/api/mcp
codex mcp login titanos

# Grok
grok mcp add --transport http titanos https://www.titanos.com.br/api/mcp
```

**Cursor** (`~/.cursor/mcp.json`):

```json
{ "mcpServers": { "titanos": { "url": "https://www.titanos.com.br/api/mcp" } } }
```

**Claude (web/desktop):** abra Conectores nas configurações do Claude e adicione um conector personalizado com a URL acima.

**Clientes sem suporte a HTTP/OAuth (stdio):** exige Node.js 18+ e `@titanos/mcp-agents` 1.44.0+. O login OAuth abre no navegador e o pacote guarda o token curto em `~/.titanos`:

```json
{
  "mcpServers": {
    "titanos": {
      "command": "npx",
      "args": ["-y", "@titanos/mcp-agents@1"],
      "env": { "TITANOS_API_URL": "https://www.titanos.com.br" }
    }
  }
}
```

- Use `@1` para sempre resolver a versão estável mais recente da linha 1.x.
- Para limitar o modo, acrescente `"TITANOS_MCP_PROFILE": "amazon-ads"` (ou outro) em `env`.
- **Nunca** coloque chave, token ou senha na configuração nem no git.
- Reinicie o cliente depois de alterar a configuração.

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
- Plano ativo (Grow+) e a organização autorizada
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
| 401 / erro de autorização | Login expirado ou não concluído | Refazer o login (`codex mcp login titanos`) e conferir o plano Grow+ |
| `SCOPE_MISSING` | Scope não aprovado no consentimento | Reconectar e aprovar o scope necessário |
| `AMAZON_CONNECTION_NOT_FOUND` | OAuth Amazon não conectado | Conectar em Conta → Integrações |
| Navegador não abre | Navegador padrão ausente | Copiar a URL de autorização do terminal |
| Tools não aparecem | Cliente não reiniciado ou scope/plano sem a função | Reiniciar; revisar scopes e plano |
| `npx` não encontrado (stdio) | Node.js ausente | Instalar Node.js 18+ |
| Seller/Ads vazio | Conexão inativa ou perfil errado | Reautorizar OAuth Amazon |

Mais respostas: <https://www.titanos.com.br/ajuda>.

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
| `amazon-ads` | Amazon Ads |
| … | Ver nota Maestri / índice completo |

## Checklist antes de declarar “MCP pronto”

- [ ] Cliente conectado e login OAuth concluído (sem chave em config)
- [ ] OAuth Seller e/ou Ads ativo (conforme uso)
- [ ] `whoami` retorna scopes esperados
- [ ] `get_server_guide` responde
- [ ] Pelo menos um teste na área que o usuário vai usar (ex.: `search_products` ou `list_seller_connections`)
