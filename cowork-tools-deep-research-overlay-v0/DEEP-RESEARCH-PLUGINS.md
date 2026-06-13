# Deep-research plugins in this marketplace

This repo (`marcusk639/cowork-tools`) serves as a Claude Cowork **plugin marketplace**. The `.claude-plugin/marketplace.json` at the repo root lists the plugins below; each lives in its own top-level folder.

| Plugin                                      | Web access                          | Setup                            | Slash commands                                                               |
| ------------------------------------------- | ----------------------------------- | -------------------------------- | ---------------------------------------------------------------------------- |
| [`deep-research`](./deep-research)         | Cowork built-in web search / fetch  | None                             | `/deep-research`, `/deep-research-quick`, `-standard`, `-deep`, `-ultradeep` |
| [`deep-research-mcp`](./deep-research-mcp) | Firecrawl and/or Exa MCP connectors | Enable a Firecrawl/Exa connector | `/deep-research-mcp`                                                         |

## Install (no terminal — most users)

1. In the Claude desktop app, open **Customize → Plugins**.
2. Click **Add marketplace** and enter `marcusk639/cowork-tools` (or the full `https://github.com/marcusk639/cowork-tools` URL).
3. Both plugins appear under the marketplace — click **Install** on each one you want.
4. For `deep-research-mcp`, also enable a Firecrawl or Exa connector under **Settings → Connectors** (see `deep-research-mcp/skills/deep-research-mcp/reference/SETUP.md`).

> If the repo is **private**, make sure the Claude GitHub App is installed on it so Cowork can sync.

## Install (CLI — developers)

```bash
claude plugin marketplace add marcusk639/cowork-tools
claude plugin install deep-research@cowork-tools
claude plugin install deep-research-mcp@cowork-tools
```

Once installed, skills fire automatically when relevant and the slash commands above are available in any Cowork task.

## Adding more plugins later

Drop a new `<plugin>/` folder at the repo root (with its own `.claude-plugin/plugin.json`) and add an entry to `.claude-plugin/marketplace.json` with `"source": "./<plugin>"`. Push, then click **Update** on the marketplace in Cowork.
