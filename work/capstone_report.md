# From Rules to Ranking: A Client-Honest Model for Predicting Content Decline

**Refresh & Content Opportunity Scoring on Real SEO Performance Data**

- **Author:** Hussain
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Hussainhhgh/flyrank-ml-internship
- **Date:** [today's date]

---

## Abstract

Content that ranks well today quietly decays tomorrow — and most teams only notice after the traffic is already gone. This project asks a narrower, answerable version of that problem: given six signals knowable at decision time (content age, average position, CTR, impressions, engagement rate, and search volume), can a model rank pages by decline risk well enough to beat a transparent hand-written rule? Using a client-grouped holdout split — so no client's pages leak between training and testing — a Random Forest classifier reached a Precision@50 of 0.66, against 0.48 for the baseline rule and a 56.4% base rate for "declining" pages. The result is a ranked, reason-coded action playbook built for human review, not autonomous action — designed to help a content team spend a limited number of monthly refresh slots on the pages most likely to need them.

---

## 1. Problem Framing

**The decision this supports:** which pages should a content or SEO team prioritize for review this month, when refresh capacity is limited and most pages cannot be manually checked.

**Unit of analysis:** one webpage per row.

**Output:** a decline-risk score (0–1), a reason code naming the dominant contributing signal, and an action label — `refresh_priority`, `monitor`, or `no_action`.

**The human action:** a team lead reviews the top of the ranked list and decides, page by page, whether to refresh, monitor, or leave it alone.

**Cost of getting it wrong:** missing a genuinely declining page (a false negative) means lost visibility compounds silently until it's expensive to reverse. Flagging a healthy page (a false positive) wastes a scarce review slot. Because refresh *capacity* — not page-checking capacity — is the actual bottleneck, this system is built to rank, not to classify with a hard cutoff. A ranked list lets a reviewer see relative priority even when they can only act on a handful of pages.

**Why this needed a model, not just a rule:** the first attempt was a hand-written rule combining staleness and position tier. It reached Precision@50 = 0.48 — barely better than chance given the 56.4% base rate. That gap was the signal that the real relationship between decline and its drivers isn't a simple threshold; it's closer to an interaction between several weak signals than a single strong one.

---

## 2. Data

**Scope of the full release:** the FlyRank ML Internship warehouse spans approximately 79 million rows (`fact_content_daily_performance`: ~78.8M rows, 519,606 content items, 104 clients, January 2025–June 2026).

**What this paper actually used:** `data/raw/content_refresh_anonymized.csv` — a 30,000-row, 44-column anonymized subset of that same production data, spanning 32 clients with trailing-90-day performance metrics. This smaller, safe sample was deliberately chosen to allow rigorous client-holdout validation and a full leakage audit within the scope of this project; the methodology is designed to extend to the full warehouse as future work, not as a shortcut around it.

**Columns deliberately excluded:**
- `trend_direction`, `trend_pct` — these *define* the target label. Using either as a feature would be direct label leakage, since the model would effectively be shown the answer.
- `content_id`, `client_id` — used only to group the train/test split by client, never as predictive features.

**Leakage audit:** every remaining feature — `content_age_days`, `avg_position`, `ctr`, `impressions_90d`, `engagement_rate`, `search_volume` — was checked for availability *before* any future outcome is known. A correlation pass against the target confirmed none behaves like a disguised copy of the label; the strongest single correlation was `content_age_days` at only −0.179, far below the range (±0.9+) that would suggest hidden leakage.

**Privacy:** no client names, domains, URLs, or private search queries appear anywhere in this repository's `work/` directory. Every reference in this paper uses only anonymized `content_id` / `client_id` pseudonyms.

---

## 3. Baseline

Before any modeling, a transparent rule was built to set a fair bar: flag a page `refresh_priority` if it fell into a stale freshness tier (31–90 or 91–180 days since last update) **and** a mid-pack position tier (`striking`, `page_1`, or `page_3_5`).

That "mid-pack" choice wasn't arbitrary — it came from an earlier signal audit that reversed an initial assumption. The expectation was that decline risk would climb steadily as ranking position worsened. Instead, the data showed the opposite: top-3 and deep-position pages had the *lowest* decline rates, while pages sitting in the middle of the pack were the most at risk. That negative result directly shaped the baseline rule.

This rule is simple, fully auditable, and represents the kind of manual triage a content team could run today with no model at all — which is exactly what makes it a fair comparison point.

**Baseline result:** Precision@50 = 0.48.

---

## 4. Methodology

**Model:** a Random Forest classifier (200 trees, max depth 6), benchmarked against Logistic Regression. Random Forest was chosen specifically because it can capture *interactions* between features — a page that is simultaneously stale, mid-position, and low-CTR behaves differently than the sum of those three signals taken separately — without requiring the hand-picked bucket cutoffs the baseline rule depended on.

**Features (6):** `content_age_days`, `avg_position`, `ctr`, `impressions_90d`, `engagement_rate`, `search_volume` — each confirmed knowable at decision time.

**Target:** a proxy label. The true goal — "will this page's value keep decaying" — isn't directly measurable at prediction time, so `trend_direction == "down"` (a 30-day-vs-previous-30-day comparison) stands in as the measurable signal.

**Base rate:** 56.4% of rows carry the declining label — reported here deliberately, so every Precision@50 figure below can be read against chance, not in isolation.

---

## 5. Results

**Split design:** grouped by `client_id`, not by row — 21 clients for training, 10 fully held out for testing, with zero client overlap. This matters because a row-level split risks letting the model learn a *specific client's* quirks (industry, writing style, reporting cadence) rather than a pattern that generalizes to clients it has never seen.

**Making the leakage risk visible:** the same Random Forest, re-run under a naive row-level split with no client grouping, scored Precision@50 = 0.88 — a full 0.22 higher than the honest 0.66. That 22-point gap isn't noise; it's the exact size of the illusion a careless split can create. Every number in this paper's headline comparison uses the honest, client-grouped split.

| Method | Precision@50 |
|---|---|
| Base rate (majority class) | 0.564 |
| Baseline rule | 0.48 |
| Logistic Regression | 0.54 |
| Random Forest — naive split *(inflated, shown for contrast)* | 0.88 |
| **Random Forest — client-grouped (honest)** | **0.66** |

**Where the model and the rule disagree:** the two methods shared *zero* pages in their respective top-50 lists. Investigating a sample of pages the model flagged that the rule missed showed a consistent pattern — mid-range content age (138–148 days) sitting right at or just past the rule's tier boundary, combined with near-zero CTR. The rule's binary buckets were discarding real information: a page one day past a boundary was being treated identically to a page one day before it, while the model could weigh the continuous value directly.

---

## 6. Interpretation

**What drives the model's decisions:** `content_age_days` (33%) and `impressions_90d` (29%) account for over 60% of the Random Forest's importance, with `avg_position` contributing 18%. `search_volume`, `ctr`, and `engagement_rate` each contribute under 10%.

**The negative result worth keeping:** the initial hypothesis — that decline risk rises steadily as position worsens — did not hold. Risk instead peaked in the *middle* of the ranking pack, not at the extremes. A well-understood "no" is still a finding, and this one directly reshaped how the baseline rule was built.

**A known limitation:** `avg_position = 0` encodes "no data" in this dataset, not literal first place (2.3% of the test set carries this value). None of this run's top model-flagged pages were affected, but it remains a caveat worth monitoring in future runs.

---

## 7. Recommendations

1. **Review `refresh_priority` pages first** — these carry the model's highest decline-risk scores.
2. **Weight combined-signal reason codes above single-signal ones** — e.g. `WEAK_POSITION_LOW_CTR` proved more reliable in error analysis than either signal alone.
3. **Revisit `monitor`-flagged pages next cycle** rather than acting immediately.
4. **Deprioritize `no_action` pages** for this cycle only — not permanently.

**Human review is mandatory, not optional.** Every `refresh_priority` flag should be checked for: whether low CTR reflects a real content problem or a data artifact, whether the page is tied to seasonal or campaign content where decline is expected, and whether the reason codes cohere with what a reviewer already knows about the page.

**What must never be automated:** publishing content changes, deprioritizing or removing pages, or adjusting budget — all without a human in the loop. The risk score is a ranking signal for prioritization, not a calibrated probability of real-world traffic loss.

**When to retrain:** if Precision@50 on fresh data drops more than ~10 points below the validated 0.66, if the client mix shifts meaningfully, if an untrained content type appears, or if the share of `avg_position = 0` rows changes sharply — any of these should trigger a re-evaluation before the playbook is trusted again.

**Confidence, stated plainly:** this is decision-support, not a verdict. The 0.66 figure reflects performance on this specific 30-client sample and should not be read as a guarantee on unseen clients, future time windows, or different content types.

---

## 8. Reproducibility

```bash
git clone https://github.com/Hussainhhgh/flyrank-ml-internship.git
cd flyrank-ml-internship
pip install -r requirements.txt
```

Run in order:
`work/notebooks/w02_ml_task_framing.ipynb` → `w03_data_contract.ipynb` → `w04_baseline_score.ipynb` → `w05_model.ipynb` → `w06_validation_audit.ipynb` → `w07_action_playbook.ipynb` → `capstone.ipynb`

**Random seed:** `random_state=42` throughout, for every split and model.
**Sealed evaluation:** the client-grouped test split was evaluated once per notebook run; the resulting metrics are committed at `work/outputs/w07_playbook_metrics.json` as the receipt these numbers trace back to.

---

## 9. Acknowledgments & Data Credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

---

*Every claim in this report uses observed, measured, directional, or decision-support language. No causal claims are made, and Precision@50 is reported alongside the base rate throughout so the model's lift is never read in isolation. No client-identifying information appears anywhere in this repository.*
