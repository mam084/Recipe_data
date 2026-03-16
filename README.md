---
layout: default
title: From Prep Time to Popularity
---
<link rel="stylesheet" href="{{ '/assets/css/style.css' | relative_url }}">

# From Prep Time to Popularity

**Matthew Mitchell**

This project studies an online recipe dataset to answer a simple question: **what recipe features are associated with popularity, measured by the number of non-zero ratings a recipe receives?** Popularity matters because it reflects which recipes actually attract attention and engagement, not just which ones receive high scores from a small number of users. That makes this dataset useful for understanding what kinds of recipes stand out on a crowded platform.

The analysis combines two source tables: a recipes table with **83,782 rows** and an interactions table with **731,927 rows**. After cleaning and merging them, the final recipe-level table keeps one row per recipe and adds a popularity outcome derived from the interactions data.

## Introduction

The central question of this project is: **Which recipe characteristics best predict recipe popularity, measured by the number of non-zero ratings?** I focus on recipe-level traits that are known at the time a recipe is posted, such as preparation time, ingredient count, number of steps, and nutritional information.

Readers should care about this question because recipe popularity is not just about taste. It also reflects visibility, accessibility, and user engagement. Understanding these relationships can help explain what kinds of recipes people are more likely to interact with, and it gives a useful example of how observable product attributes relate to attention in online platforms.

The columns most relevant to this question are listed below.

| Column | Description |
|---|---|
| `id` | Unique recipe identifier. |
| `name` | Recipe title. |
| `minutes` | Total recipe time in minutes. |
| `submitted` | Date the recipe was submitted. |
| `description` | Optional text description written by the contributor. |
| `n_steps` | Number of preparation steps. |
| `n_ingredients` | Number of ingredients. |
| `calories` | Calories parsed from the nutrition vector. |
| `sugar`, `sodium`, `protein`, `carbs`, `total_fat`, `saturated_fat` | Additional nutrition fields parsed from the original nutrition list. |
| `num_ratings` | Number of non-zero ratings received by the recipe. This is the main popularity measure. |
| `log_num_ratings` | `log(1 + num_ratings)`, used because raw rating counts are highly right-skewed. |

## Data Cleaning and Exploratory Data Analysis

I cleaned the data in a way that matches the data generating process of the source tables. In the recipes table, the `tags`, `nutrition`, `ingredients`, and `steps` columns were stored as string representations of Python lists, so I parsed them into actual lists. I converted date columns to datetime format, then expanded the seven-value `nutrition` list into separate columns: `calories`, `total_fat`, `sugar`, `sodium`, `protein`, `saturated_fat`, and `carbs`. I also created `log_minutes = log(1 + minutes)` because recipe times were strongly right-skewed.

In the interactions table, I converted dates to datetime and removed rows with `rating == 0`. Those zeros behave like placeholder or missing ratings rather than true evaluations, so keeping them would have overstated recipe popularity. I then grouped interactions by recipe and counted the remaining ratings to create `num_ratings`, merged that result back into the recipe-level data, filled missing popularity values with zero for recipes that had no non-zero ratings, and created `log_num_ratings = log(1 + num_ratings)` to stabilize the popularity distribution.

Below is the head of the cleaned recipe-level DataFrame.

<div class="table-wrap">

| id | name | num_ratings | log_num_ratings | minutes | log_minutes | n_steps | n_ingredients |
|---:|---|---:|---:|---:|---:|---:|---:|
| 333281 | 1 brownies in the world    best ever | 1 | 0.693147 | 40 | 3.713572 | 10 | 9 |
| 453467 | 1 in canada chocolate chip cookies | 1 | 0.693147 | 45 | 3.828641 | 12 | 11 |
| 306168 | 412 broccoli casserole | 4 | 1.609438 | 40 | 3.713572 | 6 | 9 |
| 286009 | millionaire pound cake | 1 | 0.693147 | 120 | 4.795791 | 7 | 7 |
| 475785 | 2000 meatloaf | 2 | 1.098612 | 90 | 4.510860 | 17 | 13 |

