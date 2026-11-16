# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Collins Anjolaoluwa
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/genius1oak/ConvexML
- **Date:** 16 November 2026

## 0. Abstract

This study examines whether content and keyword characteristics can help identify content items associated with declining search visibility and support content-review prioritisation. It uses an anonymized FlyRank ML Internship starter dataset containing 30,000 content records and 44 fields. A client-grouped holdout evaluation compares an oldest-content-first baseline with Logistic Regression and Random Forest using a leakage-aware feature set. Logistic Regression achieved the strongest pooled Average Precision (0.5290) and ROC-AUC (0.5330), although the oldest-content baseline performed better on the reported Precision@K measures. The resulting ranked review queue is intended to help content teams prioritise manual investigation, not to establish future decline or guarantee improvements from refreshing content.

## 1. Problem Framing

Content teams manage multiple pages and need a practical way to identify which items may warrant closer review. Reviewing every item with equal priority can make it difficult to allocate editorial resources efficiently.

This study supports the decision of which content items should be investigated first for possible content review or refresh.

- **Unit of analysis:** An individual anonymized content item.
- **Input:** Content characteristics, keyword characteristics, and available historical content metadata.
- **Output:** A decline-association score and ranked review queue.
- **Human action:** An editor investigates higher-ranked items, checks search intent and content quality, and decides whether an update is appropriate.
- **Cost of a wrong call:** A useful page may receive unnecessary editorial attention, while a genuinely declining page may be overlooked. Unnecessary updates can waste time and potentially disrupt content that is already performing adequately.

Machine learning is useful here because it can combine multiple characteristics into a consistent ranking rather than relying on a single manual rule. However, the model is a decision-support tool and does not replace editorial judgement.

## 2. Data Safety

### Dataset Used

The analysis uses the anonymized FlyRank ML Internship starter dataset:

- File: `content_refresh_anonymized.csv`
- Records: 30,000
- Fields: 44
- Client coverage: 32 pseudonymized clients

Each record represents an anonymized content item with aggregated search-performance, engagement, content-characteristic, and historical information.

### Features Used

The model uses the following ten predictors:

**Numerical features**
- `search_volume`
- `competition`
- `cpc`
- `word_count`
- `char_count`
- `content_age_days`
- `days_since_last_update`

**Categorical features**
- `content_type`
- `main_intent`
- `competition_level`

Missing numerical values were imputed using the median, while missing categorical values were imputed using the most frequent category. These preprocessing steps were fitted within the training pipeline.

### Deliberately Excluded Fields

The following fields were excluded to reduce leakage and prevent identifiers or overlapping performance measurements from becoming predictors:

- `trend_direction` and `trend_pct`, because they define or directly derive from the target.
- 30-day and 90-day search-performance metrics, because they overlap the period used to define the observed trend.
- `ctr`, `avg_position`, `engagement_rate`, `scroll_rate`, and `ai_traffic_pct`, because they are overlapping performance indicators.
- `content_id` and `client_id`, because they are pseudonymous identifiers rather than generalizable content characteristics.
- `provider_used` and `model_used`, which were not selected as predictors.

Client identifiers were retained only for grouping during validation. They were not used as model features.

### Privacy and Safety

No client names, website domains, page URLs, private search queries, credentials, or raw client exports are included in the research outputs. Individual content and client identifiers must remain excluded from the published paper and public-facing summaries.

The study uses the approved anonymized starter dataset. Additional warehouse data is outside the scope of this analysis unless separately approved.

## 3. Baseline

The baseline is an oldest-content-first ranking heuristic.

It uses `days_since_last_update` to rank content items, assigning higher review priority to records with longer periods since their last update.

This provides a transparent reference point because it uses one understandable content-maintenance signal rather than a trained model.

On the same held-out test set, the baseline achieved:

- Average Precision: **0.4864**
- ROC-AUC: **0.4409**
- Precision@20: **0.65**
- Precision@50: **0.50**
- Precision@100: **0.51**

The baseline is useful for determining whether a more complex model provides additional ranking value.

## 4. Model / Analysis

### Target Definition

The binary target identifies whether a content item was labelled `down` in the observed trend comparison: `trend_direction == "down"` is assigned 1, while every other category is assigned 0.

The target is retrospective. It does not represent a future outcome.

### Models

Two supervised learning models were evaluated:

**1. Logistic Regression**

Logistic Regression was selected as an interpretable classification approach that can combine numerical and categorical content characteristics. The implementation used class balancing and a maximum of 1,000 iterations.

**2. Random Forest**

Random Forest was included to evaluate whether a non-linear ensemble approach could identify useful relationships among the selected features. The implementation used 300 trees, a maximum depth of 8, a minimum of 10 samples per leaf, and class balancing.

Both models used preprocessing pipelines consisting of numerical imputation and scaling, categorical imputation, and one-hot encoding.

The selected features deliberately exclude target-derived fields and overlapping performance metrics. This helps ensure that the models are evaluated using content and keyword characteristics rather than direct access to the information defining the observed trend.

## 5. Evaluation

### Validation Design

The study used a client-grouped holdout split with `GroupShuffleSplit` and random seed 42.

- Training set: 23,837 records across 25 clients.
- Test set: 6,163 records across 7 different clients.
- Client overlap between training and test sets: zero.

This design tests whether the models can rank records from clients that were not represented in the training set.

### Target Base Rate

The overall dataset contains 16,262 declining records out of 30,000, giving an overall decline rate of **54.21%**.

The held-out test set has a decline rate of **51.10%**. This is the relevant reference rate when interpreting Precision@K results. A random ranking would be expected to achieve precision close to this test-set prevalence.

