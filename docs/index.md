# Can February Search-Performance Signals Identify Future Zero-Click Content?

**Author:** Shahzaman Jatoi
**Project:** FlyRank ML Internship — Capstone

---

## Abstract

This study asks whether February search-performance signals can be used to identify and rank content items that are more likely to receive zero clicks in the following month. Using the FlyRank ML Internship dataset, February impressions, CTR, average search position, position tier, and a training-only CTR-gap feature were used to predict whether an item received zero clicks in March. A simple ML-07 baseline rule was compared with an interpretable logistic regression model using a client-level validation split to reduce leakage. On the held-out test set, logistic regression achieved 40% Precision@10, 45% Precision@20, and 24% Precision@50, compared with 20%, 10%, and 4% for the baseline respectively. The resulting ranking is intended as decision support for prioritizing content review and investigation, rather than as causal evidence or an automatic content-refresh recommendation.

---

## 1. Introduction / Problem Statement

Content teams often need to decide which pages deserve attention when search performance changes. Reviewing every content item individually can be inefficient, so a ranking system can help prioritize the items most likely to require investigation.

This project focuses on a specific and measurable outcome: whether a content item that had sufficient search activity in February received **zero clicks in March**.

### Research Question

> Can February search-performance signals be used to identify and rank content items that are more likely to receive zero clicks in the following month?

The decision supported by this analysis is prioritization. A model-generated ranking can help a reviewer decide which content items should receive earlier investigation or monitoring.

The analysis deliberately avoids making causal claims about search-engine algorithms. A high-risk score means that an item resembles other items associated with the measured outcome in the available data; it does not establish why the outcome occurred or guarantee that a content change would improve performance.

---

## 2. Data

The analysis uses the **FlyRank ML Internship dataset** and focuses on daily content-performance observations aggregated into monthly features.

### February feature window

February 2026 was used as the feature window. For each content item, daily observations were aggregated to calculate:

* **Impressions** — total February search impressions.
* **Clicks** — total February clicks.
* **CTR** — February click-through rate.
* **Average position** — impression-weighted average search position.
* **Position tier** — a categorical representation of search visibility.
* **CTR gap** — the difference between an item's February CTR and the median CTR for its position tier, calculated using training data only.

To reduce extremely sparse observations, the modeling population required at least **100 February impressions and 3 February clicks**.

### March outcome window

March 2026 was used as the outcome window. The target variable, `went_dark`, was defined as:

* `1` — the content item received zero clicks in March.
* `0` — the content item received at least one click in March.

Items without measured March activity were excluded because their March outcome could not be reliably evaluated.

### Final modeling population

After joining the February features with the March outcome using the client and content identifiers, the final dataset contained **29,368 content-item observations**.

The overall March zero-click rate was approximately **4.0%**, making the outcome relatively uncommon and motivating the use of ranking-based evaluation rather than accuracy alone.

### Validation population

To reduce the risk of client-level leakage, the data was split by **unique client**, rather than randomly splitting individual content rows.

The final split contained:

* **24 training clients**
* **6 test clients**
* **27,473 training observations**
* **1,895 test observations**
* **0 clients shared between training and test sets**

The training zero-click rate was approximately **3.97%**, while the test zero-click rate was approximately **4.12%**.

---

## 3. Methodology

The modeling workflow was designed to answer the research question while keeping the validation process leakage-aware and interpretable.

### Features

The logistic regression model used four numerical features:

1. `impressions_feb`
2. `ctr_feb`
3. `avg_position_feb`
4. `ctr_gap`

The categorical feature was:

5. `position_tier`

February impressions were transformed using `log1p` before modeling because impression counts were highly skewed. Numerical features were standardized, while the position tier was one-hot encoded.

### Target

The target was `went_dark`, representing whether the content item received zero clicks during March.

### Baseline

The ML-07 baseline was an explicit rule-based ranking score. Items received additional priority when:

* their February CTR was below the median CTR for their position tier, and
* they had at least 100 February impressions.

The baseline was used as the benchmark that the machine-learning model needed to beat.

### Model

The predictive model was an interpretable **logistic regression** classifier with balanced class weights.

Class weighting was used because zero-click outcomes represented only a small proportion of the modeling population.

The model produces a risk score representing the estimated likelihood of the `went_dark` outcome.

### Leakage control

The position-tier median CTR and `ctr_gap` were calculated using **training data only** and then applied to the test set.

The client-level split was performed before this training-derived feature calculation. This prevents information from test clients from influencing the feature construction used to evaluate the model.

### Evaluation

Because the practical goal is to prioritize a review queue, performance was evaluated using **Precision@K** rather than overall accuracy.

The selected ranking depths were:

* Precision@10
* Precision@20
* Precision@50

These metrics measure the proportion of actual zero-click items found within the first 10, 20, or 50 ranked recommendations.

The logistic regression model and ML-07 baseline were evaluated on the **same held-out client-level test set**.

---

## 4. Results

The logistic regression model was evaluated against the ML-07 baseline using the same held-out client-level test set.

| Metric       | ML-07 Baseline | Logistic Regression |
| ------------ | -------------: | ------------------: |
| Precision@10 |            20% |             **40%** |
| Precision@20 |            10% |             **45%** |
| Precision@50 |             4% |             **24%** |
| Base rate    |          4.12% |               4.12% |

