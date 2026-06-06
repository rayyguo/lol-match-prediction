# Predicting Victory in Professional League of Legends

By Raymond Guo

## Introduction

League of Legends is one of the most popular esports in the world, with professional leagues spanning many regions and thousands of matches played each year. This project uses the 2022 League of Legends Esports Match Data from Oracle's Elixir, which records detailed player-level and team-level statistics from professional matches.

The central question of this project is: **Can the outcome of a professional League of Legends match be predicted using team-level performance statistics?**

This question matters because competitive League of Legends is built around converting small advantages into map control, objectives, and eventually a win. By studying which team statistics separate wins from losses, we can better understand what types of in-game performance are most closely associated with victory.

For this analysis, I restricted the dataset to team-level rows where `position` is `team`. After filtering and cleaning, the dataset contains **25,058 rows**. The columns most relevant to my question are:

- `result`: the match outcome, where `1` means a win and `0` means a loss.
- `kills`: the number of enemy champions killed by the team.
- `towers`: the number of enemy towers destroyed by the team.
- `dragons`: the number of dragons secured by the team.
- `barons`: the number of Baron Nashor objectives secured by the team.
- `gamelength`: the length of the game in seconds.

## Data Cleaning and Exploratory Data Analysis

I cleaned the data by loading the original Oracle's Elixir CSV, filtering to team-level rows, selecting the six columns listed above, and dropping rows with missing values in those selected columns. This made the analysis focus on one row per team per game, rather than mixing individual player statistics with team totals. The resulting DataFrame contains only numeric variables needed for the analysis and modeling steps.

Here is the head of the cleaned DataFrame:

| result | kills | towers | dragons | barons | gamelength |
|---:|---:|---:|---:|---:|---:|
| 0 | 9 | 3.0 | 1.0 | 0.0 | 1713 |
| 1 | 19 | 6.0 | 3.0 | 0.0 | 1713 |
| 0 | 3 | 3.0 | 1.0 | 0.0 | 2114 |
| 1 | 16 | 11.0 | 4.0 | 2.0 | 2114 |
| 1 | 13 | 8.0 | 2.0 | 1.0 | 1365 |

For univariate analysis, I first examined the distribution of team kills. Most team-game rows have a moderate number of kills, with fewer teams reaching very high kill totals. This makes sense in professional play, where games are often controlled and teams do not always need extreme kill counts to win.

<iframe
  src="kills_histogram.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

For bivariate analysis, I compared towers destroyed across wins and losses. Winning teams usually destroy many more towers than losing teams, which is expected because destroying towers opens the map and is often required to reach the opposing Nexus.

<iframe
  src="towers_by_result.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

The grouped table below shows average team statistics by match result:

| result | kills | towers | dragons | barons |
|---|---:|---:|---:|---:|
| Loss | 9.37 | 2.84 | 1.43 | 0.22 |
| Win | 19.61 | 9.24 | 3.03 | 1.12 |

Winning teams have higher averages for every listed statistic. The gap is especially large for towers, which supports the idea that objective control is strongly related to match outcome.

<iframe
  src="aggregate_by_result.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

## Assessment of Missingness

I examined missingness in the original team-level dataset using the `goldat25` column, which records a team's gold total at 25 minutes. This column has nontrivial missingness: 5,110 of the 25,058 team rows, or about 20.39%, are missing. I do not believe this column is best described as **NMAR**, because the missingness is likely explained by the data generating process rather than the unobserved gold value itself. If a game ends before the 25-minute mark, then there is no 25-minute gold value to record. This makes `gamelength` an important observed variable that can explain the missingness.

For the missingness dependency tests, I used `goldat25_missing`, an indicator for whether `goldat25` was missing. I found that missingness depends on `gamelength`: rows with missing `goldat25` had games that were about 218.18 seconds shorter on average than rows where `goldat25` was present. Using the difference in mean game length as the test statistic, the permutation-test p-value was 0.000.

<iframe
  src="goldat25_missing_by_gamelength.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

<iframe
  src="goldat25_gamelength_permutation.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

As a comparison, I tested whether `goldat25` missingness depends on `side`. The missingness rate was 20.39% for both Blue and Red side, the observed difference was 0, and the p-value was 1.000. This suggests that `goldat25` missingness does not depend on side.

## Hypothesis Testing

I tested whether winning teams destroy more towers than losing teams.

