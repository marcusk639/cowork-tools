# Prompt Engineer — Claude Cowork plugin

Helps you write, improve and debug prompts for Claude, based on Anthropic's prompt-engineering guidance and updated for current models (Opus 5.5, Sonnet 5.5, Fable 5.1 and the 4.6+ family). It activates when you ask for help with a prompt, system prompt, skill or agent instructions.

## What it does

A 7-step workflow: understand the requirements → pick techniques → load the matching reference → design the prompt → add quality controls → test → iterate. When it hands you a prompt, it explains which techniques it used and why.

It writes for how current models behave:

- States outcomes and constraints instead of scripting every step.
- Writes at normal volume and attaches a reason to each rule, instead of stacked `CRITICAL` / `MUST` / `NEVER`.
- Uses a few varied, illustrative examples, because Claude copies the shape of its examples.
- Uses structured outputs (`output_config.format` or `strict: true` tools) for fixed formats. Assistant-turn prefill returns an error on current models.
- Controls reasoning depth with adaptive thinking and `effort`, not "think step by step" or `budget_tokens`.
- Flags leftover workarounds for older models when improving an existing prompt.

| Need                                      | Technique                                                                     | Reference                |
| ----------------------------------------- | ----------------------------------------------------------------------------- | ------------------------ |
| Clarity, role, structure                  | Clear instructions, system prompts, XML tags                                  | `core_prompting.md`      |
| Reasoning, formats, multi-step, long docs | Chain of thought, multishot, prompt chaining, long context, adaptive thinking | `advanced_patterns.md`   |
| Accuracy, consistency, safety             | Hallucination reduction, structured outputs, jailbreak mitigation             | `quality_improvement.md` |

## When to use it

Use it for prompts you'll run in Cowork or Claude Code: task instructions, system prompts, skills, agent definitions. For building evaluation suites, A/B tests or prompts for other providers, use [`llm-prompt-engineer`](../llm-prompt-engineer).

## Usage examples

Ask in plain language and paste the prompt you're working on. No connectors or setup needed.

**Fix a prompt that misbehaves**

> "Improve this system prompt for a support bot — answers are inconsistent and sometimes made up: [paste]"

You get a revised prompt, a list of what changed and why (for example: added permission to say "I don't know", grounded answers in quoted policy text), and a few test inputs to try.

**Write a new prompt from a goal**

> "Write a prompt that turns meeting transcripts into action items with owner and due date. It runs weekly on about 20 transcripts."

It asks what it needs (audience, format, edge cases), then drafts the prompt with structure and a couple of varied examples, and suggests structured outputs if the result feeds another tool.

**Modernize an old prompt for current models**

> "This prompt was written for Claude 3. Update it for Opus 5.5: [paste]"

It removes stacked `MUST`/`NEVER`, "think step by step", hard word caps and prefill, and keeps the context and real constraints.

**Tune a skill or agent description**

> "Claude keeps picking the wrong agent. Rewrite this description so it's chosen for contract review and not general proofreading: [paste]"

**Diagnose a specific failure**

> "My extraction prompt returns valid JSON about 90% of the time and prose the rest. Why, and how do I fix it?"

It points to the cause and the fix, here structured outputs (`output_config.format`) instead of format instructions in prose.

## Install

In Cowork: **Customize → Plugins → `cowork-tools` → `prompt-engineer` → Install**. CLI:

```bash
claude plugin install prompt-engineer@cowork-tools
```

## Layout

```
prompt-engineer/
├── .claude-plugin/plugin.json
└── skills/prompt-engineer/
    ├── SKILL.md
    └── references/
        ├── core_prompting.md
        ├── advanced_patterns.md
        └── quality_improvement.md
```
