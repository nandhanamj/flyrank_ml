# Ranking Content Review Candidates Using Search and Engagement Signals

**Author:** Nandhana M J  
**Lane:** Refresh / Content Opportunity Scoring  
**Repo:** https://github.com/nandhanamj/flyrank_ml  
**Date:** September 19, 2026  

## 0. Abstract

This project asks whether a learned ranking model can improve prioritization of content items for editorial review compared with a transparent rule-based baseline. The analysis uses pseudonymized FlyRank search and engagement data, with 158,466 content items from 45 clients and an April 2026 future-decline outcome defined relative to March 2026 performance. A Logistic Regression model was trained using a client-grouped holdout split and compared with the Week-4 baseline using Precision@50. On the held-out test set, the model identified 37 future-declining items in its top 50 (Precision@50 = 0.740), while the baseline identified 0 of 50 (Precision@50 = 0.000). The resulting ranking is intended as decision-support for human review and does not establish that refreshing a selected page would cause improved search performance.

## 1. Problem framing

This project supports the decision of which content items an editor or SEO practitioner should review first for possible refresh or further investigation.

The unit of analysis is one pseudonymized content item/page at a defined reporting date. The output is a ranked review queue, where higher-ranked items have a higher model-estimated probability of the observed future-decline outcome.

A human editor uses the ranking as a prioritization aid: they review the highest-ranked items first, inspect the item's search and engagement signals and actual editorial history, and then decide whether a refresh or other action is appropriate.

A wrong call can waste editorial effort on content that does not need attention, while a missed opportunity can delay investigation of content whose search performance subsequently declines.

Data and ML are useful because multiple observed signals can interact when identifying review candidates, including search visibility, clicks, average position, engagement, content characteristics, and backlinks. A learned ranking model can therefore be compared with a transparent rule-based baseline to test whether these combined signals improve prioritization on unseen client groups.

## 2. Data safety

The analysis uses pseudonymized FlyRank search and engagement data at the content-item level. The modeling dataset contains 158,466 rows across 45 pseudonymized clients.

The model features were:

- `days_stale`
- `march_impressions`
- `march_clicks`
- `march_avg_position`
- `march_pageviews`
- `march_engaged_sessions`
- `search_volume`
- `word_count`
- `backlinks`

The following fields were deliberately excluded from the model:

- `trend_direction`
- `trend_pct`

These fields were excluded because the future-decline target is based on observed performance direction, and using trend-derived fields could introduce label leakage.

`client_hash_id` was used only for grouped validation and was not used as a predictive feature. `content_hash_id` was retained only as a pseudonymous identifier for tracing ranked recommendations and was not used as a predictive feature.

The future target was constructed from the April 2026 outcome relative to March 2026 performance. Future-window information was not used as a model feature.

The analysis does not publish client names, domains, private URLs, private search queries, credentials, or raw warehouse exports. Published recommendation outputs use pseudonymized identifiers only.
## 3. Baseline

The baseline is a transparent two-signal action score built from information available at the March 31, 2026 decision point.

The first signal is **content staleness**: older content receives more points. The second is **search visibility**: content with fewer March GSC impressions receives more points. The total baseline score is the sum of these two signal scores, with higher scores indicating higher review priority.

The baseline assigns reason codes so that the ranking remains interpretable:

- `STALE_AND_LOW_VISIBILITY` — both priority signals are present
- `STALE` — staleness indicates priority
- `LOW_VISIBILITY` — low visibility indicates priority
- `REVIEW` — neither signal crosses the priority threshold

Items with at least one priority signal receive the `REFRESH` action label; otherwise they receive `REVIEW`.

This provides a fair comparison because the baseline uses only signals available at the same March 31 decision point as the model and is evaluated against the same held-out future outcome.

On the held-out test set, the baseline's top 50 contained **0 future-declining items**, giving a **Precision@50 of 0.000**.

## 4. Model / analysis

The learned model is a **Logistic Regression** ranking model. It was chosen because the task requires a simple, interpretable scoring model that produces an estimated probability for each content item and can be compared directly with the rule-based baseline.

The exact model features were:

- `days_stale`
- `march_impressions`
- `march_clicks`
- `march_avg_position`
- `march_pageviews`
- `march_engaged_sessions`
- `search_volume`
- `word_count`
- `backlinks`

Missing numeric values were median-imputed and the features were standardized before fitting Logistic Regression.

The target is defined as:

> `is_declining_future = 1` when April 2026 total GSC clicks are lower than March 2026 total GSC clicks; otherwise `0`.

The model uses only information available at the March 31, 2026 decision point. Future April performance is used only to construct the evaluation label and is not supplied as a feature.

The model was evaluated against the same transparent baseline using the same held-out test set. The model produces a probability score, which is used to rank content items from highest to lowest estimated probability of the observed future-decline outcome.

The analysis does not claim that any individual feature causes future search performance to change. Logistic Regression coefficients are interpreted as associations within the fitted model.
## 5. Evaluation

The evaluation uses a client-grouped holdout split so that content from the same client does not appear in both training and testing. The modeling dataset contains 158,466 rows across 45 clients. The split produced 125,592 training rows from 36 clients and 32,874 test rows from 9 clients, with zero client overlap between the two sets.