The logistic regression model consistently outperformed the baseline at all three ranking depths.

At the practical review depth of 20 items, the model achieved **45% Precision@20**, compared with **10%** for the baseline.

Relative to the test-set base rate of approximately 4.12%, the model's Precision@20 represents roughly an **11× concentration of the target outcome** within the highest-ranked items.

Precision@10 was approximately 9.7× the base rate, while Precision@50 was approximately 5.8× the base rate.

These results indicate that the model provides a substantially stronger prioritization ranking than the simple baseline on this held-out client-level test set.

### Model interpretation

The logistic regression coefficients provide an interpretable view of the associations learned by the model.

The coefficients describe associations within the fitted model and should not be interpreted as causal effects.

The ranked model output was subsequently converted into a review queue containing `REVIEW`, `INVESTIGATE`, and `MONITOR` priorities.

---

## 5. Limitations & Honest Framing

This analysis is observational and is intended for decision support rather than causal inference.

The `went_dark` label represents one specific outcome: zero clicks in March. It does not capture other forms of content decline, changes in search demand, or the business reasons behind performance changes.

The analysis uses February signals to predict one following month, so performance may differ across other time periods. The model should therefore be re-evaluated before being applied to a different period.

The logistic regression model is intentionally simple and interpretable. More complex models could produce different rankings, although greater complexity would not automatically imply better decision-making.

The model also makes errors. Under the selected ranking threshold, the test set contained **61 false negatives and 61 false positives**. This demonstrates that the ranking is useful for prioritization but is not a perfect classifier.

Finally, the results do not establish that a search-engine algorithm caused any observed outcome.

The ranking should therefore be used to prioritize investigation and monitoring rather than as an automatic instruction to refresh content.

The appropriate interpretation is:

> **The model identifies items that resemble previously observed zero-click risk patterns in the available data. It does not explain why those outcomes occurred or guarantee that an intervention will improve performance.**

---

## 6. Ranked Recommendations

The model output is converted into a prioritized review queue. The purpose of the ranking is to help a reviewer decide where to spend attention first.

### REVIEW

Items with the highest predicted zero-click risk should receive the earliest human review.

### INVESTIGATE

Items with meaningful predicted risk but less extreme scores should be investigated using additional context before deciding whether any action is appropriate.

### MONITOR

Items with lower predicted risk can remain in a lower-priority monitoring queue.

### Reason Codes

The ranking provides simple reason codes to make recommendations easier to interpret:

* **HIGH_MODEL_RISK** — high predicted risk according to the fitted model.
* **MEANINGFUL_EXPOSURE** — substantial February impression volume.
* **BELOW_TIER_CTR** — February CTR below the training-set median for its position tier.
* **VISIBLE_POSITION** — relatively visible search position.
* **MONITOR** — no higher-priority reason code was triggered.

The current top-20 queue consists of items with predicted risk above 0.75 and is therefore classified as **REVIEW**.

These recommendations are prioritization signals rather than automatic actions. A human reviewer should consider additional information before making content or SEO decisions.

---

## 7. Reproducibility

The analysis was implemented in Python using DuckDB, pandas, NumPy, matplotlib, and scikit-learn.

The workflow follows these stages:

1. Load the February and March performance data.
2. Aggregate daily observations into February content-level features.
3. Apply the February minimum-exposure and minimum-click filters.
4. Aggregate March observations to construct the `went_dark` outcome.
5. Join February features with the March outcome.
6. Split unique clients into training and test groups using `random_state=42`.
7. Calculate training-set position-tier CTR medians and derive `ctr_gap`.
8. Fit the logistic regression pipeline on the training data.
9. Generate predicted risk scores for the held-out test data.
10. Compare the model against the ML-07 baseline using Precision@10, Precision@20, and Precision@50.
11. Generate model coefficients, error-analysis summaries, and ranked recommendations.
12. Save the resulting comparison and recommendation artifacts as CSV files.

The final analysis is contained in the project repository under `work/notebooks/`, including the supporting assignment notebooks and the capstone notebook.

Generated artifacts include:

* `capstone_model_comparison.csv`
* `capstone_ranked_recommendations.csv`
* `capstone_model_coefficients.csv`
* `capstone_top20.csv`

The modeling workflow uses a fixed random seed and a client-level split so that the reported evaluation can be reproduced from the same dataset release and notebook.

---

## 8. Acknowledgments & Data Credit

This project was completed as part of the **FlyRank ML Internship**.

The analysis was built on the **FlyRank ML Internship dataset** and follows the public-safe requirements of the internship.

No client names, private domains, private search queries, credentials, or raw client exports are included in this research artifact.

**Data credit:** [FlyRank](https://flyrank.ai)

---

## Conclusion

This project demonstrates an end-to-end decision-support workflow for ranking content items by observed zero-click risk.

The logistic regression model outperformed the ML-07 baseline across all evaluated ranking depths, with the strongest result at Precision@20.

The main practical value is not automatic decision-making, but the ability to turn a large content population into a prioritized queue for human review.

The results should be treated as directional evidence from the available dataset and validation period rather than as causal or universally generalizable conclusions.