</div>

### Univariate analysis

The popularity distribution is heavily right-skewed, so the log transform is useful. Even after aggregation to the recipe level, most recipes receive relatively few ratings while a much smaller set attracts substantially more engagement.

<iframe class="plotly-frame" src="{{ '/assets/plots/univariate_popularity.html' | relative_url }}"></iframe>

### Bivariate analysis

The scatterplot below compares ingredient count with log popularity. The relationship appears weakly negative overall, suggesting that recipes with fewer ingredients may be slightly more likely to attract engagement, though the spread is wide enough that ingredient count alone clearly does not determine popularity.

<iframe class="plotly-frame" src="{{ '/assets/plots/bivariate_ingredients_vs_popularity.html' | relative_url }}"></iframe>

### Interesting aggregates

Grouping recipes into ingredient-count buckets shows that average popularity is somewhat higher for smaller ingredient lists, then tends to decline as recipes become more complex. The medians are low across all buckets, which reinforces that the overall popularity distribution is strongly skewed.

<div class="table-wrap">

| Ingredient bucket | Mean ratings | Median ratings | Count |
|---|---:|---:|---:|
| 1-5 | 2.816718 | 2.0 | 14164 |
| 6-10 | 2.637028 | 1.0 | 41623 |
| 11-15 | 2.496317 | 1.0 | 22810 |
| 16-20 | 2.431808 | 1.0 | 4502 |
| 21-50 | 2.704246 | 1.0 | 683 |

</div>

A similar time-bucket summary shows that very short recipes have the highest average popularity, medium-length recipes trend slightly lower, and the longest recipes are more mixed.

## Assessment of Missingness

I believe the `description` column is plausibly **NMAR**. Contributors choose whether to write a description, so missingness may depend on unobserved factors such as effort, writing style, motivation, or how complete they feel the recipe already is. Those factors are not directly recorded in the dataset. Additional contributor-level information, such as posting history or profile completeness, could make the missingness more plausibly MAR if it explained who tends to omit descriptions.

I then tested whether missingness in `description` depends on other observed columns. The notebook currently reports these results:

- Missing rate for `description`: **0.0008355**
- Comparing missingness in `description` against `n_steps`: observed median difference = **1.0**, p-value = **0.4515**
- Comparing missingness in `description` against `calories`: observed median difference = **26.7**, p-value = **0.4608**

These p-values are both well above 0.05, so the current notebook does **not** provide evidence that missingness in `description` depends on either of those columns.

<div class="note">
The project instructions require one column that the missingness <em>does</em> depend on and one that it <em>does not</em> depend on. Your current notebook only supports the second claim. Before submitting, you should rerun this section and find a genuinely significant dependency result.
</div>

The visualization below is still useful because it shows how similar the `n_steps` distributions are when `description` is missing versus not missing.

<iframe class="plotly-frame" src="{{ '/assets/plots/missingness_nsteps_hist.html' | relative_url }}"></iframe>

## Hypothesis Testing

I tested whether quick recipes and longer recipes differ in popularity.

- **Null hypothesis:** There is no difference in average recipe popularity between quick recipes (30 minutes or less) and long recipes (more than 30 minutes).
- **Alternative hypothesis:** There is a difference in average recipe popularity between the two groups.
- **Test statistic:** Absolute difference in mean `log_num_ratings`.
- **Significance level:** 0.05.
- **Observed statistic:** 0.05899.
- **p-value:** 0.00020.

This was a good test statistic because the raw popularity counts were extremely right-skewed, and comparing means on the log scale gives a more stable summary of differences in engagement. Since the p-value is far below 0.05, the data provide strong evidence against the null hypothesis. In this dataset, quick and long recipes appear to differ in average popularity. That said, this is not evidence of causation. Recipe time may be associated with other factors, such as recipe type or difficulty, that also affect popularity.

<iframe class="plotly-frame" src="{{ '/assets/plots/hypothesis_test_permutation.html' | relative_url }}"></iframe>

## Framing a Prediction Problem