Null hypothesis: winning teams and losing teams destroy the same number of towers on average, and any observed difference is due to random chance.

Alternative hypothesis: winning teams destroy more towers than losing teams on average.

I used a permutation test with the difference in mean towers destroyed, `mean towers for wins - mean towers for losses`, as the test statistic. This statistic is appropriate because the question asks whether the average number of towers differs in a specific direction. I used a significance level of 0.05.

The observed difference was **6.40 towers**, and the p-value was **0.000** across 1,000 permutations. Since the p-value is below 0.05, I reject the null hypothesis. The data provides strong evidence that winning teams tend to destroy more towers than losing teams, though this is an observational result and does not prove that towers alone cause wins.

<iframe
  src="towers_hypothesis_permutation.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

## Framing a Prediction Problem

My prediction problem is to predict `result`, whether a team wins or loses a match, from team-level performance statistics. This is a **binary classification** problem because the response variable has two classes: win and loss.

I evaluated models using **accuracy**. Accuracy is appropriate here because the response classes are balanced by design: in each professional game, one team wins and one team loses. Since false positives and false negatives are similarly important for this project, accuracy is easier to interpret than precision or recall.

The time of prediction for this project is after team performance statistics for a game are available, but before using the official `result` column. This means the model should be interpreted as an in-game/stat-based outcome classifier rather than a pre-match forecast. I did not use future tournament information, team identities, or the `result` column itself as features.

## Baseline Model

The baseline model is a logistic regression classifier implemented in a single sklearn `Pipeline`. It uses two quantitative features from the original dataset:

- `kills`
- `towers`

There are **2 quantitative features**, **0 ordinal features**, and **0 nominal features**. Since both features are numeric, no categorical encoding was needed. The model was trained on 80% of the cleaned team-level rows and evaluated on the remaining 20%.

The baseline model achieved a test accuracy of **0.966**. This is already strong, which suggests that kills and towers contain a large amount of information about the outcome of a professional League of Legends match.

## Final Model

The final model uses a random forest classifier in a single sklearn `Pipeline`. In addition to `kills` and `towers`, I added `dragons`, `barons`, and `gamelength`, then engineered two new features:

- `kills_per_min`: kills divided by game length in minutes, which measures fighting pace while accounting for longer and shorter games.
- `objectives`: towers plus dragons plus barons, which summarizes major objective control.

These features are useful from the perspective of the data generating process because teams win by converting pressure into map objectives and by playing efficiently over the length of the game. A team with many kills in a short game may be more dominant than a team with the same number of kills in a much longer game, and a combined objective feature captures broad map control.

I tuned `max_depth` and `n_estimators` using `GridSearchCV` with 5-fold cross-validation. I tuned `max_depth` because deeper trees can capture more complex interactions but may overfit, and I tuned `n_estimators` because more trees can stabilize a random forest. The best hyperparameters were:

- `max_depth = 8`
- `n_estimators = 200`

The final model achieved a test accuracy of **0.970**, improving over the baseline accuracy of **0.966** on the same train-test split. The improvement is small, but it shows that adding objective and pace information helped the model generalize slightly better.

<iframe
  src="model_accuracy_comparison.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>

## Fairness Analysis

For the fairness analysis, I asked whether the final model performs differently for long games and short games. I defined:

- Group X: long games, where `gamelength` is at least the test-set median of 1854.5 seconds.
- Group Y: short games, where `gamelength` is below 1854.5 seconds.

The evaluation metric is accuracy, matching the metric used for the modeling task.

Null hypothesis: the model is fair across long and short games. Its accuracy for long games and short games is roughly the same, and any observed difference is due to random chance.

Alternative hypothesis: the model is not fair across long and short games. Its accuracy differs between long games and short games.

I used the absolute difference in accuracy between the two groups as the test statistic and a significance level of 0.05. The model's accuracy was **0.945** for long games and **0.996** for short games, giving an observed accuracy gap of **0.051**. The permutation-test p-value was **0.000**.

Since the p-value is below 0.05, I reject the null hypothesis. The final model appears to perform worse on long games than short games in this test set. This may be because long games are more strategically complex and can include more comeback opportunities, making the final result harder to classify from the selected summary statistics.

<iframe
  src="fairness_permutation.html"
  width="100%"
  height="520"
  frameborder="0"
></iframe>
