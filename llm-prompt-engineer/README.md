# LLM Prompt Engineer — Claude Cowork plugin

Designs, optimizes and evaluates prompts for LLM applications, with an emphasis on measurement: test suites, evaluation rubrics, A/B testing, structured-output schemas and context management. It is provider-neutral, with examples for both Anthropic and OpenAI APIs.

| Reference                  | Covers                                               |
| -------------------------- | ---------------------------------------------------- |
| `prompt-patterns.md`       | Zero-shot, few-shot, chain-of-thought, ReAct         |
| `prompt-optimization.md`   | Iterative refinement, A/B testing, token reduction   |
| `evaluation-frameworks.md` | Metrics, test suites, LLM-as-judge, CI integration   |
| `structured-outputs.md`    | JSON mode, function calling, schema design           |
| `system-prompts.md`        | Persona design, guardrails, injection defense        |
| `context-management.md`    | Attention budget, degradation patterns, optimization |

## When to use it

Use it when you're building an LLM-powered app and need to measure prompt quality: an eval set, a scoring rubric, an A/B comparison, or a prompt that has to work across providers.

For writing or fixing prompts you'll use in Cowork or Claude Code, the lighter [`prompt-engineer`](../prompt-engineer) plugin is the better fit. It is Claude-specific and current for the latest models. This plugin's code examples pin older model IDs (such as `claude-opus-4-5-20251101`), so check model names and API details against current docs.

## Usage examples

No connectors or setup needed.

**Build an eval set**

> "Build a test suite and grading rubric for this ticket-classification prompt. Categories: billing, bug, feature request, other."

You get test cases covering typical inputs, edge cases and ambiguous tickets, a scoring rubric, and a way to run it (exact match for labels, an LLM judge for free text).

**Compare two prompts**

> "Design an A/B test comparing these two system prompts for a sales-email writer: [paste both]"

It defines the metrics, sample size and judging criteria, and how to decide a winner.

**Shrink a prompt without losing quality**

> "This prompt is 3,000 tokens. Cut it down and show me how to check nothing got worse."

**Design a structured-output schema**

> "Design a JSON schema for extracting invoice fields (vendor, date, line items, totals) and the prompt to go with it."

**Harden a public-facing system prompt**

> "Add guardrails and prompt-injection defenses to this customer-facing assistant prompt: [paste]"

**Move a prompt between providers**

> "This prompt works on GPT; adapt it for Claude and list what to retest."

## Install

In Cowork: **Customize → Plugins → `cowork-tools` → `llm-prompt-engineer` → Install**. CLI:

```bash
claude plugin install llm-prompt-engineer@cowork-tools
```

## Layout

```
llm-prompt-engineer/
├── .claude-plugin/plugin.json
├── LICENSE
└── skills/llm-prompt-engineer/
    ├── SKILL.md
    └── references/          # the 6 references above
```

## Credit and license

Packaged from the `prompt-engineer` skill (v1.2.0) in [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) by [@jeffallan](https://github.com/jeffallan) ([Synergetic Solutions](https://synergetic.solutions)). `references/context-management.md` is adapted from a contribution by [Genius-apple](https://github.com/Genius-apple) ([PR #168](https://github.com/Jeffallan/claude-skills/pull/168)).

Licensed under the MIT License — see [`LICENSE`](LICENSE), copied verbatim from the upstream repository. The only change from upstream is the skill `name`, renamed from `prompt-engineer` to `llm-prompt-engineer` so it doesn't collide with this marketplace's `prompt-engineer` skill.
