---
name: deep-research
description: Deep, multi-source, fact-checked research using the Firecrawl and/or Exa MCP connectors. Produces a cited report with evidence, source attribution, and confidence ratings. Use when the user wants thorough research on any topic. Decomposes the question into angles, searches the web via Firecrawl/Exa, deep-reads (scrapes) key sources, adversarially verifies each claim before trusting it, and synthesizes a cited report. If the question is underspecified, ask 2-3 clarifying questions first.
---

# Deep Research (Firecrawl / Exa MCP)

Produce a thorough, cited, fact-checked research report driven by the **Firecrawl**
and/or **Exa** MCP connectors. This is a single-agent port of a Claude Code
multi-agent workflow — the original spawned parallel searcher and verifier
sub-agents; here you (one agent) run the same five phases sequentially. The
discipline — broad angles, source-quality grading, and **adversarial verification
before any claim enters the report** — is the point. Do not skip verification.

The original orchestration script is in
`reference/original-claude-code-workflow.js` for provenance.

## MCP connector requirements

This skill expects at least one of these connectors enabled in Cowork
(**Settings → Connectors**). Both together give the best coverage. See
`reference/SETUP.md` for how to add them.

- **Firecrawl** — `firecrawl_search` (search + optional content), `firecrawl_scrape`
  (fetch one page as clean markdown), `firecrawl_crawl`/`firecrawl_map` (site-wide).
- **Exa** — `web_search_exa` (semantic search), `web_fetch_exa` (fetch a URL's
  clean content).

> Tool names may appear namespaced depending on how the connector is registered
> (e.g. an `exa`/`firecrawl` prefix). Use whichever search and fetch tools the
> enabled connector exposes; the names above are the logical operations. If
> neither connector is available, stop and tell the user to enable one.

## Before You Start: Scope Check

If the question is vague or missing constraints that change the answer (budget,
region, timeframe, use-case), ask **2-3 clarifying questions**. If the user says
"just research it," proceed with reasonable defaults and state your assumptions.

---

## Phase 1 — Scope: decompose into search angles

Break the question into **5 complementary search angles** that cover it from
different directions; avoid redundancy. Pick angles suited to the domain:

- General: broad/primary · academic/technical · recent news · contrarian/skeptical · practitioner/implementation
- Medical: anatomy · common causes · serious differentials · authoritative refs · red flags
- Tech: state-of-art · benchmarks · limitations · industry adoption · cost/tradeoffs

For each angle write a concrete, high-signal search query. Show the user the angle
list before proceeding.

## Phase 2 — Search: gather candidate sources

For **each angle**, run a search. Prefer:

- **Firecrawl:** `firecrawl_search(query, limit: 6)`. For news-weighted angles set
  `sources: [{type: "news"}]`. Use `includeDomains`/`excludeDomains` to bias toward
  authoritative sources or filter spam.
- **Exa:** `web_search_exa(query, numResults: 6)`. Phrase the query as a description
  of the ideal page, not bare keywords (Exa is semantic). Use `category:company` or
  `category:people` when relevant.

Collect 4-6 results per angle, ranked by relevance **to the original question**.
Then **deduplicate across angles**: normalize URLs (strip `www.`, trailing slashes,
lowercase host+path), keep the first occurrence. Cap the unique pool at **~15
sources**; drop lowest-relevance extras over budget and note how many you dropped.

## Phase 3 — Fetch & extract: deep-read the sources

Do **not** rely on search snippets. For each unique source, fetch the full page:

- **Firecrawl:** `firecrawl_scrape(url, formats: ["markdown"], onlyMainContent: true)`.
  For PDFs add `parsers: ["pdf"]`.
- **Exa:** `web_fetch_exa(url)`.

From each page extract:

1. **Source quality:** `primary` (research/institution/official) · `secondary`
   (reputable reporting) · `blog` · `forum` · `unreliable`.
