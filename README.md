# cowork-tools

Tools, skills, and plugins for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork).

## Plugins

| Plugin                                                    | Skill name          | Web access                     | Setup               | Best for                                                                           |
| --------------------------------------------------------- | ------------------- | ------------------------------ | ------------------- | ---------------------------------------------------------------------------------- |
| [`deep-research-plugin`](./deep-research-plugin/)         | `deep-research`     | Cowork built-in search/fetch   | None                | Anyone — zero config, no API keys                                                  |
| [`deep-research-mcp-plugin`](./deep-research-mcp-plugin/) | `deep-research-mcp` | Firecrawl / Exa MCP connectors | Connector + API key | Users who already run Firecrawl/Exa and want richer search + clean scrape coverage |

Both are safe to install together — the skills have distinct names. Cowork selects
`deep-research-mcp` only when a Firecrawl/Exa connector is enabled; otherwise it
uses `deep-research`.

### deep-research (built-in)

Deep, multi-source, fact-checked research using **only** Cowork's built-in web
search and fetch — no API keys, no paid providers. Runs an **8-phase pipeline**
(scope → plan → retrieve → triangulate → synthesize → red-team critique → refine →
package) with evidence ledgers and configurable depth modes (`quick`, `standard`,
`deep`, `ultradeep`). The adversarial red-team loop (a claim ships only if it
survives a 3-persona review and is cited to a fetched source) is what separates it
from search-and-summarize.

→ See [`deep-research-plugin/README.md`](./deep-research-plugin/README.md).

### deep-research-mcp (Firecrawl / Exa)

The **MCP-powered sibling**. Same verification discipline, but reaches the web
through the **Firecrawl** and/or **Exa** MCP connectors instead of built-in tools —
gaining news filtering, clean markdown scrape, PDF parsing, and semantic search, at
the cost of connector setup and per-call billing. Runs a **5-phase pipeline**
(scope → search → fetch/extract → 3-pass adversarial verify → synthesize). Ported
from a Claude Code multi-agent `Workflow` harness (preserved in the plugin's
`reference/` for provenance).

→ See [`deep-research-mcp-plugin/README.md`](./deep-research-mcp-plugin/README.md)
and its [`SETUP.md`](./deep-research-mcp-plugin/skills/deep-research-mcp/reference/SETUP.md).

## Which should I use?

- **No connectors / want it to just work →** `deep-research`.
- **Already pay for Firecrawl or Exa and want their richer coverage →** `deep-research-mcp`.
- **On managed/enterprise "Cowork on 3P" →** `deep-research` works out of the box;
  `deep-research-mcp` needs an admin to provision the connectors. See the MCP
  plugin's SETUP.md.

## Installing a plugin in Cowork

Add the plugin directory as a source from Cowork's **Plugins** settings (each
plugin ships a `.claude-plugin/marketplace.json`). For the MCP plugin, also enable
a Firecrawl or Exa connector under **Settings → Connectors** first.

## Repository layout

```
cowork-tools/
├── deep-research-plugin/         # built-in-web-tools research plugin (8-phase)
│   └── skills/deep-research/
├── deep-research-mcp-plugin/     # Firecrawl/Exa MCP research plugin (5-phase)
│   └── skills/deep-research-mcp/
│       ├── SKILL.md
│       └── reference/            # SETUP, methodology, quality-gates, provenance
└── .planning/                    # project planning docs
```
