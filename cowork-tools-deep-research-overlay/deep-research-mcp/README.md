# Deep Research (MCP) — Claude Cowork plugin

A deep, multi-source, **fact-checked** research skill for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork), powered by the **Firecrawl** and/or **Exa** MCP connectors. It fans out web searches, scrapes sources, tracks every claim to a cited source, **adversarially red-teams** those claims, and synthesizes a fully referenced report.

This is the **MCP-powered sibling** of the [`deep-research`](../deep-research-plugin/) plugin in this repo. They share the same verification discipline; they differ in how they reach the web:

|            | [`deep-research`](../deep-research-plugin/) | `deep-research-mcp` (this plugin)                                                                                                 |
| ---------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Web access | Cowork's **built-in** web search/fetch      | **Firecrawl** / **Exa** MCP connectors                                                                                            |
| Setup      | None                                        | Add a connector + API key ([SETUP.md](skills/deep-research-mcp/reference/SETUP.md))                                               |
| Cost       | Included                                    | Firecrawl/Exa bill per call                                                                                                       |
| Best for   | Anyone, zero config                         | Users who already run Firecrawl/Exa and want richer search + clean scrape coverage (news filtering, PDF parsing, semantic search) |

The two skills are named **`deep-research`** and **`deep-research-mcp`**, so both
can be installed at once without colliding. Cowork picks `deep-research-mcp` only
when a Firecrawl/Exa connector is enabled; otherwise it uses `deep-research`.

## Provenance

Single-agent port of a Claude Code multi-agent `Workflow` harness (the original
349-line orchestration script — `agent()` fan-out, `pipeline()`/`parallel()`,
JSON output schemas, 3-vote adversarial verification — is preserved verbatim at
[`skills/deep-research-mcp/reference/original-claude-code-workflow.js`](skills/deep-research-mcp/reference/original-claude-code-workflow.js)). The `Workflow` runtime is Claude-Code-only, so the five phases were re-expressed as sequential instructions one Cowork agent executes itself; the parallel voter agents become inline skeptical passes.

## What it does (5-phase pipeline)

| Phase              | Action                                                                                                            |
| ------------------ | ----------------------------------------------------------------------------------------------------------------- |
| 1. Scope           | Decompose the question into 5 independent search angles; state assumptions                                        |
| 2. Search          | Per-angle search via `firecrawl_search` / `web_search_exa`; dedup; cap at ~15 sources                             |
| 3. Fetch & extract | Scrape each source (`firecrawl_scrape` / `web_fetch_exa`); grade quality; pull 2–5 falsifiable claims with quotes |
| 4. Verify          | **3-pass adversarial red team** per claim — a claim dies if ≥2 passes refute it                                   |
| 5. Synthesize      | Merge survivors, rank by confidence, write a fully cited report                                                   |

## Reference docs

The skill loads these on demand during the relevant phase:

- [`reference/SETUP.md`](skills/deep-research-mcp/reference/SETUP.md) — enabling the Firecrawl/Exa connectors.
- [`reference/methodology.md`](skills/deep-research-mcp/reference/methodology.md) — search discipline, fetch-before-cite, source-quality tiers, atomic claims.
- [`reference/quality-gates.md`](skills/deep-research-mcp/reference/quality-gates.md) — the 2-of-3 adversarial kill protocol, iteration budget, release checklist.
- [`reference/original-claude-code-workflow.js`](skills/deep-research-mcp/reference/original-claude-code-workflow.js) — original Claude Code harness (provenance).

## Requirements

At least one of the **Firecrawl** or **Exa** remote MCP connectors enabled in Cowork. See [SETUP.md](skills/deep-research-mcp/reference/SETUP.md) for adding them (standard Cowork vs. managed "Cowork on 3P"), and a note on shipping connectors bundled with the plugin.

## Slash command

The plugin registers a slash command. Type `/` in Cowork and pick it, or invoke directly:

| Command                      | Effect                                                                 |
| ---------------------------- | ---------------------------------------------------------------------- |
| `/deep-research-mcp <topic>` | Runs the full 5-phase Firecrawl/Exa pipeline on `<topic>`.             |

Example: `/deep-research-mcp competitive landscape for AI code editors in 2026`

The command invokes `skills/deep-research-mcp/SKILL.md`; the skill and its slash command are one capability in the Cowork UI. If no Firecrawl/Exa connector is enabled, it will stop and point you to `reference/SETUP.md` (or defer to the built-in `deep-research` skill).

## Install

Open the `.plugin` file with the Claude desktop app (Cowork's native plugin installer), or add this directory as a plugin source in Cowork (Plugins settings) / via the `marketplace.json` in `.claude-plugin/`. Then enable a Firecrawl/Exa connector and ask Cowork to research something.
