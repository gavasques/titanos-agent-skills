# TITANOS agent skills

[Site](https://www.titanos.com.br) · [Integrações / Integrations](https://www.titanos.com.br/conta/integracoes) · [MCP npm `@titanos/mcp-agents`](https://www.npmjs.com/package/@titanos/mcp-agents)

---

## Português

### O que é o TITANOS

O [TITANOS](https://www.titanos.com.br) é uma plataforma brasileira de operação para vendedores em marketplaces (Amazon e outros canais). A conta, as conexões de loja e o painel ficam no site oficial — não neste repositório. Para usar as integrações com agentes, você precisa de uma conta TITANOS e de uma API key criada lá.

### O que é deste repositório

Este repositório é o **pacote público de skills oficiais de vendedor** que o TITANOS publica em [Conta → Integrações](https://www.titanos.com.br/conta/integracoes). Cada pasta em `skills/` contém um `SKILL.md`: receitas para agentes (Cursor, Claude Code e clientes compatíveis). Os IDs são os mesmos do catálogo do produto. **Não é o aplicativo TITANOS** nem o servidor MCP.

### O que NÃO está aqui

Não há chaves de API, credenciais de vendedor, IDs de Ads nem dados de contas reais. Crie a API key na conta TITANOS (**Conta → Configurações / MCP**) e use-a só no ambiente local (`TITANOS_API_KEY`). Nunca commite a key.

### Comece no site TITANOS

1. Abra [titanos.com.br](https://www.titanos.com.br)
2. Veja as skills em [Conta → Integrações](https://www.titanos.com.br/conta/integracoes)

### Seções

- **essenciais** — setup do MCP, mineração, listing
- **ads** — Sponsored Products, PPC, relatórios, DSP
- **seller** — Seller Central, FBA, auditoria
- **erp** — Olist Tiny, Bling

---

## English

### What TITANOS is

[TITANOS](https://www.titanos.com.br) is a Brazilian seller operations platform for Amazon and other marketplaces. Your account, store connections, and dashboard live on the official site — not in this repository. To use agent integrations you need a TITANOS account and an API key created there.

### What this repo is

This repository is the **public pack of official seller skills** published by TITANOS on [Account → Integrations](https://www.titanos.com.br/conta/integracoes). Each folder under `skills/` has a `SKILL.md`: recipes for agents (Cursor, Claude Code, and compatible clients). IDs match the product catalog. **This is not the TITANOS app** and not the MCP server.

### What is NOT here

There are no API keys, seller credentials, Ads IDs, or live account data. Create the API key in your TITANOS account (**Account → Settings / MCP**) and use it only in the local environment (`TITANOS_API_KEY`). Never commit the key.

### Start on the TITANOS site

1. Open [titanos.com.br](https://www.titanos.com.br)
2. See the skills at [Account → Integrations](https://www.titanos.com.br/conta/integracoes)

### Sections

- **essenciais** — MCP setup, mining, listing
- **ads** — Sponsored Products, PPC, reports, DSP
- **seller** — Seller Central, FBA, audit
- **erp** — Olist Tiny, Bling

---

## Como usar / How to use

### skills.sh

**PT — Instalar o pack** / **EN — Install the pack:**

```bash
npx skills add gavasques/titanos-agent-skills
```

**PT — Instalar uma skill** / **EN — Install one skill** (mesmo formato da página find-skills / same shape as the find-skills page):

```bash
npx skills add https://github.com/gavasques/titanos-agent-skills --skill amazon-ads
```

**PT — Páginas canônicas** / **EN — Canonical pages** (aparecem após o primeiro install / appear after the first install):

- https://www.skills.sh/gavasques/titanos-agent-skills
- https://www.skills.sh/gavasques/titanos-agent-skills/amazon-ads
- Padrão / pattern: `https://www.skills.sh/gavasques/titanos-agent-skills/<skill-name>`

O listing no skills.sh é criado por telemetria do CLI (`npx skills add`), não por um formulário de submissão. / The skills.sh listing is created by CLI telemetry (`npx skills add`), not a submit form.

**Manifests** — o mesmo `skills/` entra em cada cliente por um manifesto diferente. / The same `skills/` tree is loaded through a different manifest per client.

| Cliente / Client | Manifest |
| --- | --- |
| Claude Code / VS Code Copilot | `.claude-plugin/plugin.json` |
| Cursor | `.cursor-plugin/plugin.json` or root `plugin.json` |
| Codex / ChatGPT | `.codex-plugin/plugin.json` |

Cursor: marketplace or plugin directory. Codex: [plugin install docs](https://developers.openai.com/codex/plugins/build). Claude: `claude --plugin-dir ./titanos-agent-skills`.

**URL bruta / raw GitHub** — o agente lê o `SKILL.md` pelo raw, por exemplo:

`https://raw.githubusercontent.com/gavasques/titanos-agent-skills/main/skills/amazon-ads/SKILL.md`

Ou clone o repo e aponte o cliente de skills para `skills/`. / Or clone the repo and point the skills client at `skills/`.

**Plugin (Claude Code / Cursor)**

```bash
claude --plugin-dir ./titanos-agent-skills
```

As skills aparecem com namespace (`/titanos:amazon-ads`, `/titanos:super-listing-power`, …). / Skills are namespaced (`/titanos:amazon-ads`, `/titanos:super-listing-power`, …).

**MCP + API key** — o MCP sobe com `npx @titanos/mcp-agents@1`. Defina `TITANOS_API_KEY` no ambiente (não commitar). / Start MCP with `npx @titanos/mcp-agents@1`. Set `TITANOS_API_KEY` in the environment (do not commit it).

**Comece por / Start here:** `skills/titanos-mcp-setup`

---

## Catálogo / Catalog

Catálogo máquina / machine catalog: [`catalog.json`](catalog.json).

| Seção / Section | IDs |
| --- | --- |
| essenciais | `titanos-mcp-setup`, `amazon-mining-research`, `amazon-market-entry`, `amazon-pricing-signals`, `amazon-keyword-intelligence`, `amazon-competitor-radar`, `super-listing-power`, `listing-create-publish` |
| ads | `amazon-ads`, `titanos-ads-quickstart`, `amazon-sp-campaign-launcher`, `amazon-ads-keyword-target-builder`, `amazon-ads-write-recovery`, `amazon-ads-launch-playbook`, `amazon-ads-reports-troubleshooting`, `amazon-ads-optimization`, `wasted-ad-spend-dashboard`, `search-term-harvest-dashboard`, `amazon-dsp` |
| seller | `amazon-seller-central`, `fba-inventory-risk-dashboard`, `amazon-reorder-planning`, `amazon-listing-audit` |
| erp | `olist-tiny-erp`, `bling-erp` |

---

## Licença / License

[MIT](LICENSE). Copyright (c) 2026 Titanos.
