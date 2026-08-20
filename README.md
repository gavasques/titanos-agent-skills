# Titanos agent skills

Skills de vendedor que o TITANOS publica em [Conta → Integrações](https://www.titanos.com.br/conta/integracoes). Fonte para agentes (Cursor, Claude, Codex, Grok Bot).

Cada pasta em `skills/` tem um `SKILL.md`. IDs iguais ao catálogo do produto.

## Como o agente usa

Leia o `SKILL.md` pelo raw do GitHub, por exemplo:

`https://raw.githubusercontent.com/gavasques/titanos-agent-skills/main/skills/amazon-ads/SKILL.md`

Ou clone este repo e aponte o cliente de skills para `skills/`.

Dependência comum: conectar o MCP TITANOS (`@titanos/mcp-agents`) com API key e OAuth Amazon. Comece por `skills/titanos-mcp-setup`.

## Seções

- **essenciais** — setup MCP, mineração, listing
- **ads** — Sponsored Products, PPC, reports, DSP
- **seller** — Seller Central, FBA, auditoria
- **erp** — Olist Tiny, Bling

Catálogo máquina: `catalog.json`.

## Instalar como plugin (Claude Code / Cursor)

Mesmo formato do [skill-amazon-ads](https://github.com/MarketplaceAdPros/skill-amazon-ads):

```
.claude-plugin/plugin.json
skills/<nome>/SKILL.md
.mcp.json
```

No Claude Code:

```bash
claude --plugin-dir ./titanos-agent-skills
```

Ou adicione o GitHub como marketplace/plugin e instale `titanos`. As skills aparecem namespaced (`/titanos:amazon-ads`, `/titanos:super-listing-power`, …).

O MCP sobe com `npx @titanos/mcp-agents@1`. Defina `TITANOS_API_KEY` no ambiente (não commitar a key).
