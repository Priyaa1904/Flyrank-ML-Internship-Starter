# Capstone Report — Content Performance Prioritization

- **Author:** Priya Pandey
- **Lane:** Content Performance / SEO
- **Repo:** https://github.com/Priyaa1904/Flyrank-ML-Internship
- **Date:** 14 August 2026

> Copy this file to `work/capstone_report.md` and fill it in as you build. Sections 1–8
> mirror the Pass / Needs-Work rubric axes, so nothing here is optional. Sections 0 and 9
> are **paper sections**: your deployed research paper must carry both, and they're here so
> you never rebuild them from memory at ship time.

## 0. Abstract

This study asks whether observable content and search-performance signals can help prioritize potentially declining content for human review. Using 30,000 content records across 32 clients from the FlyRank ML Internship dataset, the analysis defines an `is_declining` outcome from a recent 30-day trend and uses eight observable features to train an interpretable Logistic Regression model. The model is compared with the Week-4 baseline using Precision@10 and evaluated under both a random row-level split and a client-grouped split that keeps clients entirely separate between training and testing. The Logistic Regression model measured 0.40 Precision@10 under the client-grouped evaluation, compared with 0.30 for the Week-4 baseline, while the random row-level split produced a higher 0.60 result, showing that validation design materially affects the measured performance. The resulting output is therefore intended as a ranked decision-support queue for human review, not as an autonomous system for changing content or claiming causal SEO effects.


## 1. Problem framing

This project supports the decision of which content items should be reviewed first when there are signals of potential decline.

The unit of analysis is an individual content item/page associated with a pseudonymous client identifier. Client identifiers are used for grouping during validation, but are not used as model features.

The model produces a probability score for the `is_declining` outcome. Content items can then be ranked by this score and translated into a review queue containing `REVIEW`, `INVESTIGATE`, and `REFRESH` action categories.

A human reviewer can use the ranked queue to decide which pages deserve closer inspection first. Before taking action, the reviewer should check current page relevance, observed performance signals, recent content changes, search intent and page purpose, business/editorial/legal/brand context, and expected effort versus potential value.

A wrong call has two main costs. A false positive can direct limited editorial resources toward a page that does not require intervention, while a false negative can cause a potentially important page to receive less attention than it otherwise might.

Machine learning is useful here because content performance can involve several observable signals at the same time. Logistic Regression provides a simple and interpretable way to combine eight observable features into a ranking score.

The model is not intended to determine why a page declined or to establish that a particular intervention will improve performance. Its role is to provide a measured ranking signal that can support human review.

## 2. Data safety

The modeling dataset used for this analysis contains 30,000 content records across 32 clients. Each record contains content-performance and search-related signals used to assess whether the content was classified as declining.

The final model used these eight features:

- `days_since_last_update`
- `impressions_90d`
- `ctr`
- `avg_position`
- `word_count`
- `search_volume`
- `competition`
- `cpc`

Pseudonymous identifiers such as `client_id` and `content_id` were retained for record identification and client-grouped validation, but were not used as model features.

### Deliberately excluded fields

Target-derived fields were excluded from the model. In particular, `trend_direction`, `trend_pct`, and the target-related label fields were not used as model inputs.

The dataset also contained `is_declining_label`, which was identified as a potential decision-derived/product field. It was not included among the model features.

These exclusions are important because using information derived from the outcome would allow the model to learn the answer rather than independently estimate it.

### Leakage testing

The final feature audit found:

`Forbidden features present: []`

A deliberate leakage experiment was then performed by adding `trend_pct` to the model features.

The resulting Precision@10 increased to:

**1.00**

The honest client-grouped model achieved:

**0.40**

This large difference demonstrated that a target-related feature could create an artificially strong result. `trend_pct` was therefore excluded from the final model.

### Temporal leakage review

The target is derived from a recent 30-day trend relative to the previous 30-day period.

Some available model signals use a 90-day observation window. In particular, `impressions_90d`, `ctr`, and `avg_position` were flagged for temporal-overlap review.

This does not by itself establish that these features are leaking the target, but it identifies a methodological limitation that should be checked against an exact prediction timestamp before production use.

### Data-safety conclusion

The final eight-feature model contains no identified forbidden or target-derived columns. Client identifiers are used only for grouping and record identification, never as predictive features.

The leakage experiment provides an explicit receipt that target-derived information can substantially inflate the measured score, supporting the decision to exclude those fields from the final model.

## 3. Baseline

