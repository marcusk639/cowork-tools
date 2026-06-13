# Methodology — search, sourcing, and evidence rules

Read this before Phase 3 (RETRIEVE). It governs _how_ searching and evidence capture work on Cowork's built-in tools.

## Search discipline (built-in web search)

- **Multiple queries per angle.** Each of the 4–6 angles gets several distinct phrasings: a broad framing, a specific-term query, a contrarian/"criticism of X" query, and a recency query (e.g. include the current year). One query per angle is insufficient.
- **Parallelize.** Where Cowork lets you dispatch concurrent sub-agent searches, run angles in parallel. Otherwise interleave so no single angle dominates the budget.
- **Stop condition per angle:** stop when two consecutive new queries surface no novel, independent sources (diminishing returns), or the mode's source floor is met with margin.

## Fetch-before-cite (built-in web fetch)

- A search snippet is a lead, not evidence. **Web-fetch the page** and quote the actual text before recording evidence.
- Cowork web fetch only reaches search-result URLs and URLs the user shared. If a high-value source is unreachable, log it in `sources.jsonl` with `"quality":"unreachable"` and treat its claims as `unverified` — never paraphrase from memory.
- Capture **verbatim quotes ≤40 words** with a locator. Long paraphrase without a quote is not citable evidence.

## Source quality tiers

Tag every source in `sources.jsonl`:

| Tier         | Examples                                                                         | Weight          |
| ------------ | -------------------------------------------------------------------------------- | --------------- |
| `primary`    | Official docs, filings, standards bodies, the thing itself, peer-reviewed papers | Strongest       |
| `secondary`  | Reputable news, established analysts, vendor docs about their own product        | Good            |
| `blog`       | Personal/company blogs, Medium, dev.to, forum posts                              | Supporting only |
| `unreliable` | SEO content farms, anonymous aggregators, contradicts primaries                  | Exclude or flag |

Rules:

- A **single `blog` source cannot establish a load-bearing claim** — require a `primary`/`secondary` corroborator or mark `confidence: low`.
- "Independent sources" = different ownership/origin. Three outlets syndicating one wire story count as **one**.
- **Prefer the official reference for exact specifics** (config keys, API signatures, version numbers). For any claim of the form "set X to do Y," confirm the precise key/path against the project's own docs or source, and treat forum threads, GitHub Discussions, and Q&A answers as _corroboration only_ — never as the sole basis for an exact configuration value. This guards against search-snippet overreach where a snippet implies a page documents something it does not.

## Atomic claims

- Decompose findings into **atomic claims** (one assertion each) in `claims.jsonl`. "X is faster and cheaper" is two claims.
- Each claim links to its supporting `evidence.jsonl` IDs and a count of _independent_ sources.
- Claims start `verdict: unverified` and only become `confirmed` after triangulation (Phase 4) or surviving red-team (Phase 6).
