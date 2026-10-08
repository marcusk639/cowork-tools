# cowork-tools

Tools, skills, and plugins for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork).

## Plugins

This repo is a Claude Cowork **plugin marketplace**: `.claude-plugin/marketplace.json`
at the repo root lists the plugins, and each plugin lives in its own top-level folder.

| Plugin                             | Skill name      | Web access                                                    | Setup                      |
| ---------------------------------- | --------------- | ------------------------------------------------------------- | -------------------------- |
| [`deep-research`](./deep-research) | `deep-research` | Firecrawl/Exa connectors if enabled, else Cowork built-in web search/fetch | None (connectors optional) |
| [`prompt-engineer`](./prompt-engineer) | `prompt-engineer` | None | None |
| [`prompt-engineer-advanced`](./prompt-engineer-advanced) | `prompt-engineer-advanced` | None | None |

### deep-research

Deep, multi-source, fact-checked research. Runs an **8-phase pipeline** (scope → plan →
retrieve → triangulate → synthesize → red-team critique → refine → package) with
evidence ledgers and depth modes (`quick`, `standard`, `deep`, `ultradeep`). A claim
ships only if it is cited to a fetched source and survives a 3-persona red-team review.

It searches and fetches with the **Firecrawl and/or Exa** connectors when they are
enabled (any public URL, PDF parsing, domain filters, semantic search) and falls back
to Cowork's built-in web search and fetch otherwise, so it works with zero setup.

→ See [`deep-research/README.md`](./deep-research/README.md) and the connector
[`SETUP.md`](./deep-research/skills/deep-research/reference/SETUP.md).

### prompt-engineer

Helps write, improve and debug prompts for Claude using Anthropic's prompt-engineering
guidance: clarity, XML structure, system prompts, chain of thought, multishot examples,
prompt chaining, hallucination reduction, consistency and jailbreak mitigation.

→ See [`prompt-engineer/README.md`](./prompt-engineer/README.md).

### prompt-engineer-advanced

Measurement-focused prompt engineering: evaluation frameworks and test suites, A/B
testing, structured-output schemas, system prompts with guardrails, and context
management. Packaged from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) (MIT).

→ See [`prompt-engineer-advanced/README.md`](./prompt-engineer-advanced/README.md).

## Installing in Cowork

1. In the Claude desktop app, open **Customize → Plugins**.
2. Click **Add marketplace** and enter `marcusk639/cowork-tools`.
3. Click **Install** on the plugins you want: `deep-research`, `prompt-engineer`, `prompt-engineer-advanced`.
4. Optional: enable a Firecrawl or Exa connector under **Settings → Connectors**, then
   switch it on in a task via **"+" → Connectors**.

After pushing changes, click **Update** on the marketplace in Cowork.

> If the repo is **private**, install the Claude GitHub App on it so Cowork can sync.

CLI:

```bash
claude plugin marketplace add marcusk639/cowork-tools
claude plugin install deep-research@cowork-tools
claude plugin install prompt-engineer@cowork-tools
claude plugin install prompt-engineer-advanced@cowork-tools
```

### Adding a plugin

Add a `<plugin>/` folder at the repo root with its own `.claude-plugin/plugin.json`,
then add an entry to `.claude-plugin/marketplace.json` with `"source": "./<plugin>"`.

## Repository layout

```
cowork-tools/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest (lists the plugins)
├── deep-research/                # the deep-research plugin
│   ├── .claude-plugin/plugin.json
│   ├── commands/                 # /deep-research + mode aliases
│   └── skills/deep-research/
│       ├── SKILL.md
│       └── reference/            # SETUP, methodology, quality-gates
├── prompt-engineer/              # the prompt-engineer plugin
│   ├── .claude-plugin/plugin.json
│   └── skills/prompt-engineer/
│       ├── SKILL.md
│       └── references/           # core, advanced, quality-improvement
├── prompt-engineer-advanced/     # third-party (MIT) prompt-engineering plugin
│   ├── .claude-plugin/plugin.json
│   ├── LICENSE
│   └── skills/prompt-engineer-advanced/
│       ├── SKILL.md
│       └── references/           # 6 topic references
└── .planning/                    # project planning docs
```
