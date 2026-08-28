

# Capstone Report: Refresh / Content Opportunity Scoring

- **Author:** [Attaf Muneeb]
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/Msdff/FlyRankAiAssignment
- **Date:** [28-8-2026]

## 0. Abstract

> The objective of this project is to check which page the editor should edit first by using age, visibility, and search position. The data are 30,000 pages provided by FlyRank. We made a simple rule-based system and also a Random Forest ML model. We try to test it in a fair way by keeping clients separate. The rule performed better than the ML model, with (precision@50: 0.70 vs 0.62). So it shows that rules can work better than complex models. In the end, we created a ranked list of pages with a reason for each page. This list helps a human editor decide which pages to review first. It does not automatically edit or publish pages.   

## 1. Problem framing


**Decision this supports:** In limited time, which page should be reviewed first by the content editor?

**Unit of analysis:** One content page, belonging to one client (starter CSV); in the warehouse data-contract check, one content page × one client × one day.

**Output:** A ranked priority score per page (the task is Ranking/Scoring , not classification or clustering). A reason code and a suggested action.

**Action a human takes:** A content editor runs down the ranked list from the Handles highest priority pages and manually reviews refreshes those pages.

**Cost of a wrong call:** If a low-priority page is ranked too high, the editor wastes limited time on a page that didn't need attention. If a truly bad page is ranked too high, continues the trend of being unchecked in visibility. There is a real cost in both directions.

**Why ML/data helps:** The simple rule was to flag pages that had not been updated for more than 6 months. But this rule only looked at the page’s age. It did not look at traffic or search position. When I checked the data, I found that being old did not always mean that a page would decline. In a small sample, older pages actually had a lower decline rate. So, we checked more than one signal, such as age, traffic, and search position. The ML model did not perform better than the simple rule, but checking multiple signals helped us find this result honestly.


## 2. Data safety


**Data used:** The small anonymized starter dataset (`data/raw/content_refresh_anonymized.csv`, 30,000 rows) for baseline, modeling, and validation work. A March 2026 slice of the FlyRank warehouse (`fact_content_daily_performance`, ~9.8M rows, via Hugging Face + DuckDB) was used separately to verify the data contract (grain, availability, feature timing).

**Deliberately excluded:** Each of the scores calculated by any pre-built FlyRank product (`health_score`, `priority_score`, `action_type`) ,even though this dataset did not contain these, I noted these would never be used as features, since a model trained on them would just copy an existing decision rather than learn from raw signals.

**Leakage risks considered:** `trend_direction` and `trend_pct` were used only to *build* the label (`is_declining_label`) and to verify signals, they never included as model features. `content_id` and `client_id` were only used for grouping and joining (Do not use train/test split as predictive features. ), never as predictive features. A deliberate leakage demonstration (Week 3) gave a hands-on proof that showed how a correlation went from -0.011 to 0.644, which was derived from a label.This is important.

**Client-identifying details:** Not found in any of `work/`: All IDs arepseudonymous hash codes shipped pre-anonymized by FlyRank.


## 3. Baseline


**The rule:**If the page is not updated in 180+ days, it is a stale page and should be reviewed and still visible (500+ impressions in the last 90 days).


baseline_score = stale × visible × impressions_90d
reason_code = "stale_visible_page"


**Why it's a fair comparison:** It employs only signals that are observed in time of decision, no .The same precision@K metric and the same "no product flags" evaluation are used for future data. Later, the test split was done client-grouped, and this split was used for the model.

**Baseline numbers (client-grouped split, n=7 held-out clients):**

 K    Precision
 20    0.80 
 50    0.70 

Signal verification behind this rule: visibility was CONFIRMED as a real signal (high-visibility pages showed a 59.6% decline rate vs 47.5% for low-visibility, n=16,726 vs n=13,274). Staleness alone was more ambiguous, in a small stale-page bucket (n=174), decline rate was actually lower (47.1%) than fresh pages (54.2%), an OPPOSITE result likely driven by small sample size, so the rule deliberately requires staleness *combined with* visibility rather than staleness alone.


## 4. Model / analysis


**Method:** Random Forest Classifier. Selected as it has the capability of combining multiple, possibly interacts with signals without using a handwritten formula, and returns a probability which fits Directly to a ranking/scoring pipeline.

**Feature list (5 features, all decision-time-available):**
- `days_since_last_update`
- `impressions_90d`
- `avg_position`
- `word_count`
- `search_volume`

**Deliberately left out:** `trend_direction`/`trend_pct` (label-derived, would leak the answer); any FlyRank product score (not in this dataset, and excluded on principle per Section 2).

**Target/proxy, in one sentence:** `is_declining_label` a same-window proxy defined as `trend_direction == "down"`, acknowledged as a starter-level proxy rather than a validated future outcome (a stronger version would use a forward-looking window, e.g., decline over the 
*next* 30 days).