This design tests whether the ranking generalizes to clients that were not used for model fitting, rather than allowing the model to benefit from seeing the same client's pages in both sets.

The future-decline label distribution in the full modeling dataset was 44,295 positive rows and 114,171 negative rows, giving a future-decline base rate of approximately **27.95%**. This base rate is important context when interpreting Precision@50.

Both the baseline and Logistic Regression model were evaluated on the same held-out test set.

| Method | Precision@50 | Future declines in top 50 |
|---|---:|---:|
| Week-4 baseline | 0.000 | 0 / 50 |
| Logistic Regression | 0.740 | 37 / 50 |

The Logistic Regression model therefore identified 37 observed future declines among its top 50 ranked items, while the baseline identified none.

### Error analysis

Among the model's top 50 predictions, 37 were future declines and 13 were false positives. Some false positives had substantial March search activity, showing that a high model score does not guarantee a future decline.

The result is an observed ranking measurement on one client-grouped holdout split. It should not be interpreted as evidence that the model will achieve the same performance on future months or different client populations.

## 6. Interpretation

The Logistic Regression ranking was driven most strongly by standardized March average position and March impressions. The largest fitted coefficients were:

| Feature | Coefficient |
|---|---:|
| `march_avg_position` | -0.679 |
| `march_impressions` | 0.657 |
| `word_count` | 0.136 |
| `days_stale` | -0.105 |
| `march_engaged_sessions` | 0.080 |
| `march_clicks` | 0.062 |
| `search_volume` | -0.042 |
| `march_pageviews` | 0.027 |
| `backlinks` | 0.007 |

These coefficients describe associations used by the fitted model. They are not causal effects and should not be interpreted as evidence that changing any individual feature would cause future search performance to change.

One important data-quality finding was that `days_stale` should not be interpreted as a reliable freshness measure in this evaluation. In the held-out test set, 95.6% of rows had negative `days_stale` values, meaning their recorded content update dates were later than the March 31, 2026 decision point. This makes the staleness feature difficult to interpret substantively even though it was included in the fitted model.

The model's false positives were also informative. Several top-ranked false positives had substantial March search activity, showing that the model can assign a high future-decline probability even when an item currently has meaningful visibility or clicks.

The main observed result is therefore a ranking improvement over the tested baseline on this holdout split, rather than evidence of a particular causal mechanism behind future content decline.

## 7. Recommendation

The model output should be used as a **ranked review queue**, not as an automatic refresh list.

A FlyRank editor can use the queue in the following workflow:

1. Start with the highest-ranked content items.
2. Review the item's March search and engagement signals.
3. Check the actual content update history and editorial context.
4. Decide whether a refresh or another action is appropriate.
5. Monitor later performance after any intervention.

The ranking provides a way to prioritize limited editorial review time. It does not determine which pages must be refreshed.

Confidence in the ranking is **moderate and evaluation-specific**. The model achieved Precision@50 of 0.740 on the held-out client-grouped test set, but this was measured on one future month and nine held-out clients. The substantial data-quality issue in `days_stale` also limits interpretation of that feature.

The 37 future declines in the model's top 50 are observed matches to the evaluation label. They do not demonstrate that refreshing those pages would prevent decline or improve traffic, clicks, rankings, or conversions.

The ranked recommendations therefore support **human review and prioritization**, with the final editorial decision remaining with the practitioner.

## 8. Reproducibility

The main analysis is implemented in the capstone notebook:

`work/notebooks/capstone.ipynb`

The notebook contains the data loading, feature construction, grouped train/test split, Logistic Regression model, baseline reconstruction, evaluation, error analysis, ranked recommendations, and reproducible result artifacts.

The evaluation uses `random_state = 42` for the client-grouped holdout split.

The main modeling environment uses Python with pandas, NumPy, scikit-learn, DuckDB, Hugging Face Hub access, and Matplotlib. The repository's `requirements.txt` provides the project environment dependencies.

The analysis uses a client-grouped holdout evaluation rather than a random row-level split. The test set contains 32,874 rows from 9 clients, with zero client overlap with the training set.

The notebook also contains the code used to reconstruct the baseline and directly calculate Precision@50 from the exact top-50 rows. This keeps the reported metric tied to the displayed evaluation ranking.

The main reported evaluation values are:

- Baseline Precision@50: **0.000**
- Logistic Regression Precision@50: **0.740**
- Baseline future declines in top 50: **0 / 50**
- Logistic Regression future declines in top 50: **37 / 50**

The analysis is reproducible from the committed notebook and repository code. The hosted full-release data access requires the appropriate Hugging Face authentication described in the repository setup instructions.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset. See https://flyrank.ai.

## Claims checklist

- Observed / measured / directional / decision-support language used throughout.
- Future-decline base rate reported alongside Precision@50.
- No causal claims about refreshing content.
- No claim that the model predicts Google's algorithm.
- No client-identifying details, private URLs, private queries, or credentials.
- Model and baseline are evaluated on the same client-grouped held-out test set.