The Week-4 baseline provides a simple reference point for determining whether the machine-learning model adds useful ranking signal.

The baseline was evaluated using the same modeling population and the same primary metric, Precision@10. This makes it a transparent comparison rather than an unrelated benchmark.

The Week-4 baseline achieved:

**Precision@10 = 0.30**

The baseline is intentionally simple. Its purpose is to establish what performance can be obtained from the earlier rule-based approach before introducing a learned model.

The Logistic Regression model subsequently achieved **0.40 Precision@10 under the client-grouped evaluation**, giving a measured improvement of **0.10 Precision@10 points** over the baseline.

| Approach | Precision@10 |
|---|---:|
| Week-4 baseline | 0.30 |
| Logistic Regression | 0.40 |

The comparison should be interpreted as evidence that the Logistic Regression ranking provided additional measured signal over the Week-4 baseline under the grouped evaluation. It does not establish that the model will improve content performance or that its recommendations are causal.

## 4. Model / analysis

The task was treated as a binary classification and ranking problem. The model estimates the probability that a content item belongs to the `is_declining` class, and these probabilities are used to rank items for review.

### Target definition

The target is `is_declining`, derived from the observed recent 30-day trend relative to the preceding 30-day period.

The target was kept separate from the model features so that the model did not directly receive information derived from the outcome.

### Model choice

Logistic Regression was selected because the task has an observed binary outcome and the model provides interpretable coefficients.

The final model used the following eight features:

- `days_since_last_update`
- `impressions_90d`
- `ctr`
- `avg_position`
- `word_count`
- `search_volume`
- `competition`
- `cpc`

### Preprocessing

The model was implemented as a pipeline consisting of:

1. Median imputation for missing numerical values
2. StandardScaler feature standardization
3. Logistic Regression with `max_iter=1000` and `random_state=42`

Keeping preprocessing within the pipeline ensures that the transformations are fitted as part of the training process rather than separately using the test data.

### Features deliberately left out

The following types of information were deliberately excluded:

- Target-derived trend fields such as `trend_direction` and `trend_pct`
- The target label itself
- `is_declining_label` as a decision-derived/product field
- `client_id` and `content_id` as predictive features

Client identifiers were retained only where needed for record identification and client-grouped validation.

### Feature interpretation

The fitted Logistic Regression coefficients provided a simple way to inspect which features contributed most strongly to the model's ranking.

The three largest absolute coefficients were:

| Feature | Coefficient |
|---|---:|
| `ctr` | -0.197 |
| `days_since_last_update` | 0.189 |
| `word_count` | 0.133 |

These coefficients describe the directional associations learned by the model. They should not be interpreted as causal effects.

The feature ranking was used as an interpretation aid and sanity check rather than as evidence that changing any individual feature would produce a particular outcome.

## 5. Evaluation

The model was evaluated using Precision@10 because the practical objective is to rank a small set of content items for initial human review.

The evaluation included both a random row-level split and a client-grouped split. The grouped split was used as the more conservative evaluation because content from the same client can share characteristics that would not be available when evaluating a completely unseen client.

### Random row-level evaluation

The random split contained:

- Training rows: 24,000
- Test rows: 6,000
- Test base rate: 0.542
- Precision@10: **0.60**

This result provides a useful reference point but may be optimistic because the same clients can appear in both training and testing data.

### Client-grouped evaluation

For the grouped evaluation, all rows belonging to a client were kept entirely within either the training or test set.

The resulting split contained:

- Training rows: 23,837
- Test rows: 6,163
- Clients shared between train and test: **0**
- Test base rate: 0.511
- Precision@10: **0.40**

The grouped result is treated as the more conservative measure of generalization to clients that were not observed during training.

### Validation comparison

| Evaluation approach | Precision@10 | Test base rate |
|---|---:|---:|
| Random row-level split | 0.60 | 0.542 |
| Client-grouped split | **0.40** | 0.511 |

The measured Precision@10 decreased from 0.60 to 0.40 when the evaluation changed from a random row-level split to a client-grouped split.

This gap is itself an important finding. It shows that the measured model performance is sensitive to the validation design. The analysis therefore does not use the 0.60 random-split result as evidence of generalization to unseen clients.

### Model versus baseline

On the comparable grouped evaluation, the Logistic Regression model achieved **0.40 Precision@10**, compared with **0.30 for the Week-4 baseline**.

| Approach | Precision@10 |
|---|---:|
| Week-4 baseline | 0.30 |
| Logistic Regression | **0.40** |

