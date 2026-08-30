# Capstone Report — Refresh Opportunity Scoring

- **Author: Jai Kulkarni**

- **Lane: Refresh Opportunity Scoring**

- **Repo: https://github.com/Jai-Kulkarni1905/Applied_Search_Intelligence/tree/main**

- **Date: 29/08/2026**

## 1. Problem framing

This capstone helps decide which pages should be prioritized for human review and possible content refreshes. The unit of analysis is a page within a client, with the client grouping kept as context so pages are evaluated within the right search environment. The system creates a ranked review queue, helping an SEO or content practitioner focus limited time on pages with the clearest signs of opportunity instead of reviewing every page manually.

\
The system does not automatically recommend a content refresh. Instead, the ranked output shows which pages an analyst or content practitioner should investigate first, and the practitioner can then decide whether a refresh or another action makes sense. Poor prioritization could lead to wasted review and editorial effort on a low-value page, while overlooking a strong opportunity could delay action on a page experiencing meaningful search visibility decline.

\
Data and ML are useful because page-level search performance is shaped by several signals, including visibility, clicks, position, and performance changes over time. A transparent baseline gives the project a simple benchmark, while a learned model can test whether combining these signals improves the ranking. The goal is not to predict Google's algorithm or prove that a content refresh will restore performance, but to determine whether data-driven prioritization can make the human review process more efficient.

## 2. Data safety

The analysis uses the FlyRank pseudonymized warehouse release, mainly using the content dimension and daily content-performance data. It includes pseudonymized client and content IDs, along with search, engagement, and content information. These IDs were kept only for joining tables, grouping records, removing duplicates, and checking the data. They were not used as model features.

\
Some fields were intentionally left out. In particular, trend_direction and trend_pct were treated as fields related to the label because they describe the performance change being measured. Including them could give the model information about the outcome it is supposed to predict. Pseudonymous IDs, including client, content, URL, and keyword hashes, were used only as reference fields for joins and checks. FlyRank decision fields, such as priority or action scores, were also excluded so that the model used the available search and content data instead of copying existing decisions.

\
The leakage audit checked whether any features used information that would not have been available when the prediction was made. It also checked whether the feature and target time periods overlapped, whether calculated fields directly described the target, and whether similar records made the validation results look better than they really were. These checks were completed before the model results were interpreted.

\
The project is intended for public release. The dataset is pseudonymized, and the public files do not include client names, domains, full URLs, private search queries, passwords, or other identifying content. The published results should contain only pseudonymous IDs, summary results, approved charts, and general recommendations about pages.

## 3. Baseline

The baseline is a transparent, rule-based action score designed to rank pages for review using only information from the first half of the decision window. It first establishes position-adjusted CTR by calculating the mean CTR within each position bucket, then derives four observable components: CTR need, visibility strength, position need, and zero-visibility need. These four components are given equal weight and summed to produce the final `action_score`.

\
The score is defined as:

\
`action_score` = `ctr_need` + `visibility_strength` + `position_need` + `zero_visibility_need`

\
The individual scores flag different types of potential problems:

* `ctr_need` flags pages whose CTR is below the expected CTR for their average-position bucket. A higher value indicates a larger CTR shortfall and suggests that the page may need a title, snippet, or relevance review.

* `visibility_strength` flags pages that already have stronger search visibility, using normalized `log1p` impressions. A higher value indicates greater existing exposure and helps prioritize pages where an intervention could affect a meaningful amount of traffic.

* `position_need` flags pages with weaker average search positions. A higher value indicates poorer ranking position and suggests a potential ranking or content-optimization opportunity.

* `zero_visibility_need` flags pages with no impressions in the first half of the window. This identifies pages with complete lack of observed visibility and separates them from pages that have some exposure but weaker performance.

\
Pages are then ranked by the combined score, with the resulting queue also carrying an action level and an interpretable reason code. Because the components represent different signals, the final score balances underperformance, existing visibility, ranking position, and complete lack of visibility rather than relying on CTR alone.