The prediction problem is to predict **recipe popularity**, measured as `log_num_ratings`, using only information that would be known when a recipe is submitted. This is a **regression** problem because the response variable is continuous.

The response variable is `log_num_ratings`, rather than the raw count of ratings, because the original popularity distribution is extremely skewed. Predicting the log-transformed target makes the modeling problem more stable and makes error magnitudes easier to interpret.

At the time of prediction, I would know recipe attributes such as `minutes`, `n_steps`, `n_ingredients`, nutrition fields, and the submission year. I would not know future interactions, future ratings, or anything derived from later user engagement. That is why the model only uses recipe-level features available at posting time.

I evaluate model performance with **RMSE** on `log_num_ratings`. RMSE is appropriate for regression because it captures the typical size of prediction errors and penalizes larger misses more heavily. That is useful here because some recipes are far more popular than others.

## Baseline Model

My baseline model is a **Ridge regression** model built in a single sklearn pipeline. It uses four quantitative features and one nominal feature:

- Quantitative features: `minutes`, `n_steps`, `n_ingredients`, `calories`
- Nominal feature: `submitted_year`

Within the pipeline, `minutes` is log-transformed using `log(1 + minutes)` to reduce skew. The other quantitative features are median-imputed if necessary and left otherwise unchanged. The `submitted_year` feature is treated as categorical, imputed with the most frequent value when needed, and one-hot encoded.

The baseline model achieved a held-out test **RMSE of 0.5648646** on `log_num_ratings`.

This is a reasonable starting point, but it is not especially strong. Popularity is influenced by many factors outside the dataset, such as recommendation systems, contributor followings, timing, and broader user trends. Because of that, even a carefully built baseline may have limited predictive power.

## Final Model

The final model improves on the baseline by adding several engineered features that better reflect recipe complexity and structure:

- `ingredients_per_step`
- `minutes_per_step`
- `calories_per_ingredient`
- `log_minutes`
- `log_calories`
- `ingredients_times_steps`

These are sensible additions because they capture relationships the raw variables miss. For example, `minutes_per_step` approximates how time-intensive each step is, `ingredients_per_step` captures procedural density, and `ingredients_times_steps` reflects overall recipe complexity. The log transforms help account for skew in time and calorie values.

For the model itself, I compared **RandomForestRegressor** and **ExtraTreesRegressor** inside a pipeline with preprocessing. I selected hyperparameters using **GridSearchCV** with 3-fold cross-validation and optimized **negative root mean squared error**.

The best model was:

- **Algorithm:** `ExtraTreesRegressor`
- **n_estimators:** 200
- **max_depth:** 10
- **max_features:** 0.75
- **min_samples_leaf:** 5

Its best cross-validation RMSE was **0.5624105**, and its final held-out test RMSE was **0.5645146**.

This is an improvement over the baseline RMSE of **0.5648646**, but only a very small one. I would not oversell that gain. The final model is better, so it technically satisfies the goal of improving on the baseline, but the tiny margin suggests that recipe metadata alone only explains a limited share of popularity.

## Fairness Analysis

For fairness analysis, I compared model performance between two groups defined by recipe length:

- **Group X:** quick recipes, 30 minutes or less
- **Group Y:** long recipes, more than 30 minutes

I used **RMSE** as the evaluation metric because this is a regression problem.

- **Null hypothesis:** The final model is fair with respect to these groups. Any difference in RMSE between quick and long recipes is due to random chance.
- **Alternative hypothesis:** The final model performs worse for quick recipes than for long recipes.
- **Test statistic:** `RMSE_quick - RMSE_long`
- **Significance level:** 0.05
- **RMSE for quick recipes:** 0.5781670
- **RMSE for long recipes:** 0.5533734
- **Observed difference:** 0.0247936
- **p-value:** 0.0009990

Because the p-value is well below 0.05, the data provide evidence that the model performs worse for quick recipes than for long recipes. In that sense, the model does not appear equally accurate across these two groups.

<p class="small">This Jekyll page is based directly on the current notebook outputs. The missingness section still needs one significant dependency result before it fully meets the project requirements.</p>
