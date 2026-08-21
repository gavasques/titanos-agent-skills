---
name: bling-erp
description: Bling ERP v3 via Titanos MCP para Claude, Codex e outros agentes. Use quando precisar conectar, consultar ou operar Bling API v3 com multi-contas, OAuth, billing por chamada, quotas, rate limits (3 req/s, 120k/dia), idempotência e webhooks.
license: MIT
---

# Bling ERP — MCP Titanos

## Quando usar / When to Use This Skill

Use esta skill quando o usuário / Use this skill when the user:
- Pede para conectar, consultar ou operar Bling ERP pelo Titanos
- Precisa da API Bling v3 (OAuth, quotas, webhooks, rate limits)
- Menciona Bling via MCP (não tokens OAuth no .env local)

Use esta skill quando o usuário pedir para acessar Bling ERP pelo Titanos MCP. O Titanos é o broker: o agente usa `TITANOS_API_KEY`; tokens OAuth do Bling ficam criptografados no Titanos, nunca no `.env` local.

## Comece sempre

1. Rode `whoami` e confirme plano, organização e scopes.
2. Rode `get_server_guide({ "topic": "full" })` se as tools Bling não aparecerem.
3. Rode `get_bling_guide` quando disponível.
4. Rode `list_bling_connections` e escolha uma `connection_id` ativa.
5. Se faltar tool ou scope, pare e explique: integração Bling ainda não publicada, key sem scope, ou conta Bling não conectada.

## Setup do usuário

O usuário conecta a conta no painel Titanos, não no cliente MCP:

1. Criar aplicativo público na Central de Extensões Bling (categoria "Soluções em IA").
2. Configurar redirect URI do Titanos exatamente como `https://titanos.com.br/BlingCallback`.
3. Autorizar OAuth no Titanos (`/conta/integracoes?tab=bling`).
4. Criar API Key MCP com scopes Bling (preset `bling-erp` ou leitura).
5. Configurar Claude/Codex com `@titanos/mcp-agents@1` e `TITANOS_API_KEY`.

OAuth Bling: authorization code expira em **1 minuto** (exchange imediato no callback). Access token expira em **6 horas**; refresh token em **30 dias**. Se a conexão estiver expirada, peça reautorização no Titanos.

## Scopes (v1)

| Scope | Quando precisa |
|-------|----------------|
| `bling:connections:read` | Listar conexões e dados da empresa |
| `bling:catalog:read` | Consultar produtos, categorias, grupos, canais e depósitos |
| `bling:stock:read` | Consultar saldos de estoque |
| `bling:orders:read` | Consultar pedidos de vendas e compras |
| `bling:crm:read` | Consultar contatos |
| `bling:logistics:read` | Consultar logísticas, remessas e etiquetas |
| `bling:catalog:write` | Criar/editar produtos, categorias, depósitos |
| `bling:stock:write` | Criar/editar registros de estoque |
| `bling:orders:write` | Criar/editar pedidos e lançamentos de estoque |
| `bling:crm:write` | Criar/editar contatos |

NF-e, financeiro e escrita fiscal ficam **fora da v1**. Não peça write scopes para auditoria.

## Tools esperadas

| Tool | Uso |
|------|-----|
| `get_bling_guide` | Guia embutido, 0 créditos |
| `list_bling_connections` | Contas conectadas, 0 créditos |
| `bling_get_catalog` | Lista endpoints REST v3 catalogados, scopes e path params |
| `bling_call_endpoint` | Executa endpoint catalogado por `tool` |

Mapeamento HTTP Titanos: `POST /api/mcp/bling/guide`, `GET /api/mcp/bling/connections`, `POST /api/mcp/bling/catalog` e `POST /api/mcp/bling/call`.

Use somente `tool` catalogada. Nunca tente URL livre.

## Custos e quotas

Chamadas de setup e guias custam 0. Cada chamada REST v3 enviada ao Bling custa 5 créditos ou consome quota inclusa do plano.

| Plano | Chamadas inclusas/mês |
|-------|----------------------|
| Extensão | Sem acesso |
| Launch | 100 |
| Grow | 800 |
| Scale | 2000 |
| Dominate | 5000 |
| Titan | 10000 |
| Empire / Enterprise | Ilimitado |

Informe custo estimado antes de loops grandes. Retries idempotentes não devem cobrar novamente.

## Rate limits Bling

- **3 req/s** e **120.000 req/dia** por conta (global).
- Em `429`, respeite `retry_after_seconds` retornado pelo Titanos.
- Filtros de data com intervalo **> 1 ano** são rejeitados (HTTP 400 na API) — use intervalos ≤ 366 dias.

## Paginação

Produtos usam `pagina` e `limite` (default 100). Leia uma página, processe, avance `pagina` até a página vir com menos de 100 itens.

Exemplo `bling_call_endpoint`:

```json
{
  "tool": "bling_produtos_list",
  "connection_id": "<uuid>",
  "query": { "pagina": 1, "limite": 100 }
}
```

Pedidos de vendas: `bling_pedidos_vendas_list` com `dataInicial`/`dataFinal` (máx. 1 ano).

## Idempotência

Para POST/PUT/PATCH/DELETE:

1. Leia o estado atual quando possível.
2. Gere `idempotency_key` única por operação de negócio.
3. Peça confirmação humana para operações irreversíveis ou em massa.
4. Em timeout, repita com a mesma `idempotency_key`.

Formato sugerido:

```text
bling:{connection_id}:{tool}:{natural_id}:{YYYYMMDDHHmm}
```

## Retries

Retry apenas para 408, 429 e 5xx com backoff. Em 429, aguarde o tempo indicado. Não retry 400, 401, 403, 404. Em 401, a conexão pode ter expirado; peça reautorização.

## Webhooks

Webhooks Bling usam HMAC-SHA256 (`X-Bling-Signature-256`) com o `client_secret` do app. O Titanos persiste eventos e processa de forma assíncrona. Eventos `product.*` e `stock.*` atualizam produtos já sincronizados em Meus Produtos.

Quando analisar webhook:

- Trate como sinal; reconsulte a API antes de decisões críticas.
- Não registre payload bruto com tokens em respostas ao usuário.

## Prompts úteis

```text
Liste minhas conexões Bling no Titanos MCP e diga qual está ativa. Não faça writes.
```

```text
Use a conexão Bling ativa para listar produtos (bling_produtos_list), página por página, respeitando rate limit.
```

```text
Liste pedidos de vendas dos últimos 7 dias (bling_pedidos_vendas_list) e resuma por situação.
```

## Erros comuns

| Sintoma | Causa provável | Ação |
|---------|----------------|------|
| `SCOPE_MISSING` | API Key sem scope Bling | Recriar key com preset `bling-erp` |
| `BLING_CONNECTION_NOT_FOUND` | Conta não conectada | Conectar em `/conta/integracoes?tab=bling` |
| `BLING_CONNECTION_EXPIRED` | Refresh token venceu | Reautorizar OAuth |
| `BLING_RATE_LIMITED` | 3 req/s ou 120k/dia | Aguardar e reduzir paralelismo |
| Tool Bling não aparece | Package MCP desatualizado | Atualizar `@titanos/mcp-agents` |

## Fontes oficiais

- https://developer.bling.com.br/
- https://developer.bling.com.br/build/assets/openapi-Bv1-CYM5.json
