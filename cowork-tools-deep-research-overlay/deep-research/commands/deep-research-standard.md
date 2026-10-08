---
description: Run deep-research in standard mode
argument-hint: [topic]
---

Invoke the **deep-research** skill (`skills/deep-research/SKILL.md`) in **standard** mode on the following request:

$ARGUMENTS

Run the pipeline with mode fixed to `standard` (do not re-infer the mode from the text). Apply that mode's source floor and red-team rounds as defined in the skill's mode table. Read `reference/methodology.md` before Phase 3 and `reference/quality-gates.md` before Phase 6/8. Use the Firecrawl/Exa connectors when enabled, falling back to the built-in web search and web fetch tools, as set out in the skill's Tooling contract. No claim ships unverified or uncited.
