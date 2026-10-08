---
description: Run the multi-source, fact-checked deep-research pipeline
argument-hint: [topic] (optionally end with: quick | standard | deep | ultradeep)
---

Invoke the **deep-research** skill (`skills/deep-research/SKILL.md`) on the following request:

$ARGUMENTS

Follow the skill's full 8-phase pipeline. Determine the mode from the request: if the user named a mode (`quick`, `standard`, `deep`, `ultradeep`), use it; otherwise default to `standard`. Read `reference/methodology.md` before Phase 3 (RETRIEVE) and `reference/quality-gates.md` before Phase 6 (CRITIQUE) and Phase 8 (PACKAGE). Use the Firecrawl/Exa connectors when enabled, falling back to the built-in web search and web fetch tools, as set out in the skill's Tooling contract. Do not ship any claim that is unverified or uncited.
