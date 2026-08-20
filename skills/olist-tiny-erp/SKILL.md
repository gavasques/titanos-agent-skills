---
name: olist-tiny-erp
description: Olist Tiny ERP via Titanos MCP para Claude, Codex e outros agentes. Use quando precisar conectar, consultar ou operar Olist/Tiny ERP REST v3 com multi-contas, OAuth, billing por chamada, quotas, rate limits, idempotência e webhooks.
---

# Olist Tiny ERP — MCP Titanos

Use esta skill quando o usuário pedir para acessar Olist Tiny ERP pelo Titanos MCP. O Titanos deve ser o broker: o agente usa `TITANOS_API_KEY`; tokens OAuth da Olist ficam criptografados no Titanos, nunca no `.env` local.

## Comece sempre

1. Rode `whoami` e confirme plano, organização e scopes.
2. Rode `get_server_guide({ "topic": "full" })` se as tools Olist não aparecerem.
3. Rode `get_olist_guide` quando disponível.
4. Rode `list_olist_connections` e escolha uma `connection_id` ativa.
5. Se faltar tool ou scope, pare e explique: integração Olist ainda não publicada, key sem scope, ou conta Olist não conectada.

## Setup do usuário

O usuário conecta a conta no painel Titanos, não no cliente MCP:

1. Criar aplicativo no ERP da Olist/Tiny.
2. Configurar redirect URI do Titanos exatamente como `https://titanos.com.br/TinyCallback`.
3. Autorizar OAuth no Titanos.
4. Criar API Key MCP com scopes Olist.
5. Configurar Claude/Codex com `@titanos/mcp-agents@1` e `TITANOS_API_KEY`.

OAuth oficial: access token expira em 4 horas e refresh token em 1 dia. Se a conexão estiver expirada, peça reautorização no Titanos.

## Scopes

| Scope | Quando precisa |
|-------|----------------|
| `olist:connections:read` | Listar conexões e status |
| `olist:catalog:read` | Consultar produtos, categorias, marcas, listas de preço e estoque |
| `olist:orders:read` | Consultar pedidos, separação, expedição e logística |
| `olist:finance:read` | Consultar contas a pagar/receber |
| `olist:documents:read` | Consultar notas fiscais e documentos fiscais |
| `olist:crm:read` | Consultar contatos, vendedores, serviços e CRM |
| `olist:catalog:write` | Criar/editar catálogo, preços e estoque |
| `olist:orders:write` | Criar/editar pedidos e fluxos operacionais |
| `olist:finance:write` | Baixar contas, criar marcadores e lançar financeiro |
| `olist:documents:write` | Autorizar/cancelar/incluir notas e lançar contas/estoque |
| `olist:crm:write` | Criar/editar contatos, assuntos, estágios e serviços |

Não peça write scopes para tarefas de auditoria. Para writes financeiros, fiscais, estoque ou preço em massa, peça confirmação humana antes de aplicar.

## Tools esperadas

| Tool | Uso |
|------|-----|
| `get_olist_guide` | Guia embutido, 0 créditos |
| `list_olist_connections` | Contas conectadas, 0 créditos |
| `olist_get_catalog` | Lista os 178 endpoints REST v3 catalogados, scopes e path params |
| `olist_call_endpoint` | Executa endpoint REST v3 catalogado por `tool` |

Mapeamento HTTP Titanos: `POST /api/mcp/olist/guide`, `GET /api/mcp/olist/connections`, `POST /api/mcp/olist/catalog` e `POST /api/mcp/olist/call`.

Use somente recursos/operações catalogados. Nunca tente URL livre.

## Custos e quotas

Chamadas de setup e guias custam 0. Cada chamada REST v3 enviada à Olist custa 5 créditos ou consome quota inclusa do plano.

| Plano | Chamadas inclusas |
|-------|-------------------|
| Extensão | Sem acesso |
| Launch | 100 |
| Grow | 800 |
| Scale | 2000 |
| Dominate | 5000 |
| Titan | 10000 |
| Empire / Enterprise | Ilimitado |

Informe custo estimado antes de loops grandes. Retries internos e replay idempotente não devem cobrar novamente.

## Recursos Olist REST v3

