# Cowork Agents — Claude Cowork plugin

Two subagents Claude can hand work to inside a Cowork task. Each runs in its own context and returns only its result, so the research or checking doesn't fill up your main conversation.

| Agent                              | Model  | Job                                              | Returns                                        |
| ---------------------------------- | ------ | ------------------------------------------------ | ---------------------------------------------- |
| `cowork-agents:research-analyst`   | Sonnet | Find things out across several sources           | Findings with source links and confidence notes |
| `cowork-agents:verifier`           | Opus   | Check one specific claim before you act on it    | A verdict with the evidence behind it          |

Claude delegates to an agent when your request matches its description. You can also ask for one by name: _"use the verifier to check this."_

## research-analyst

**Use it for** a question that needs several sources and a combined answer: market and industry trends, competitor scans, comparing options, or background before a decision.

**What it does:** clarifies the question, searches and reads sources, weighs how credible each one is, cross-checks them, and reports what it found with links. It says where evidence is thin or sources disagree, and it never reports counts or confidence figures it didn't measure.

**Example:**

> "Research how mid-size clinics are adopting AI scribes in 2026: the vendors, pricing models, and the main objections."

**Compared with the [`deep-research`](../deep-research) skill:** `deep-research` runs a full 8-phase pipeline with evidence files and a red-team review, and produces a long cited report in your main conversation. `research-analyst` is lighter. Use it when research is one part of a bigger task, or to research a few questions side by side.

## verifier

**Use it for** one specific claim where being wrong has a real cost: a statistic going into a deck, a research finding you'll repeat, a recommendation you're about to follow, or a conclusion another agent reached.

**What it does:** starts from the assumption that the claim might be wrong. It restates the claim, re-derives the answer from primary sources rather than trusting the earlier reasoning, opens cited sources itself, and looks for what would make the claim false.

**What it returns:**

```
Claim under test: [one sentence]
Verdict: CONFIRMED | REFUTED | PLAUSIBLE (unverified) | MISROUTED
Evidence: [what it checked, with links]
Failure mode considered: [what would have made the claim false, and whether it applies]
Confidence: [low | medium | high] — [why]
```

- **CONFIRMED** — true.
- **REFUTED** — false.
- **PLAUSIBLE** — not disproven, but not fully checked. It says what it didn't get to.
- **MISROUTED** — there was no specific claim to test.

**Example:**

> "Verify this before I send it: 'Over 40% of US physicians now use ambient AI documentation.'"

**Not for** general proofreading or "look this over." It needs a claim it can test.

## Using them together

Let `research-analyst` gather, then have `verifier` check the two or three findings your decision rests on:

> "Research X, then have the verifier check the top three findings before you write the summary."

## Usage examples

You can name an agent or just describe the job; naming it is more reliable.

**Research a market question**

> "Use the research-analyst to find what mid-size clinics pay for AI scribe tools in 2026, with sources."

**Research several questions in parallel**

> "Have research-analyst agents look into these three vendors separately — pricing, integrations, and customer complaints for each — then compare them in a table."

**Check a statistic before it goes out**

> "Use the verifier on this line from my deck: 'Ambient AI documentation cuts charting time by 50%.'"

Typical result: `PLAUSIBLE (unverified)` or `REFUTED`, with the studies it checked, the figures they actually report and the conditions behind them.

**Check a recommendation before acting on it**

> "The research says we should switch to annual billing to reduce churn. Have the verifier check whether the evidence actually supports that for B2B SaaS under $50/month."

**Check another agent's work**

> "Run deep research on state rules for telehealth prescribing, then have the verifier check the three claims the recommendation depends on."

**Research, then verify**

> "Research the current FDA position on AI clinical decision support, then verify the two findings that matter most for our product."

## Tools and connectors

Neither agent has a `tools:` list, so each inherits whatever the session has: Cowork's built-in web search and fetch, plus the Firecrawl or Exa connectors when they're enabled. No setup needed.

## Install

In Cowork: **Customize → Plugins → `cowork-tools` → `cowork-agents` → Install**. CLI:

```bash
claude plugin install cowork-agents@cowork-tools
```

## Origin

Adapted from the author's Claude Code agents. Changes for Cowork:

- Removed the `tools:` lists so the agents use Cowork's tools and connectors.
- Removed references to Claude Code-only agents and a "context manager" protocol that doesn't exist in Cowork.
- Removed made-up example metrics from `research-analyst` that invited it to report invented numbers.
- Allowed `verifier` to cite source URLs; the original banned links, which blocked citing evidence.

## Layout

```
cowork-agents/
├── .claude-plugin/plugin.json
└── agents/
    ├── research-analyst.md
    └── verifier.md
```
