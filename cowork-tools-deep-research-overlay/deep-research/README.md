# Deep Research — Claude Cowork plugin

A deep, multi-source, **fact-checked** research skill for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork). It fans out web searches, fetches sources, tracks every claim to a cited source, **adversarially red-teams** those claims, and synthesizes a fully referenced report — using the **Firecrawl and/or Exa connectors** when they are enabled, and Cowork's **built-in web search and web fetch** otherwise — so it works with zero setup and gets better with connectors.

It is a port of the [199-biotechnologies/claude-deep-research-skill](https://github.com/199-biotechnologies/claude-deep-research-skill) pipeline, re-targeted from the Claude Code CLI (which uses an external `search-cli` + Python scripts) to Cowork's native tooling.

## Why this exists

Cowork has built-in web search/fetch, and there are official research-flavored skills (e.g. `customer-research` in [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins)) — but none reproduce the **adversarial verification loop** that separates real deep research from search-and-summarize. This plugin keeps that loop: a claim ships only if it survives a 3-persona red team and is cited to a fetched source.

## What it does (8-phase pipeline)

| Phase               | Action                                                                                    |
| ------------------- | ----------------------------------------------------------------------------------------- |
| 1. Scope            | Decompose the question into 4–6 independent search angles; state assumptions              |
| 2. Plan             | Draft a section outline mapped to angles _(skipped in `quick`)_                           |
| 3. Retrieve         | Parallel web searches per angle → **fetch pages** → store verbatim evidence               |
| 4. Triangulate      | Cross-validate each claim across independent sources; flag conflicts                      |
| 4.5. Outline refine | Evolve outline to match evidence; queue gap-filling searches                              |
| 5. Synthesize       | Draft prose with inline `[Sn]` citations on every factual sentence                        |
| 6. Critique         | **Red-team**: 3 adversarial personas; a claim dies if ≥2 refute it _(`deep`/`ultradeep`)_ |
| 7. Refine           | Delta-retrieve to shore up weak claims; re-verify (max 2 rounds)                          |
| 8. Package          | Assemble cited report against hard quality gates                                          |

### Modes

| Mode        | Phases              | Min sources | Red-team rounds |
| ----------- | ------------------- | ----------- | --------------- |
| `quick`     | 1,3,5,8             | 5           | 0               |
| `standard`  | 1–5,8               | 8           | 0               |
| `deep`      | 1–8                 | 10          | up to 2         |
| `ultradeep` | 1–8 (wider fan-out) | 15          | up to 2         |

## Search and fetch tools

| Operation | First choice       | Second choice    | Fallback            |
| --------- | ------------------ | ---------------- | ------------------- |
| Search    | `firecrawl_search` | `web_search_exa` | built-in web search |
| Fetch     | `firecrawl_scrape` | `web_fetch_exa`  | built-in web fetch  |

Connectors are optional. With them, the skill can fetch any public URL (the built-in fetch only reaches search results and URLs you share), parse PDFs, and filter by domain. Both providers bill per call. Setup: [`skills/deep-research/reference/SETUP.md`](skills/deep-research/reference/SETUP.md).

## Evidence ledgers

Each run writes to `~/Documents/Claude/Research/<topic>_<YYYYMMDD>/`:

- `run_manifest.json` — query, mode, assumptions, timestamp
- `sources.jsonl` — canonical source registry with stable IDs + quality tier
- `evidence.jsonl` — append-only verbatim quotes + locators
- `claims.jsonl` — atomic claims, per-source support, verdict, confidence
- `report.md` — final cited report

## Install

From the Cowork plugin marketplace (web): add this repository at [claude.com/plugins](https://claude.com/plugins/).

Or via the Claude CLI:

```bash
claude plugin marketplace add marcusk639/cowork-tools
claude plugin install deep-research@cowork-deep-research
```

> The marketplace manifest lives at `.claude-plugin/marketplace.json`; the plugin manifest at `.claude-plugin/plugin.json`.

## Usage

In Cowork, ask for research and the skill auto-activates, e.g.:

> "Do deep research on the trade-offs between FSEvents and kqueue for a macOS file-sync daemon."

Specify a mode explicitly if you want: _"...run this ultradeep."_

### Slash commands

The plugin also registers slash commands. Type `/` in Cowork and pick one, or invoke directly:

| Command                      | Effect                                                        |
| ---------------------------- | ------------------------------------------------------------- |
| `/deep-research <topic>`     | Runs the pipeline; mode inferred from the request (default `standard`). You may append a mode word. |
| `/deep-research-quick <topic>`     | Forces `quick` mode (5 sources, no red-team).           |
| `/deep-research-standard <topic>`  | Forces `standard` mode (8 sources).                     |
| `/deep-research-deep <topic>`      | Forces `deep` mode (10 sources, up to 2 red-team rounds). |
| `/deep-research-ultradeep <topic>` | Forces `ultradeep` mode (15 sources, wider fan-out).    |

Example: `/deep-research-ultradeep EKRA exposure for recovery-coaching referral arrangements`

> All commands invoke the same `skills/deep-research/SKILL.md`; the skill and its slash commands are one capability in the Cowork UI.

## Layout

```
deep-research/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace entry
├── commands/                # slash commands (/deep-research + mode aliases)
│   ├── deep-research.md
│   ├── deep-research-quick.md
│   ├── deep-research-standard.md
│   ├── deep-research-deep.md
│   └── deep-research-ultradeep.md
└── skills/deep-research/
    ├── SKILL.md             # the 8-phase pipeline
    └── reference/
        ├── SETUP.md         # enabling the optional Firecrawl/Exa connectors
        ├── methodology.md   # search/sourcing/evidence rules (read before Phase 3)
        └── quality-gates.md # red-team protocol + release checklist (Phase 6/8)
```

## Known limitations

- **Untested in the live Cowork runtime.** Two things to confirm on first run: (1) whether Cowork grants skill-driven fetches network access (its _built-in_ web fetch always works server-side), and (2) whether concurrent sub-agent searches are available for the Phase 3 fan-out. If sub-agents aren't available, retrieval degrades gracefully to sequential.
- Without a connector, Cowork's web fetch only reaches search-result URLs and URLs you've shared; sources no tool can reach are recorded as gaps, never paraphrased from memory.
- Built-in tools are referenced by capability ("web search", "web fetch") since Cowork's tool IDs aren't publicly documented. Connector tools are referenced by name but may appear with a namespace prefix.

## Credit

Pipeline architecture adapted from [199-biotechnologies/claude-deep-research-skill](https://github.com/199-biotechnologies/claude-deep-research-skill).