Famílias principais: categorias, contas a pagar, contas a receber, contatos, CRM, dados da empresa, depósitos, estoque, expedição, logística, formas de pagamento/recebimento, intermediadores, listas de preços, marcas, notas fiscais, ordens de compra, ordens de serviço, pedidos, produtos, separação, serviços, tags de produtos, usuários e vendedores.

Dados financeiros, fiscais e de pedidos podem conter informações sensíveis. Resuma por padrão; só exponha dados pessoais completos se o usuário pedir e tiver permissão.

## Paginação e alto volume

Não tente carregar tudo de uma vez.

- Use filtros de data, SKU, situação ou página sempre que possível.
- Leia uma página, processe, então avance.
- Respeite `X-RateLimit-Remaining` e `X-RateLimit-Reset`.
- Não rode múltiplos loops paralelos na mesma `connection_id` sem necessidade.
- Para sincronização grande, prefira job assíncrono Titanos em vez de loop MCP longo.

## Idempotência

Para POST/PUT/DELETE:

1. Leia o estado atual quando possível.
2. Use `dry_run: true` se a tool suportar.
3. Gere `idempotency_key`.
4. Peça confirmação humana para operações irreversíveis ou financeiras/fiscais.
5. Se houver timeout, repita com a mesma `idempotency_key`; não crie outra.

Formato sugerido:

```text
olist:{connection_id}:{resource}:{operation}:{natural_id_or_hash}:{YYYYMMDDHHmm}
```

## Retries e rate limits

Retry apenas para 408, 429 e 5xx com backoff. Em 429, aguarde o reset informado. Não retry 400, 401, 403, 404 ou validação. Em 401, a conexão pode ter expirado; peça reautorização se o Titanos não conseguir refresh.

## Webhooks

Webhooks Olist são configurados dentro da conta ERP. A Olist espera HTTP 200 e pode reenviar até 10 vezes com delay progressivo. Eventos principais: vendas, pedidos enviados, estoque e notas fiscais autorizadas.

Quando analisar webhook:

- Trate como sinal, não como verdade final se a decisão for crítica.
- Reconsulte a Olist antes de ação financeira, fiscal ou de estoque.
- Não registre payload bruto com PII em respostas ao usuário.

## Prompts úteis

```text
Liste minhas conexões Olist no Titanos MCP e diga qual está ativa. Não faça writes.
```

```text
Use a conexão Olist ativa para listar produtos alterados nos últimos 7 dias. Processe página por página e respeite rate limit.
```

```text
Audite pedidos Olist dos últimos 3 dias e destaque pedidos sem rastreio. Não altere nada.
```

```text
Prepare uma atualização de estoque para o SKU ABC-123. Leia o saldo atual, rode dry_run e peça confirmação antes de aplicar.
```

```text
Consulte contas a receber vencidas no mês corrente e gere um resumo por cliente, sem expor CPF/CNPJ completo.
```

## Erros comuns

| Sintoma | Causa provável | Ação |
|---------|----------------|------|
| `SCOPE_MISSING` | API Key sem scope Olist | Recriar key com scope correto |
| `OLIST_CONNECTION_NOT_FOUND` | Conta não conectada ou org errada | Conectar/reautorizar no Titanos |
| `OLIST_CONNECTION_EXPIRED` | Refresh token venceu | Reautorizar OAuth |
| `OLIST_TOKEN_REFRESH_FAILED` | Falha ao renovar token | Reautorizar OAuth e conferir app Olist |
| `OLIST_RATE_LIMITED` | Limite Olist por conta estourado | Aguardar reset e reduzir paginação |
| Tool Olist não aparece | Package MCP local desatualizado | Atualizar `@titanos/mcp-agents` para versão com tools Olist |

## Fontes oficiais

- https://api-docs.erp.olist.com/documentacao/comecando/criando-um-aplicativo
- https://api-docs.erp.olist.com/documentacao/comecando/autenticacao
- https://api-docs.erp.olist.com/documentacao/comecando/limites-de-consulta
- https://api-docs.erp.olist.com/documentacao/webhooks/webhooks
- https://api-docs.erp.olist.com/api-reference/categorias/obter-categoria-por-identificador
