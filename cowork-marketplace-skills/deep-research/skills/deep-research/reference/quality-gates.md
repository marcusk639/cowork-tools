# Quality gates — verification loop and release checklist

Read this before Phase 6 (CRITIQUE) and Phase 8 (PACKAGE). These gates are the part that separates this from a generic "search and summarize."

## Phase 6 — adversarial verification protocol

For each contested or load-bearing claim, evaluate it independently from **three personas**. Each persona answers one question: _"On the evidence cited, is this claim refuted?"_

1. **Adversarial Reviewer** — Is the methodology sound? Are scope/assumptions hiding a weakness? Is the claim broader than the evidence supports?
2. **Domain Skeptic** — Is this claim surprising, convenient, or novel? Does it rest on a single source? What would a knowledgeable critic say?
3. **Source-Quality Auditor** — Does each `[Sn]` actually contain a quote supporting the sentence? Any circular sourcing (blog citing the blog)? Any `blog`/`unreliable` tier doing `primary` work?

### Verdict rule (the 2-of-3 kill)

- A claim is **killed (`verdict: refuted`)** if a **majority (≥2 of 3) personas refute it.**
- A claim **survives (`verdict: confirmed`)** only if not refuted by a majority _and_ it meets the independent-source floor.
- Anything else is **`unverified`** → goes to Limitations, never stated as fact in the body.
- Bias toward refutation when uncertain: an unverified claim omitted is cheaper than a wrong claim shipped.

Record each persona's call in the claim's notes so the decision is auditable.

## Iteration budget

- After a kill, **delta-retrieve** (Phase 7) to try to rescue the claim with better sourcing, then re-judge.
- **Maximum 2 rounds.** After round 2, ship with surviving claims only and disclose what couldn't be verified.

## Phase 8 — release checklist (hard gates)

Do not deliver until every box is true:

- [ ] **Source floor met** after dedup: `quick`≥5, `standard`≥8, `deep`≥10, `ultradeep`≥15.
- [ ] **Citation coverage:** every factual sentence in the body carries an inline `[Sn]`.
- [ ] **Citation integrity:** every `[Sn]` resolves to a real `evidence.jsonl` quote from a _fetched_ page (not a snippet, not memory).
- [ ] **No survivors that should be dead:** zero claims with `verdict: refuted` stated as fact in the body.
- [ ] **Conflicts disclosed:** every flagged conflict and every `unverified` claim appears in **Limitations & Open Questions**.
- [ ] **Confidence labeled** on headline conclusions (high/medium/low).
- [ ] **Bibliography complete:** every `Sn` listed with URL + quality tier.
- [ ] **No placeholders / no fabricated numbers.**

## Honest-failure rule

If the topic can't clear the gates (sources unreachable, evidence too thin), say so plainly and deliver a partial brief that states what is known, what isn't, and why — rather than padding to look complete. A small verified answer beats a large unverifiable one.
