# Prompt Engineer — Claude Cowork plugin

Helps you write, improve and debug prompts for Claude, based on Anthropic's published prompt-engineering guidance. It auto-activates when you ask for help with a prompt or system prompt.

## What it does

A 7-step workflow: understand the requirements → pick techniques → load the matching reference → design the prompt → add quality controls → test → iterate. When it hands you a prompt, it explains which techniques it used and why.

| Need                                      | Technique                     | Reference                |
| ----------------------------------------- | ----------------------------- | ------------------------ |
| Clarity, role, structure                  | Clear instructions, system prompts, XML tags | `core_prompting.md`      |
| Reasoning, formats, multi-step, long docs | Chain of thought, multishot, prompt chaining, long context, extended thinking | `advanced_patterns.md`   |
| Accuracy, consistency, safety             | Hallucination reduction, consistency, jailbreak mitigation | `quality_improvement.md` |

## Usage

In Cowork, ask for help with a prompt, e.g.:

> "Improve this system prompt for a support bot — answers are inconsistent and sometimes made up."

No connectors or setup needed.

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
