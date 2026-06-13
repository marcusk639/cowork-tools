# Quality gates — verification loop and release checklist

Read this before Phase 4 (VERIFY) and Phase 5 (SYNTHESIZE). These gates are the
part that separates this from a generic "search and summarize."

## Phase 4 — adversarial verification protocol (the 2-of-3 kill)

The Claude Code original ran three _independent_ verifier agents per claim. In
Cowork you (one agent) reproduce this as **three separate skeptical passes** over
each candidate claim, each from a distinct persona. Each persona answers one
question: _"On the evidence cited, is this claim refuted?"_

1. **Adversarial Reviewer** — Is the claim broader than the evidence supports? Are
   scope/assumptions hiding a weakness? Is the methodology behind the source sound?
2. **Domain Skeptic** — Is this claim surprising, convenient, or novel? Does it rest
   on a single source? **Run a fresh `firecrawl_search` / `web_search_exa` for
   contradicting evidence** — what would a knowledgeable critic cite against it?
3. **Source-Quality Auditor** — Does the quote actually support the claim, or is it
   an overreach/misread? Any circular sourcing (blog citing the blog)? Is a
   `blog`/`forum`/`unreliable` tier doing `primary` work? Is it outdated for a
   fast-moving field? Is it marketing / a press release / a cherry-picked benchmark?

### Verdict rule

- A claim is **killed (refuted)** if **≥2 of 3 passes refute it.**
- A claim **survives (confirmed)** only if not refuted by a majority _and_ it meets
  the source-quality bar for its strength.
- Anything else is **unverified** → goes to Caveats / Open Questions, never stated
  as fact in the body.
- **Bias toward refutation when uncertain.** An omitted unverified claim is cheaper
  than a wrong claim shipped.

Record each pass's call and the resulting vote tally (e.g. `2-1`) so the decision is
auditable and can be reported in the Refuted-Claims section.

## Iteration budget

- After a kill, you may **delta-search** once to try to rescue a `central` claim
  with better sourcing, then re-judge. Skip rescue for `tangential` claims.
- **Maximum 2 rounds.** After round 2, ship with surviving claims only and disclose
  what couldn't be verified.
- Cost-aware shortcut: for light topics, drop to 3 angles and skip the Phase 4
  re-search step for `tangential` claims (see the Cost note in `SKILL.md`).

## Phase 5 — release checklist (hard gates)

Do not deliver until every box is true:

- [ ] **Source floor:** at least ~8 independent, fetched sources after dedup (more
      for broad topics). Snippets do not count.
- [ ] **Citation coverage:** every factual sentence in the body carries an inline
      source link.
- [ ] **Citation integrity:** every citation resolves to a real verbatim quote from
      a _fetched_ page (Firecrawl/Exa), not a search snippet and not memory.
- [ ] **No zombies:** zero refuted claims stated as fact in the body.
- [ ] **Conflicts disclosed:** every flagged conflict and every `unverified` claim
      appears in **Caveats** / **Open Questions**.
- [ ] **Confidence labeled** on each finding (high / medium / low).
- [ ] **Bibliography complete:** every source listed with URL + quality tier.
- [ ] **Refuted-Claims section present** when any claim was killed (transparency).
- [ ] **No placeholders / no fabricated numbers.**

## Honest-failure rule

If the topic can't clear the gates (sources unreachable, evidence too thin, every
claim refuted), say so plainly and deliver a partial brief stating what is known,
what isn't, and why — rather than padding to look complete. A small verified answer
beats a large unverifiable one.
