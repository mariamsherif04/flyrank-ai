# Capstone Report — Refresh / Content Opportunity Scoring
- **Author:** Mariam Sherif
- **Lane:** Refresh / Content Opportunity Scoring — score pages that are growing, declining, recovering, or worth review; output a ranked action engine with reason codes
- **Repo:** https://github.com/mariamsherif04/flyrank-ai
- **Date:** 30 Aug 2026

## 0. Abstract
Content teams tracking hundreds of pages per client need a way to decide which page to review first when performance slips. This work uses FlyRank's pseudonymized ML-internship dataset — 30,000 content records across 32 clients — to build and honestly validate a per-page decline-risk ranking. A transparent staleness-based rule is compared against Logistic Regression and a Random Forest, evaluated identically on clients held out entirely from training. The baseline rule reached a precision@50 of 0.62 against a 0.52 base rate; the Random Forest reached 0.68, and permutation importance showed the rule's core assumption — that staleness drives decline — does not hold once tested honestly, with visibility signals mattering far more. The result is a ranked, reason-coded review queue an editor can act on today, alongside an honest account of where this approach's confidence ends.

## 1. Problem framing
The unit of analysis is a single content page. The output is a per-page risk score, paired with a reason code explaining *why* it scored that way, that feeds a ranked action engine; the action a human takes is opening the highest-scored pages first and deciding whether to refresh, leave, or double down on them.

Content that ranks well in search quietly decays over time — rankings slip, clicks drop, and most teams notice too late. But not every page moving is declining: some are growing, some are recovering, and some just warrant a look. For a team tracking hundreds or thousands of pages per client, the practical question is never "is anything happening?" but "which pages should a human look at first, and why?" That's a ranking problem, not a simple yes/no classification: an editor with limited time needs a short, trustworthy, explained list — not a flood of undifferentiated flags.

Getting this wrong has an asymmetric cost: a false positive burns limited editor time reviewing a page that was never actually at risk, while a false negative lets a genuinely declining page sit unreviewed until the drop is harder to reverse. Ranking with reason codes (not just flagging) matters because it lets a team spend that limited time on the pages most likely to be worth it, understand at a glance why the engine thinks so, and act accordingly — rather than treating every flagged page as equally urgent or opaque. This is also used as an opportunity to test whether a common assumption — "the older and more neglected a page is, the more likely it's declining" — actually survives contact with validated data.

## 2. Data safety
**Dataset:** FlyRank's pseudonymized ML-internship starter release, `content_refresh_anonymized.csv` — 30,000 rows, one per content item, spanning 32 pseudonymized clients, with trailing-90-day search performance metrics per item.

**Features used:** content age, days since last update, 90-day impressions, average search position, click-through rate, word count.

**Excluded fields:** `trend_direction` and `trend_pct` — these define the label itself, so using them as features would be circular (leakage).

**Context-only fields:** `content_id` and `client_id` — used solely for grouping and train/test splitting, never as model inputs.

No client names, domains, or raw queries appear anywhere in this dataset or in `work/` — all identifiers are pseudonymized hashes.

## 3. Baseline
A transparent, hand-written rule: a page is flagged if it hasn't been updated in 180+ days **and** still receives 500+ impressions in the last 90 days — reasoning that a page which used to matter but has gone stale is worth a look.

This is a fair comparison because it's evaluated on the exact same held-out clients and the exact same metric (precision@50) as the model. It reached 0.62 against a 0.52 base rate.

## 4. Model / analysis
**Method:** Logistic Regression, then a Random Forest — chosen because the underlying task is ranking a continuous risk score (not producing a fixed label), which then drives the reason-coded action engine. Simplicity first, complexity only where it earned its place.

**Feature list:** content age, days since last update, 90-day impressions, average search position, click-through rate, word count. Left out on purpose: `trend_direction` and `trend_pct` (label-derived, would leak), and `content_id`/`client_id` (context-only, not predictive features).

**Target/proxy definition:** a page is labeled "declining" if its current trend direction is downward — an observed, current-window signal, not a confirmed future outcome.

## 5. Evaluation
**Split:** client-grouped train/test split (25% of clients held out entirely), not row-random. A random split would let the same client appear in both sides, letting the model learn client-specific quirks instead of a signal that generalizes to a client it has never seen — the honest version of this question.

**Metrics — model vs. baseline, same split:**

| Method | Precision@50 | Base rate |
|---|---|---|
| Baseline rule | 0.62 | 0.52 |
| Logistic Regression | 0.66 | 0.52 |
|