### Model Comparison

All three approaches were evaluated on the same held-out test records.

| Approach | Average Precision | ROC-AUC | Precision@20 | Precision@50 | Precision@100 |
|---|---:|---:|---:|---:|---:|
| Oldest-content baseline | 0.4864 | 0.4409 | 0.65 | 0.50 | 0.51 |
| Logistic Regression | **0.5290** | **0.5330** | 0.50 | 0.48 | 0.45 |
| Random Forest | 0.5172 | 0.5166 | 0.40 | 0.44 | 0.45 |

Logistic Regression achieved the strongest pooled Average Precision and ROC-AUC. However, the oldest-content baseline performed better on Precision@20 and Precision@50 and slightly better on Precision@100.

The results therefore do not show that machine learning outperforms the simple baseline across every ranking measure.

### Client-Level Variation

Performance varied across the seven held-out client groups.

The unweighted mean Average Precision across clients was:

- Oldest-content baseline: 0.5171
- Logistic Regression: 0.5403
- Random Forest: 0.5455

Random Forest had the highest unweighted mean Average Precision across the client groups, while Logistic Regression had the strongest pooled Average Precision. Neither model performed best consistently across all clients.

This difference matters because pooled results are influenced by clients with more records, while an unweighted client average gives each client equal weight. Smaller client groups also produced less stable estimates.

### Error Analysis

The models produce rankings rather than a final editorial decision. A high-ranked item can still be a false positive for review, while an item ranked lower may still warrant investigation.

The baseline's stronger top-20 precision suggests that a simple freshness heuristic may remain useful for short review queues. Logistic Regression provides better overall pooled ranking discrimination, but its lower Precision@K values show that this advantage does not automatically translate into better results at every operational queue size.

No causal or future-prediction conclusion is drawn from these results.

## 6. Interpretation

The main finding is that content and keyword characteristics contain some useful information associated with the observed decline label, but the strength of that information depends on the evaluation measure and client group.

Logistic Regression produced the strongest pooled Average Precision and ROC-AUC. Random Forest achieved the strongest unweighted mean Average Precision across the seven test clients. The oldest-content baseline remained competitive and performed better at the reported top-ranking cutoffs.

The ranked queue also showed differences in content age:

- High-priority group median content age: 105 days.
- Medium-priority group median content age: 126 days.
- Standard-review group median content age: 390 days.

The median days since last update was 20 across all three tiers. This indicates that the model's ranking is not simply reproducing the oldest-content-first baseline.

No feature-importance or causal analysis is claimed here. The observed results should be interpreted as retrospective associations within the available anonymized dataset.

## 7. Recommendation

The Logistic Regression ranking is used to generate a review queue because it achieved the strongest pooled Average Precision and ROC-AUC on the held-out test set.

The 6,163 held-out records were divided into three review tiers:

| Review tier | Records | Share |
|---|---:|---:|
| High priority | 616 | 10% |
| Medium priority | 1,232 | 20% |
| Standard review | 4,315 | 70% |

### Suggested Editorial Workflow

**High priority**

Begin manual investigation with this group. Check whether the page still satisfies its search intent, whether its information is accurate and current, and whether its content structure needs attention.

**Medium priority**

Review these records after the high-priority group. Consider their search demand, content characteristics, and editorial importance before deciding whether an update is warranted.

**Standard review**

Retain these records in the normal monitoring and editorial review process. A lower score does not mean that a page is performing well or should never be updated.

### Confidence and Limitations

Confidence in the ranking is moderate at best and should be treated as directional. The model showed an improvement in pooled Average Precision and ROC-AUC over the baseline, but the baseline performed better on the reported Precision@K measures.

The results are limited by the retrospective target, lack of actual report dates, client-level variation, missing feature values, and the use of only seven held-out client groups.

The scores are not calibrated probabilities of future decline. They do not establish that refreshing a page will improve its search visibility. Human review and further validation are necessary before operational use.

## 8. Reproducibility

### Repository

The analysis notebook is maintained at:

`work/notebooks/capstone.ipynb`

Repository:

https://github.com/genius1oak/ConvexML

### Reproduction Procedure

1. Clone or open the ConvexML repository.
2. Ensure the authorized anonymized starter dataset is available at `data/raw/content_refresh_anonymized.csv`.
3. Open `work/notebooks/capstone.ipynb` in Google Colab.
4. Execute the notebook from the first cell using **Runtime → Run all**.
5. Confirm that data preparation, client-grouped splitting, model training, evaluation, ranked queue generation, and charts complete successfully.

The dataset is not reproduced in this report. Access to the approved data release is required to reproduce the numerical results.

### Random Seeds and Model Settings

- Grouped holdout seed: 42.
- Logistic Regression: `max_iter=1000`, `class_weight="balanced"`, `random_state=42`.
- Random Forest: 300 estimators, `max_depth=8`, `min_samples_leaf=10`, `class_weight="balanced"`, `random_state=42`.

### Environment

The notebook uses Python and the following libraries:

- pandas
- NumPy
- scikit-learn
- Matplotlib

Exact package versions should be recorded from the successful notebook environment and committed with the final repository materials.

### Evaluation Artifacts

The notebook contains the split construction, model evaluation, and result-generation steps. A machine-readable metrics export and environment-version record should also be committed to the repository so that the reported holdout results can be independently checked.

The evaluation is described as a client-grouped held-out test, not as a sealed blind evaluation. No claim is made that the test set was evaluated only once.

## 9. Acknowledgments & Data Credit

This research was completed as part of the FlyRank ML Internship Capstone. The author acknowledges FlyRank for providing the anonymized starter dataset and the opportunity to conduct this analysis.

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
