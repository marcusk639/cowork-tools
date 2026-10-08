# Prompt Engineer (Advanced) — Claude Cowork plugin

Designs, optimizes and evaluates prompts for LLM applications, with an emphasis on measurement: test suites, evaluation rubrics, A/B testing, structured-output schemas and context management. For quick prompt writing and fixing, see the lighter [`prompt-engineer`](../prompt-engineer) plugin.

| Reference                     | Covers                                                   |
| ----------------------------- | -------------------------------------------------------- |
| `prompt-patterns.md`          | Zero-shot, few-shot, chain-of-thought, ReAct             |
| `prompt-optimization.md`      | Iterative refinement, A/B testing, token reduction       |
| `evaluation-frameworks.md`    | Metrics, test suites, LLM-as-judge, CI integration       |
| `structured-outputs.md`       | JSON mode, function calling, schema design               |
| `system-prompts.md`           | Persona design, guardrails, injection defense            |
| `context-management.md`       | Attention budget, degradation patterns, optimization     |

No connectors or setup needed.

## Credit and license

Packaged from the `prompt-engineer` skill (v1.2.0) in [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) by [@jeffallan](https://github.com/jeffallan) ([Synergetic Solutions](https://synergetic.solutions)). `references/context-management.md` is adapted from a contribution by [Genius-apple](https://github.com/Genius-apple) ([PR #168](https://github.com/Jeffallan/claude-skills/pull/168)).

Licensed under the MIT License — see [`LICENSE`](LICENSE), copied verbatim from the upstream repository. The only change from upstream is the skill `name`, renamed from `prompt-engineer` to `llm-prompt-engineer` so it doesn't collide with this marketplace's `prompt-engineer` skill.
