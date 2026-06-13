---
description: Run the Firecrawl/Exa-powered deep-research pipeline
argument-hint: [topic]
---

Invoke the **deep-research-mcp** skill (`skills/deep-research-mcp/SKILL.md`) on the following request:

$ARGUMENTS

Run the full 5-phase pipeline (Scope → Search → Fetch & extract → Verify → Synthesize) using the enabled Firecrawl and/or Exa MCP connectors — not the built-in web tools. If neither connector is enabled, stop and tell the user to enable one (point them to `reference/SETUP.md`), or suggest the built-in `deep-research` skill instead. Read `reference/methodology.md` before Phase 2–3 and `reference/quality-gates.md` before Phase 4–5. Apply the 2-of-3 adversarial kill protocol in Phase 4; no claim enters the report unverified or uncited. If the request is underspecified (missing budget, region, timeframe, or use-case that would change the answer), ask 2–3 clarifying questions before searching.
