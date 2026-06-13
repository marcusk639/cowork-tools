# Methodology — search, sourcing, and evidence rules (Firecrawl / Exa)

Read this before Phase 2 (SEARCH) and Phase 3 (FETCH & EXTRACT). It governs _how_
searching and evidence capture work over the Firecrawl/Exa MCP connectors.

## Search discipline

- **Multiple queries per angle.** Each of the 5 angles gets several distinct
  phrasings: a broad framing, a specific-term query, a contrarian/"criticism of X"
  query, and a recency query (include the current year). One query per angle is
  insufficient.
- **Use the right tool for the angle:**
  - **Firecrawl `firecrawl_search`** — keyword-style queries; supports
    `sources: [{type: "news"}]` for recency angles and `includeDomains` /
    `excludeDomains` to bias toward authoritative sources or strip content farms.
  - **Exa `web_search_exa`** — semantic; phrase the query as a _description of the
    ideal page_, not bare keywords ("benchmark comparing X and Y throughput in
    2026", not "X vs Y"). Use `category:company` / `category:people` when relevant.
- **Stop condition per angle:** stop when two consecutive new queries surface no
  novel, independent sources (diminishing returns), or the ~15-source pool is full.

## Fetch-before-cite

- A search snippet is a lead, not evidence. **Fetch the page and quote the actual
  text** before recording a claim.
  - **Firecrawl:** `firecrawl_scrape(url, formats: ["markdown"], onlyMainContent: true)`;
    add `parsers: ["pdf"]` for PDFs. One retry with `proxy: "stealth"` is allowed for
    a soft block — do not loop on it.
  - **Exa:** `web_fetch_exa(url)`.
- If a high-value source is unreachable (paywall, hard block, fetch error), record
  it with `quality: unreliable` and treat its claims as `unverified` — never
  paraphrase from memory or from the snippet.
- Capture **verbatim quotes ≤40 words** with the source URL as locator. Long
  paraphrase without a quote is not citable evidence.

## Source quality tiers

Tag every fetched source:

| Tier         | Examples                                                                         | Weight          |
| ------------ | -------------------------------------------------------------------------------- | --------------- |
| `primary`    | Official docs, filings, standards bodies, the thing itself, peer-reviewed papers | Strongest       |
| `secondary`  | Reputable news, established analysts, vendor docs about their own product        | Good            |
| `blog`       | Personal/company blogs, Medium, dev.to                                           | Supporting only |
| `forum`      | Reddit, HN, Q&A threads, GitHub Discussions                                      | Corroboration   |
| `unreliable` | SEO content farms, anonymous aggregators, contradicts primaries, unreachable     | Exclude or flag |

Rules:

- A **single `blog`/`forum` source cannot establish a load-bearing claim** — require
  a `primary`/`secondary` corroborator or mark `confidence: low`.
- "Independent sources" = different ownership/origin. Three outlets syndicating one
  wire story count as **one**.
- **Prefer the official reference for exact specifics** (config keys, API
  signatures, version numbers, prices). Treat forum/Q&A answers as _corroboration
  only_, never the sole basis for an exact value — this guards against snippet
  overreach where a snippet implies a page documents something it does not.

## Atomic claims

- Decompose findings into **atomic claims** (one assertion each). "X is faster and
  cheaper" is two claims.
- Each claim carries its supporting verbatim quote, source URL, source-quality tier,
  and an importance rating (`central` / `supporting` / `tangential`).
- Claims start unverified and only enter the report after surviving the Phase 4
  adversarial verification (see `quality-gates.md`).