\
The baseline provides a fair comparison because it is built entirely from observable first-half information and does not use the held-out outcome. The notebook explicitly verifies that scoring uses days 1–15 only, while the observed outcome is constructed from days 16 onward. It also verifies that future outcome fields and product-decision flags are not present in the scoring dataframe.

\
The baseline ranking is evaluated using **Precision@K**, with K values of 10, 25, 50, 100, 250, and 500. The held-out label defines a page as declining when its second-half impressions are lower than its first-half impressions. This same ranking metric and held-out outcome provide the basis for comparing the baseline against the later model.

## 4. Model / analysis

A Random Forest classifier was used as the learned method for the Refresh Opportunity Scoring lane. The purpose was not simply to classify pages, but to use the model's predicted probability of decline as a ranking score and test whether learning relationships among multiple page-level signals could improve prioritization beyond the baseline rule.

\
The model used the following features, all constructed from the March 1–15 decision window:

* `impressions`: total search impressions received by the page.

* `clicks`: total search clicks received by the page.

* `avg_position`: average search-result position of the page.

* `sessions_organic`: number of organic search sessions.

* `sessions_ai`: number of sessions attributed to AI-related traffic.

* `ctr`: click-through rate, calculated as clicks divided by impressions.

* `position_bucket_encoded`: ordinally encoded category representing the page's average-position bucket.

* `rf_expected_ctr`: expected CTR for the page's position bucket, used as a position-adjusted benchmark.

* `decision_impressions_avg`: average daily impressions during the decision window.

\
The `client_hash_id` and `content_hash_id` were retained as identifiers/context but were not included as model features. The future-period fields `holdout_impressions` and the outcome `is_declining` were also not used as inputs. The model was trained to predict an observed subsequent-period outcome rather than using `trend_direction` or `trend_pct` as predictors.

\
Target definition: A page is labeled `is_declining = 1` when its average daily impressions during March 16–31 are lower than its average daily impressions during March 1–15; otherwise it is labeled 0.

\
The model was evaluated as a ranking system because the practical output of this lane is a prioritized review queue, with the predicted probability of decline used to rank pages.

## 5. Evaluation

The primary evaluation uses a **client-grouped 70/30 train-test split**. `GroupShuffleSplit` was used with `client_hash_id` as the grouping variable, so pages from the same client could not appear in both the training and test sets. This was chosen to test whether the model could generalize to **unseen clients**, rather than benefiting from having pages from the same client in both datasets. The split used the March 1–15 decision-window features, while the March 16–31 holdout window was used only to construct the observed decline outcome.

The Random Forest and the Week-4 baseline were evaluated on the **same held-out test pages** using **Precision@K** at K = 10, 20, 30, 50, 100, 250, and 500. The held-out base rate was also reported to show how much better each ranking was than selecting pages without prioritization. On the primary test split, the **Random Forest did not outperform the baseline at any evaluated K**.


| K | Baseline Precision@K | Random Forest Precision@K | Difference |

| --: | --------------------: | -------------------------: | ---------: |

| 10 | 0.80 | 0.70 | -0.10 |

| 100 | 0.72 | 0.70 | -0.02 |

| 500 | 0.66 | 0.62 | -0.04 |


 The Random Forest nevertheless remained above the held-out base rate across the evaluated K values, indicating that it learned useful signal even though it did not produce a better ranking than the simpler baseline.

 \
 The notebook also performed a **five-fold client-grouped robustness check using GroupKFold**. Across the five unseen-client folds, the Random Forest's Average Precision exceeded the corresponding fold base rate, providing evidence of generalizable predictive signal. However, this did not change the main conclusion: the learned model did not improve the operational ranking over the rule-based baseline.

 \
 The error analysis examined two types of ranking failures: **highly ranked pages that did not subsequently decline** and **declining pages that the Random Forest ranked relatively low**. This shows that the model's mistakes were not simply classification errors at a fixed threshold; they were ranking errors where some non-declining pages received high scores and some genuinely declining pages were missed by the top of the queue. The notebook also compared the overlap between the top-50 baseline and Random Forest queues to understand how differently the two approaches prioritized pages.

 \
 Overall, the evaluation indicates that the Random Forest contains useful predictive signal but **does not justify replacing the simpler baseline for this prioritization task**. On the measured objective, the transparent baseline remains the stronger decision-support approach.