The model therefore measured a 0.10 Precision@10 improvement over the baseline under the grouped evaluation.

The positive-class test base rate for this grouped evaluation was 0.511. Precision@10 should therefore be interpreted alongside the base rate rather than as a standalone accuracy-like number.

### Error analysis

The grouped evaluation produced **2,995 total classification errors** when predictions were converted to binary labels.

Three observed false-positive examples had predicted probabilities close to the decision boundary:

| Probability | Days since update | Impressions | CTR | Average position |
|---:|---:|---:|---:|---:|
| 0.549 | 103 | 307 | 0.00 | 39.8 |
| 0.511 | 20 | 371 | 1.35 | 5.4 |
| 0.547 | 20 | 16 | 0.00 | 4.6 |

These examples illustrate that the model can flag items that do not ultimately belong to the declining class. Some have weak visibility or engagement signals, while others have relatively recent updates, showing that the available features do not always cleanly separate the two outcomes.

Three observed false-negative examples were:

| Probability | Days since update | Impressions | CTR | Average position |
|---:|---:|---:|---:|---:|
| 0.497 | 25 | 15,320 | 0.05 | 20.3 |
| 0.489 | 8 | 9 | 0.00 | 10.1 |
| 0.489 | 20 | 25 | 4.00 | 6.2 |

These cases show why the model should not be treated as a definitive decision system. Some declining items receive probabilities just below the classification threshold, indicating that the available signals can be ambiguous.

### Evaluation conclusion

The strongest measured result is the **0.40 Precision@10 under client-grouped validation**, compared with the **0.30 Week-4 baseline**.

The higher 0.60 random-split result is retained as a diagnostic comparison, not as the primary generalization claim.

## 6. Interpretation

The Logistic Regression model provides an interpretable ranking signal based on the eight observable features rather than an opaque prediction rule.

### What the model relied on

The three features with the largest absolute coefficients were:

| Feature | Coefficient | Direction |
|---|---:|---|
| `ctr` | -0.197 | Negative |
| `days_since_last_update` | 0.189 | Positive |
| `word_count` | 0.133 | Positive |

The coefficients indicate the directional associations used by the fitted model. They do not show that changing a feature will cause the target outcome to change.

### What this suggests

Within this dataset and fitted model, **CTR, content freshness, and word count were among the strongest model signals**.

The positive coefficient for `days_since_last_update` is directionally consistent with the idea that older content can be associated with declining outcomes. The negative coefficient for CTR indicates that higher CTR values were associated with lower predicted probability of the positive class in the fitted model.

These are model-level associations rather than causal findings.

### Validation was an important finding

The largest methodological insight was not simply the model's score, but how that score changed with the validation design.

Precision@10 was:

- **0.60** under the random row-level split
- **0.40** under the client-grouped split

The grouped evaluation prevents the same client from appearing in both training and test data. The reduction in measured performance therefore demonstrates that the evaluation design materially affects the reported result.

Rather than treating the 0.60 result as the headline performance, this analysis uses the more conservative **0.40 grouped result** when discussing generalization to unseen clients.

### Leakage experiment

The deliberate leakage test produced another important result.

Adding `trend_pct`, a target-related feature, increased Precision@10 to **1.00**.

This was not treated as a model improvement. Instead, it demonstrated how target-derived information can create an artificially strong result. The feature was removed from the final model.

### Error patterns

The model produced both false positives and false negatives, including examples with predicted probabilities close to the classification threshold.

This indicates that the available features do not perfectly separate declining and non-declining content. Some pages can look similar in their observable signals while having different observed outcomes.

The errors therefore reinforce the need for human review rather than automatic action.

### Negative and cautionary findings

The analysis did not establish that updating content, increasing word count, improving CTR, or changing another individual feature will cause better search performance.

It also did not establish that the model generalizes equally well to all clients or content environments.

The useful finding is narrower: the model measured a ranking signal that was stronger than the Week-4 baseline under client-grouped validation and can therefore support prioritization for human review.

## 7. Recommendation

The model output is best used as a ranked content-review queue. It should help a FlyRank editor decide where to spend attention first, while leaving the final decision to a human reviewer.

### Ranked actions

The resulting queue contains 30,000 content items across 32 clients and assigns three action categories:

| Action | Records | Share |
|---|---:|---:|
| `REVIEW` | 17,790 | 59.30% |
| `INVESTIGATE` | 10,576 | 35.25% |
| `REFRESH` | 1,634 | 5.45% |

### 1. Review potential decline signals

`REVIEW` is the largest category and represents the broadest set of items requiring human inspection.

