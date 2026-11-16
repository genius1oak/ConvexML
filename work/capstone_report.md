# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Collins Anjolaoluwa
- **Lane:** Refresh / Content Opportunity Scoring
- **Repository:** [https://github.com/genius1oak/ConvexML](https://github.com/genius1oak/ConvexML)
- **Date:** 16 November 2026

## 0. Abstract

This study examines whether content and keyword characteristics can help identify content items associated with declining search visibility and support content-review prioritisation. It uses an anonymized FlyRank ML Internship starter dataset containing 30,000 content records and 44 fields. A client-grouped holdout evaluation compares an oldest-content-first baseline with Logistic Regression and Random Forest using a leakage-aware feature set. Logistic Regression achieved the strongest pooled Average Precision (0.5290) and ROC-AUC (0.5330), although the oldest-content baseline performed better on the reported Precision@K measures. The resulting ranked review queue is intended to help content teams prioritise manual investigation, not to establish future decline or guarantee improvements from refreshing content.

## 1. Problem Framing

Search performance can change over time, and content teams need practical ways to identify which pages may deserve closer review. Reviewing every content item with equal priority can be inefficient, particularly when a large content inventory is involved.

The research question is:

**Can available content and keyword characteristics help distinguish content records associated with declining search performance and support a prioritised content-review process?**

The objective is to develop and evaluate a decision-support approach that ranks content records for manual investigation.

The intended users are content strategists, SEO analysts and digital marketing teams responsible for monitoring and maintaining content performance.

The project focuses on identifying associations within the available dataset. It does not attempt to reproduce Google's ranking algorithm, predict guaranteed future search performance or establish that refreshing a page will cause an improvement.

## 2. Data Description and Safety

### 2.1 Dataset

The analysis uses the anonymized FlyRank ML Internship starter dataset:

- **Records:** 30,000
- **Fields:** 44
- **Target:** `is_declining`
- **Target definition:** `is_declining = 1` where `trend_direction` is `down`; otherwise, `0`.
- **Observed decline share:** Approximately 54.21%.

The dataset contains content characteristics, keyword information, content age and freshness indicators, and historical performance measurements.

### 2.2 Data Preparation

The target was derived from `trend_direction`. Records without a usable target were excluded from model evaluation.

The dataset was reviewed for missing values, class distribution, feature suitability and potential target leakage. Missing numerical and categorical feature values were handled within the modelling pipelines using median and most-frequent-value imputation, respectively.

### 2.3 Privacy and Data Safety

The analysis uses the approved anonymized starter dataset. Client names, domains, URLs, page titles, private search queries and other identifying information are not included in the public report.

Identifiers such as `client_id` and `content_id` were not used as predictive features. Client identifiers were used only to keep client groups separate during validation.

No raw private exports or identifiable content records are reproduced in this report.

### 2.4 Data Limitations

The dataset does not provide a reliable reporting date for each observation. Consequently, the analysis is retrospective and cross-sectional rather than a prospective time-based evaluation.

The target is derived from an existing trend indicator. The results therefore describe associations with the supplied decline label and should not be interpreted as proof of future decline.

## 3. Baseline and Validation Design

### 3.1 Target Definition

The classification target is `is_declining`, derived from the supplied `trend_direction` field.

Records labelled `down` are assigned a value of 1. Other usable trend labels are assigned a value of 0.

### 3.2 Baseline

The baseline uses an oldest-content-first prioritisation rule.

Records are ranked by `days_since_last_update`, with larger values receiving higher review priority. This provides a simple operational benchmark against which the machine-learning approaches can be compared.

The baseline is not a predictive model. It represents a practical prioritisation strategy based on content-update recency.

### 3.3 Feature Selection

The selected predictive features are:

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

Target-derived fields and historical performance measures were excluded to reduce leakage risk. These exclusions include `trend_direction`, `trend_pct`, 30-day and 90-day performance metrics, CTR, average position, engagement rate, scroll rate and AI traffic percentage.

`content_id` and `client_id` were also excluded from the predictive feature set.

### 3.4 Validation Strategy

A client-grouped holdout split was used to reduce the risk of records belonging to the same client appearing in both training and test sets.

The split used `GroupShuffleSplit` with a target test size of 20% and random state 42. A suitable split was selected after checking candidate splits for the desired test size and class distribution.

The final split contained:

- **Training records:** 23,837
- **Test records:** 6,163
- **Training client groups:** 25
- **Test client groups:** 7
- **Client overlap:** Zero

The training decline rate was approximately 55.01%, while the test decline rate was approximately 51.10%.

All approaches were evaluated on the same held-out test records.

## 4. Model Development

Two classification models were developed and compared with the oldest-content-first baseline.

### 4.1 Logistic Regression

The Logistic Regression pipeline included:

- Median imputation for numerical features.
- Numerical feature scaling.
- Most-frequent-value imputation for categorical features.
- One-hot encoding of categorical features, with unknown categories ignored.
- Logistic Regression with a maximum of 1,000 iterations and balanced class weights.

### 4.2 Random Forest

The Random Forest pipeline used the same numerical and categorical preprocessing approach.

The model configuration included:

- 300 decision trees.
- Maximum tree depth of 8.
- Minimum of 10 samples per leaf.
- Balanced class weights.
- Random state of 42.

### 4.3 Leakage Prevention

Features directly defining the target or representing overlapping historical performance measurements were excluded.

Preprocessing steps were fitted within the training pipelines rather than being fitted on the full dataset before the split. This helped prevent information from the test records from influencing training-time preprocessing.

## 5. Evaluation Results

The models were evaluated using Average Precision, ROC-AUC and Precision@K.

| Approach | Average Precision | ROC-AUC | Precision@20 | Precision@50 | Precision@100 |
|---|---:|---:|---:|---:|---:|
| Oldest-content-first baseline | 0.4864 | 0.4409 | 0.65 | 0.50 | 0.51 |
| Logistic Regression | 0.5290 | 0.5330 | 0.50 | 0.48 | 0.45 |
| Random Forest | 0.5172 | 0.5166 | 0.40 | 0.44 | 0.45 |

### 5.1 Main Findings

Logistic Regression achieved the highest pooled Average Precision of 0.5290 and ROC-AUC of 0.5330.

Random Forest achieved an Average Precision of 0.5172 and ROC-AUC of 0.5166. Both machine-learning models performed better than the baseline on these two pooled ranking and discrimination metrics.

However, the oldest-content-first baseline achieved higher Precision@20, Precision@50 and Precision@100 than either machine-learning model.

The test-set decline rate was approximately 51.10%. The Precision@K results should therefore be interpreted in the context of this class distribution and the relatively small number of records in the top-ranked groups.

### 5.2 Client-Level Diagnostics

The test set contained seven client groups of different sizes. Their record counts were substantially uneven, meaning that larger groups contributed more heavily to the pooled metrics.

The unweighted mean Average Precision across the seven client groups was:

| Approach | Mean Client-Level Average Precision |
|---|---:|
| Oldest-content-first baseline | 0.5171 |
| Logistic Regression | 0.5403 |
| Random Forest | 0.5455 |

Random Forest had the highest unweighted mean client-level Average Precision, while Logistic Regression had the strongest pooled Average Precision and ROC-AUC.

The results indicate that model performance varies across client groups. Results from very small groups should be treated cautiously because their metrics are less stable.

## 6. Interpretation and Review Queue

A ranked review queue was generated using the Logistic Regression scores on the held-out test records.

The ranking score is named `decline_association_score`. It is used as a relative prioritisation signal and should not be interpreted as a calibrated probability of future decline.

The queue contains 6,163 held-out records and is divided into three review tiers:

| Priority Tier | Records | Share |
|---|---:|---:|
| High | 616 | 10% |
| Medium | 1,232 | 20% |
| Standard | 4,315 | 70% |

The high-priority tier contains records with the strongest relative model scores. These records may be considered first for manual review.

The median content age across the tiers was:

| Priority Tier | Median Content Age |
|---|---:|
| High | 105 days |
| Medium | 126 days |
| Standard | 390 days |

The median days since last update was 20 across all three tiers. This suggests that content age and time since last update do not necessarily produce identical prioritisation patterns.

The queue should be used to guide investigation rather than automatically trigger content changes. Analysts should review search intent, content accuracy, relevance, technical issues, competition and available performance evidence before deciding whether an update is appropriate.

Individual content identifiers and record-level scores are not published in this report.

## 7. Recommendations

Based on the evaluation, the following recommendations are proposed:

1. **Use the review queue as a decision-support tool.** Begin with the high-priority tier and validate each recommendation through manual review.
2. **Retain the oldest-content-first baseline.** It performed better on the reported Precision@K measures and remains a useful operational benchmark.
3. **Use Logistic Regression as the primary model candidate for further testing.** It achieved the strongest pooled Average Precision and ROC-AUC in this evaluation.
4. **Monitor performance across client groups.** Pooled results can conceal differences between groups, particularly where group sizes vary considerably.
5. **Avoid automatic content refresh decisions.** A high association score should trigger investigation, not an automatic rewrite or update.
6. **Improve future data collection.** Add reliable observation dates and repeated performance snapshots to support prospective, time-based evaluation.
7. **Conduct further validation before deployment.** Test the approach on additional approved data and monitor whether its ranking performance remains stable over time.

## 8. Reproducibility

The analysis was conducted in a Google Colab notebook and organised within the ConvexML repository.

**Project repository:**  
https://github.com/genius1oak/ConvexML

**Capstone notebook:**  
`work/notebooks/capstone.ipynb`

The notebook documents the data inspection, target construction, feature selection, grouped validation, preprocessing, model training, evaluation, interpretation and review-queue generation.

The experiment uses fixed random states for reproducibility, including random state 42 for the grouped split and model configurations.

The dataset is not included in the public repository. Reproduction requires access to the approved anonymized starter dataset in the expected project location.

The reported results are based on the executed notebook. They should be reproduced by running the notebook in the intended environment with the same dataset and compatible package versions. Any additional metrics exports or environment-version records should be maintained alongside the notebook when finalising the submission.

## 9. Acknowledgments and Data Credit

This work was completed as part of the ML Internship capstone.

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).

The analysis and conclusions presented in this report are limited to the supplied anonymized starter dataset and the methodology documented in the accompanying notebook.