## 6. Interpretation

The Random Forest's permutation importance shows that the model's useful signal is concentrated in a small number of features:

| Feature | Perumutation Importance | Interpretation |

| --| :----------------------: | -----------------------|

 | `decision_impressions_avg` | 0.0844 | Strongest contributor; captures a page's average search visibility |

 | `impressions` | 0.0294 | Indicates the page's existing level of search exposure |

 | `ctr` | 0.0139 | Captures observed click-through performance |

 | `clicks` | 0.0017| Adds relatively little predictive value|

 | `avg_position` | Very small| Adds limited predictive value |

 | `session_based_features` | Add limited predictive value|

 | `position_bucket` | Negative |Did not add useful predictive contribution under held-out evaluation|

 | `expected_ctr` | Negative | Did not add useful predictive contribution under held-out evaluation |

 \
 Overall, the model relies primarily on how much search visibility a page already receives and on its observed click-through performance.

 \
The error analysis also shows why a useful model does not necessarily produce the best action queue. Some pages ranked highly by the Random Forest did not subsequently decline, while some pages that did decline were ranked lower than desired. The disagreement between the model and baseline therefore represents different prioritization behavior rather than evidence that one approach captures every opportunity.

\
The main finding is consequently not that the Random Forest discovered a causal recipe for recovering pages. Rather, the analysis shows that multiple observable search signals contain predictive information about subsequent movement, but a transparent baseline was sufficient and stronger for the specific page-prioritization task evaluated here.

## 7. Recommendation

The output should be used as a **ranked review queue**, not as an automatic instruction to refresh a page. A FlyRank editor can start at the top of the queue, review the page's priority and reason code, and then decide whether the page warrants further investigation or an actual content action.

The recommended action hierarchy is:

1. **Review first** — pages receiving the strongest prioritization should be investigated first because they represent the highest-ranked opportunities in the action queue.

2. **Review with context** — pages with weaker or incomplete signals should be checked alongside their available search-performance context before taking action.

3. **Monitor / do not prioritize** — pages without sufficient evidence for immediate intervention should remain lower in the queue rather than consuming editorial capacity.

\
The reason codes make the queue more actionable by indicating *why* a page was surfaced. In particular, `low_ctr_visible_page` identifies pages with meaningful visibility but weak CTR relative to their position, while `insufficient_visibility` indicates that there is not enough search visibility to make a reliable CTR-based review. `position_unavailable` identifies pages where average position cannot support the analysis, and `not_prioritized` identifies pages that do not meet the baseline review conditions.

\
The editor should therefore treat the ranked output as a **triage mechanism**: use the ranking to decide what to inspect first, use the reason code to understand the initial signal, and apply editorial judgment before making a content change.

\
**Confidence and limits.** Confidence is strongest in the claim that the workflow can produce a reproducible, interpretable prioritization queue from the available signals. The results do **not** establish that a recommended page will recover after a refresh, that refreshing causes improvement, or that the ranking represents Google's algorithm. The output is therefore best treated as **decision-support and directional evidence**, with final action remaining a human decision.

## 8. Reproducibility

The analysis is organized as a sequence of notebooks in `work/notebooks/`, with each stage documenting the decisions and computations used to produce the final ranking workflow. The completed notebooks provide the implementation and validation trail from problem framing through the baseline, model, validation, and action playbook.

The final repository is intended to allow the analysis to be inspected and rerun from the documented workflow. The project uses the fixed FlyRank warehouse release described in the data documentation rather than an evolving external source, so the data snapshot is part of the reproducibility specification.

The exact environment specification, package versions, and execution commands should be taken from the repository's final environment files and notebook configuration rather than reproduced manually in this report.