The reviewer should first verify whether the observed signals correspond to a meaningful content problem before considering any intervention.

### 2. Investigate fresh content showing decline signals

The `FRESH_DECLINING` reason code accounts for 10,576 items (35.25%).

These items are worth investigating because the observed decline signal occurs despite the content being relatively fresh. The reviewer should check search intent, recent changes, performance signals, and other contextual information before deciding whether any action is appropriate.

### 3. Consider refresh candidates among stale, visible content

The `STALE_HIGH_VISIBILITY` category contains 1,634 items (5.45%).

These pages combine observed staleness with relatively substantial visibility signals and may therefore be reasonable candidates for closer human review before considering a refresh.

This does not mean that refreshing the content will improve its performance. It only identifies pages that may deserve attention based on the observed signals.

### Intended workflow

The recommended workflow is:

**Rank → Review → Investigate → Decide → Measure**

A reviewer should:

1. Check current page relevance.
2. Verify observed performance signals.
3. Check recent content changes.
4. Review search intent and page purpose.
5. Consider business, editorial, legal, and brand context.
6. Assess effort against potential value.

### What should not be automated

The queue should not be used to automatically:

- publish or rewrite content;
- delete or redirect pages;
- change factual or sensitive claims;
- make legal, medical, or financial decisions;
- treat the action score as causal evidence;
- override human reviewers.

No content action should be executed solely from the queue. A human reviewer must make the final decision.

### Confidence and limitations

Confidence in the ranking should be considered **moderate and conditional on the validation design**.

The model measured 0.40 Precision@10 under client-grouped validation, compared with 0.30 for the Week-4 baseline. However, the random row-level result was higher at 0.60, demonstrating that the measured performance is sensitive to how clients are divided between training and testing.

The recommendations should therefore be treated as **decision-support**, not as guaranteed interventions or causal prescriptions.

### Monitoring and retraining

The output should be monitored for changes in:

- missingness and data quality;
- population composition;
- feature distributions;
- action-category mix;
- observed out-of-sample performance.

Unexpected changes should trigger human investigation. Retraining should not be automatic; the cause of data drift or performance deterioration should be understood before changing the model.

## 8. Reproducibility

The project is organized as a sequence of notebooks covering baseline construction, model training, validation auditing, and action-playbook development.

### Key notebooks

- `work/notebooks/capstone.ipynb` — final capstone analysis and paper artifacts
- `work/notebooks/w06_validation_audit.ipynb` — grouped validation, leakage audit, and error analysis
- `work/notebooks/w07_action_playbook.ipynb` — ranked action queue, human-review rules, and monitoring guidance

### Reproducibility settings

The Logistic Regression pipeline uses `random_state=42` and `max_iter=1000`.

The analysis also uses fixed random-state settings for the relevant train/test procedures so that the reported comparisons can be reproduced under the same data and environment.

### Main evaluation receipts

The notebooks retain the evidence supporting the main findings:

- Week-4 baseline Precision@10: **0.30**
- Random row-level Precision@10: **0.60**
- Client-grouped Precision@10: **0.40**
- Random test base rate: **0.542**
- Grouped test base rate: **0.511**
- Clients shared between grouped train and test: **0**
- Deliberate leakage test with `trend_pct`: **1.00**
- Total binary classification errors in the grouped evaluation: **2,995**

The validation audit also records the feature leakage check and examples of false-positive and false-negative predictions.

### Action-playbook artifacts

The Week-7 notebook generates the ranked action queue at:

`work/outputs/ranked_action_queue.csv`

The queue contains 30,000 rows and 9 exported columns. It is regenerated by the notebook rather than treated as a manually maintained dataset.

### Source repository

The complete project is available at:

https://github.com/Priyaa1904/Flyrank-ML-Internship

The repository contains the analysis notebooks, figures, outputs, and supporting project artifacts needed to inspect the workflow.

### Re-running the work

The analysis should be rerun from the notebooks in their intended sequence using the same dataset and environment. The reported headline numbers should be checked against a fresh notebook run before being treated as final.

This project does not make a sealed-holdout claim. Therefore, no sealed evaluation receipt is claimed.

## 9. Acknowledgments & data credit

Built on the **FlyRank ML Internship dataset**.

Data source: https://flyrank.ai

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> **Metrics vs. base rate:** report your task's base rate (majority-class %) next to any
> precision@K or accuracy — a high score can just be a high base rate. AUC / lift over
> baseline are the honest discrimination numbers.
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