## 5. Evaluation


**Split:** Client-grouped holdout (`GroupShuffleSplit`, 25 train clients / 7 test clients, 
23,837 / 6,163 rows). pages are selected to be able to share client-specific data. a client across train and test would have caused a model to "recognize" a client; mixing across train and test would cause quirks. Instead of making a broad sweeping statement, giving the illusion of a high score.
**Before/after demonstration (Week 6):** To make this concrete, the same model was also evaluated under a naive random row-level split (no client grouping):

 Split type                           Precision@20     Precision@50
 Naive random split                      0.75            0.86 
 Honest client-grouped split             0.65            0.62 

The naive split made the model look meaningfully stronger than it really is, especially at 
K=50 a direct, measured illustration of why validation design matters.

**Model vs. baseline, same split, same metric:**

 K     Baseline  Model 
 20      0.80    0.65 
 50      0.70    0.62 

The baseline performed better than the model for both cuts. Base rate (decline rate overall in this) 
However, neither baseline nor model precision is 54.2% (dataset) and both are much higher. 
Only reflecting the base rate.

**Error analysis:** Feature importance showed the model relies mostly on `impressions_90d` and`avg_position` (~70% combined importance) and barely on `days_since_last_update`, the exact signal the baseline rule was built around, which helps explain the gap. Misclassifications clustered on low-visibility pages (impressions_90d under ~60), where the signal is weakest and even a human reviewer would likely find the call ambiguous.


## 6. Interpretation


**What the model found:** Visibility-related signals (impressions, position) dominate the model's decisions far more than staleness. This is a real but as yet unremarkable discovery: it suggests that in this portfolio, *how much a page is seen* carries more predictive power for decline than *how long since it was updated*, which runs counter to the intuitive assumption behind 
the baseline rule's design.

**Negative result, stated plainly:** The model was not successful in outperforming the baseline. This is not a failure of the exercise, but rather a publishable finding, which proves that added model.  omplexity is not always superior to a well thought out and clear rule, particularly when complex environment is the only concern. 
Five basic features and strict client-holdout test.

**Surprise:**This is because the initial staleness signal check (Section 3) reported staleness , this was a phenomenon whereby a small sub-sample of the sample decreased, and not the opposite is true, as is often thought.  It was flagged as a likely small sample artifact (n=174) and not a confirmed reversal.

## 7. Recommendation


**Ranked actions:** Pages are scored with a blended formula `0.70 × baseline_score_rank + 0.30 × model_probability`  weighted toward the baseline since it validated better under honest testing. Each page receives one of three reason codes and actions:

- `stale_visible_page` → **review_for_refresh** (matches the validated baseline rule)
- `model_flagged_risk` → **monitor** (model probability ≥ 0.6, lower-confidence signal)
- neither → **no_action**

**How an editor uses this tomorrow:** Start at the top of the ranked queue (`work/outputs/action_playbook_queue.csv`) and manually review the `review_for_refresh` pages first These carry the strongest, validated signal. Treat `monitor` pages as a secondary watchlist, not an 
immediate priority, since the model's precision is lower and least reliable on low-visibility pages specifically.

**Confidence and limits, stated explicitly:** This is decision-support, not an automated system see Section 8's checklist. No page should be auto-refreshed, auto-published, or permanently deprioritized based on this score alone. The system should not be extended to a 
brand-new client with no history without re-running the client-grouped validation first.

## 8. Reproducibility


**Re-run from a fresh clone:**
```bash
git clone https://github.com/Msdff/FlyRankAiAssignment
cd FlyRankAiAssignment
pip install -r requirements.txt
```
Then open and run each notebook in order: `work/notebooks/w01_research_question.ipynb` through `w07_action_playbook.ipynb`, followed by `work/notebooks/capstone.ipynb`.

**Random seeds:** `random_state=42` used throughout (GroupShuffleSplit, RandomForestClassifier, train_test_split) for reproducibility.

**Environment:** pandas, numpy, scikit-learn, duckdb, python-dotenv (see `requirements.txt`).

**Sealed/holdout evaluation:** The client-grouped test split (7 held-out clients) was evaluated 
once per notebook run; the metrics behind every number in this report are committed at 
`work/outputs/playbook_metrics.json` and can be regenerated by re-running `w07_action_playbook.
ipynb`.

**Claims checklist:** 
 1. observed / measured / directional / decision-support language used throughout · 
 2. no causal claims made ·
 3. no "predicted Google's algorithm" claims · 
 4. no client-identifying details anywhere ·
 5. base rate (54.2%) reported alongside precision@K numbers.


## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. Learn more at [flyrank.ai](https://flyrank.ai).


