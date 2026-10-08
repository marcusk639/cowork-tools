# cowork-tools

Tools, skills, and plugins for [Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork).

## Plugins

This repo is a Claude Cowork **plugin marketplace**: `.claude-plugin/marketplace.json`
at the repo root lists the plugins, and each plugin lives in its own top-level folder.

| Plugin                             | Skill name      | Web access                                                    | Setup                      |
| ---------------------------------- | --------------- | ------------------------------------------------------------- | -------------------------- |
| [`deep-research`](./deep-research) | `deep-research` | Firecrawl/Exa connectors if enabled, else Cowork built-in web search/fetch | None (connectors optional) |
| [`prompt-engineer`](./prompt-engineer) | `prompt-engineer` | None | None |
| [`llm-prompt-engineer`](./llm-prompt-engineer) | `llm-prompt-engineer` | None | None |
| [`cowork-agents`](./cowork-agents) | Agents: `research-analyst`, `verifier` | Session's web tools / connectors | None |

### Which one should I use?

| You want to…                                                        | Use                                  |
| ------------------------------------------------------------------- | ------------------------------------ |
| Produce a long, fact-checked, fully cited research report           | `deep-research` skill                |
| Research one part of a bigger task, or several questions in parallel | `cowork-agents:research-analyst`     |
| Check one specific claim before you rely on it                       | `cowork-agents:verifier`             |
| Write or fix a prompt, skill or agent for Cowork / Claude Code       | `prompt-engineer` skill              |
| Build evals, rubrics or A/B tests for an LLM app                     | `llm-prompt-engineer` skill          |

### Quick examples

```text
/deep-research-deep evidence that peer recovery coaching improves 12-month outcomes
Use the research-analyst to compare pricing for the top three AI scribe vendors, with sources.
Use the verifier on this claim before I send it: "Ambient AI documentation cuts charting time by 50%."
Improve this system prompt for a support bot — answers are inconsistent: [paste]
Build a test suite and grading rubric for this ticket-classification prompt.
```

Each plugin README below has more examples.

### deep-research

Deep, multi-source, fact-checked research. Runs an **8-phase pipeline** (scope → plan →
retrieve → triangulate → synthesize → red-team critique → refine → package) with
evidence ledgers and depth modes (`quick`, `standard`, `deep`, `ultradeep`). A claim
ships only if it is cited to a fetched source and survives a 3-persona red-team review.
Slash commands: `/deep-research` plus one per mode.

It searches and fetches with the **Firecrawl and/or Exa** connectors when they are
enabled and falls back to Cowork's built-in web search and fetch otherwise, so it works
with zero setup.

→ [`deep-research/README.md`](./deep-research/README.md) · connector
[`SETUP.md`](./deep-research/skills/deep-research/reference/SETUP.md)

### prompt-engineer

Writes, improves and debugs prompts for Claude, updated for current models: outcome-focused
instructions at normal volume, varied examples, structured outputs instead of prefill, and
adaptive thinking with `effort` instead of thinking budgets.

→ [`prompt-engineer/README.md`](./prompt-engineer/README.md)

### llm-prompt-engineer

Measurement-focused, provider-neutral prompt engineering: evaluation frameworks and test
suites, A/B testing, structured-output schemas, system prompts with guardrails, and context
management. Packaged from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) (MIT).

→ [`llm-prompt-engineer/README.md`](./llm-prompt-engineer/README.md)

### cowork-agents

Subagents Claude can hand work to inside a Cowork task. Each runs in its own context and
returns only its result.

- **`research-analyst`** (Sonnet) — multi-source research and synthesis with cited findings.
- **`verifier`** (Opus) — re-checks one specific claim from primary sources and returns
  CONFIRMED / REFUTED / PLAUSIBLE with the evidence.

Neither agent restricts its tools, so each uses whatever the session has, including
Firecrawl or Exa when enabled.

→ [`cowork-agents/README.md`](./cowork-agents/README.md)

## Installing in Cowork

1. In the Claude desktop app, open **Customize → Plugins**.
2. Click **Add marketplace** and enter `marcusk639/cowork-tools`.
3. Click **Install** on the plugins you want: `deep-research`, `prompt-engineer`, `llm-prompt-engineer`, `cowork-agents`.
4. Optional: enable a Firecrawl or Exa connector under **Settings → Connectors**, then
   switch it on in a task via **"+" → Connectors**.

After pushing changes, click **Update** on the marketplace in Cowork.

> If the repo is **private**, install the Claude GitHub App on it so Cowork can sync.

CLI:

```bash
claude plugin marketplace add marcusk639/cowork-tools
claude plugin install deep-research@cowork-tools
claude plugin install prompt-engineer@cowork-tools
claude plugin install llm-prompt-engineer@cowork-tools
claude plugin install cowork-agents@cowork-tools
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
├── llm-prompt-engineer/          # third-party (MIT) prompt-engineering plugin
│   ├── .claude-plugin/plugin.json
│   ├── LICENSE
│   └── skills/llm-prompt-engineer/
│       ├── SKILL.md
│       └── references/           # 6 topic references
├── cowork-agents/                # subagents plugin
│   ├── .claude-plugin/plugin.json
│   └── agents/                   # research-analyst.md, verifier.md
└── .planning/                    # project planning docs
```
