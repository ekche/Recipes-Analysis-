# Do Calories Affect Recipe Ratings?

**By Eric Che** | DSC 80 Final Project at UCSD

## Introduction

This project analyzes a dataset of recipes and user ratings from food.com, which was originally collected for a recommender systems research paper. The dataset contains detailed information about recipes including cooking time, number of steps, ingredients, and nutritional content, along with ratings submitted by users.

**Central Question: Do higher-calorie recipes tend to receive lower average ratings than lower-calorie recipes?**

The dataset contains **83,782 recipes** after merging with the interactions data.

| Column | Description |
|---|---|
| `name` | Recipe name |
| `minutes` | Minutes to prepare the recipe |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients |
| `nutrition` | List of nutritional values |
| `calories` | Calories per serving |
| `avg_rating` | Average user rating per recipe |

## Data Cleaning and Exploratory Data Analysis

1. Left merged the recipes and interactions datasets on recipe ID.
2. Replaced ratings of 0 with NaN. food.com requires ratings between 1 and 5 stars. A 0 means the user left a review without a star rating so it is treated as missing.
3. Computed the average rating per recipe and merged it back as avg_rating.
4. Parsed the nutrition column into individual numeric columns for each nutrient.
5. Removed recipes over 1440 minutes and over 5000 calories as these are data entry errors.

| name | minutes | n_steps | n_ingredients | calories | avg_rating |
|---|---|---|---|---|---|
| 1 brownies in the world best ever | 40 | 10 | 9 | 138.4 | 4.0 |
| 1 in canada chocolate chip cookies | 45 | 12 | 11 | 595.1 | 5.0 |
| 412 broccoli casserole | 40 | 6 | 9 | 194.8 | 5.0 |
| millionaire pound cake | 120 | 7 | 7 | 878.3 | 5.0 |
| 2000 meatloaf | 90 | 17 | 13 | 267.0 | 5.0 |

<iframe src="assets/calories_distribution.html" width="800" height="500" frameborder="0"></iframe>

The distribution of calories is strongly right-skewed, with most recipes falling between 100 and 600 calories.

<iframe src="assets/rating_distribution.html" width="800" height="500" frameborder="0"></iframe>

The distribution of average ratings is heavily left-skewed. Nearly all recipes are rated between 4 and 5 stars, reflecting a positivity bias common on recipe platforms.

<iframe src="assets/calorie_bin_vs_rating.html" width="800" height="500" frameborder="0"></iframe>

Recipes in the highest calorie range (800+) tend to have slightly lower average ratings compared to lower-calorie recipes.

| Steps | Num Recipes | Avg Rating | Avg Calories | Avg Minutes | Avg Ingredients |
|---|---|---|---|---|---|
| 1-5 steps | 20,843 | 4.64 | 357.89 | 46.54 | 7.58 |
| 6-10 steps | 34,297 | 4.63 | 403.09 | 57.81 | 9.47 |
| 11-15 steps | 14,861 | 4.62 | 437.14 | 72.19 | 10.81 |
| 16+ steps | 10,881 | 4.60 | 476.52 | 97.43 | 12.14 |

More complex recipes have higher calorie counts and take longer, but average ratings stay consistent around 4.6 across all complexity levels.

## Assessment of Missingness

I believe the avg_rating column is likely **NMAR (Not Missing At Random)**. Recipes with no ratings are probably ones that were never discovered or tried by users. The missingness is tied to the recipe visibility on food.com, which is not captured in our dataset. To make this MAR we would need data like the number of times each recipe was viewed.

The missingness of avg_rating **does depend on n_steps** (p = 0.000). Recipes with missing ratings have significantly more steps on average, meaning more complex recipes are less likely to be tried and rated.

The missingness of avg_rating **does not depend on sodium** (p = 0.879). Sodium content has no meaningful relationship to whether a recipe received a rating.

<iframe src="assets/missingness_permutation.html" width="800" height="500" frameborder="0"></iframe>

## Hypothesis Testing

**Null Hypothesis:** Recipes with above-median calories and recipes with below-median calories have the same average rating. Any observed difference is due to random chance.

**Alternative Hypothesis:** Recipes with above-median calories have a lower average rating than recipes with below-median calories.

**Test Statistic:** Difference in mean avg_rating (high-calorie minus low-calorie).

**Significance Level:** 0.05

**p-value:** 0.033

**Conclusion:** Since 0.033 is less than 0.05 we reject the null hypothesis. The data is consistent with higher-calorie recipes receiving slightly lower ratings. However since this is an observational study we cannot conclude that calories directly cause lower ratings.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

## Framing a Prediction Problem

**Prediction problem:** Predict the average rating of a recipe based on its nutritional and structural properties.

**Type:** Regression — avg_rating is a continuous value between 1.0 and 5.0.

**Response variable:** avg_rating. I chose this because it is the most direct measure of how well a recipe is received and connects directly to the central question of this project.

**Evaluation metric:** RMSE. I chose RMSE over MAE because it penalizes larger errors more heavily. I chose it over R2 because it is interpretable in the original rating scale.

Only features known before any user rates a recipe were used: minutes, n_steps, n_ingredients, and all nutrition fields.

## Baseline Model

The baseline model predicts avg_rating using two quantitative features: minutes and n_steps. I used Linear Regression with a StandardScaler inside a single sklearn Pipeline.

**Performance:**
- RMSE: 0.6533
- R2: 0.0009

This is not a good model. An R2 near 0 means cooking time and number of steps alone explain almost no variance in ratings, which makes sense since those features say very little about how much someone will enjoy a dish.

## Final Model

I added 4 engineered features:

- **log_minutes**: cooking time is right-skewed so log-transforming compresses extreme values
- **log_calories**: same reasoning as log_minutes
- **protein_ratio**: protein divided by calories captures how protein-dense a recipe is, a proxy for healthiness
- **complexity**: n_steps times n_ingredients gives a single score for overall recipe difficulty

I switched to a **Random Forest Regressor** because recipe ratings are likely influenced by non-linear interactions. I used GridSearchCV with 3-fold cross-validation to tune n_estimators, max_depth, and min_samples_split.

**Performance:**
- Final RMSE: 0.6526
- Final R2: 0.0031
- Improvement over baseline RMSE: 0.0007

## Fairness Analysis

**Group X:** Quick recipes (minutes 30 or under)

**Group Y:** Longer recipes (minutes over 30)

**Evaluation metric:** RMSE

**Null Hypothesis:** The model is fair. Its RMSE for quick and longer recipes are roughly the same and any difference is due to random chance.

**Alternative Hypothesis:** The model is unfair. Its RMSE is higher for quick recipes than for longer recipes.

**Test statistic:** Difference in RMSE (quick minus longer)

**Significance level:** 0.05

**Results:**
- RMSE for quick recipes: 0.6277
- RMSE for longer recipes: 0.6722
- Observed difference: -0.0444
- p-value: 0.994

**Conclusion:** We fail to reject the null hypothesis. The model actually performs slightly better on quick recipes and this difference is not statistically significant. There is no evidence that our model is unfair toward either group.

<iframe src="assets/fairness_test.html" width="800" height="500" frameborder="0"></iframe>