2. **2-5 falsifiable claims** bearing on the question. Each must be concrete and
   checkable, carry a **direct supporting quote**, and be rated
   `central`/`supporting`/`tangential`.
3. **Publish date** if available.

If a page is paywalled, irrelevant, or fails to load, record zero claims and mark
it `unreliable`. Do not invent content. (Firecrawl `firecrawl_scrape` with
`proxy: "stealth"` can sometimes get past soft blocks — try once, don't loop.)

Pool all claims. Rank by importance (`central` first), then source quality
(`primary` first). Keep the **top ~25 claims** for verification.

## Phase 4 — Verify: adversarial 3-pass per claim (THE CRITICAL PHASE)

For each candidate claim, run **three independent skeptical passes**. In each pass,
actively try to **refute** it. A claim is **killed if ≥2 of 3 passes refute it**.

Each pass works the checklist:

1. Is the claim actually supported by its quote, or an overreach/misread?
2. **Run a fresh `firecrawl_search` / `web_search_exa` for contradicting evidence** —
   does any credible source dispute or heavily qualify it?
3. Is the source quality sufficient for the claim's strength? Extraordinary claims
   need primary sources.
4. Is it outdated? Old claims about fast-moving fields are suspect — check dates.
5. Is it marketing / a press release / a cherry-picked benchmark / forum speculation?

**Refute** if: unsupported by the quote, contradicted, low-quality source for a
strong claim, outdated, or marketing fluff. **Pass** only if well-supported,
current, and source quality matches claim strength. **When uncertain, refute** —
the bar to survive is deliberately high.

Record each claim's vote tally (e.g. `2-1`) and a one-line evidence note. A claim
survives only if you completed the passes and fewer than 2 refuted it.

If **every** claim is refuted, stop and report the research as inconclusive.

## Phase 5 — Synthesize: write the cited report

From **surviving claims only**:

1. Merge claims that say the same thing; combine their sources.
2. Group related claims into coherent findings, each addressing the question.
3. Assign confidence per finding: **high** (multiple primary sources, unanimous
   passes) · **medium** (secondary or split votes) · **low** (single source / blog).
4. Write a 3-5 sentence executive summary answering the question.
5. State caveats: what's uncertain, weak sources, time-sensitivity.
6. List 2-4 open questions that emerged but weren't answered.

### Report format

```markdown
# [Topic]: Research Report

_Generated: [date] · Sources fetched: [N] · Claims verified: [M] · Confidence: [High/Medium/Low]_

## Executive Summary

[3-5 sentences answering the question]

## Findings

### [Finding 1] — confidence: [high/medium/low]

[Synthesis] ([Source](url)), ([Source](url))

> verification vote: [e.g. 3-0]

## Caveats

[What's uncertain, weak sources, time-sensitivity]

## Open Questions

- ...

## Refuted Claims (transparency)

- "[claim]" — killed [vote], [source]

## Sources

1. [Title](url) — [quality] — [one-line summary]

## Methodology

[N] angles searched via Firecrawl/Exa, [N] sources scraped, [M] claims extracted,
[K] verified, [C] confirmed / [X] refuted via 3-pass adversarial verification.
```

## Quality Rules (non-negotiable)

1. **Every claim needs a source.** No unsourced assertions.
2. **Cross-reference.** A single-source claim is flagged, not stated as fact.
3. **Recency matters.** Prefer sources from the last ~12 months for live topics.
4. **Acknowledge gaps.** If an angle yielded nothing solid, say so.
5. **No hallucination.** "Insufficient data found" beats a confident guess.
6. **Separate fact from inference.** Label estimates, projections, opinions.

## Cost note

Firecrawl and Exa bill per call. A full run is ~5 searches + ~15 scrapes + up to
~25 verification searches. For light topics, reduce angles to 3 and skip Phase 4's
re-search step for `tangential` claims.

## Examples

- "Research the current state of nuclear fusion energy"
- "Deep dive into Rust vs Go for backend services"
- "Investigate the competitive landscape for AI code editors"
