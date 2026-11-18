# Baseline: KNIME results (university project)

This is the human baseline that the agents in this repository are compared against. It comes from a 3-person team project for the course *Aprendizagem e Decisão Inteligentes* (Intelligent Learning and Decision-Making), University of Minho, built in KNIME following CRISP-DM (phases 1–5, no deployment). All random seeds were set to **2026**.

There are two datasets and three tasks:

| Task | Dataset | Type | Target | Best model | Main metric |
|---|---|---|---|---|---|
| A | Bank Marketing | Binary classification | `y` (yes/no) | Gradient Boosted Trees | Cohen's κ = **0.423** |
| B1 | Energy production | Multiclass classification | `Productions` (5 bins, rebuilt) | Random Forest | Accuracy **68.18%**, κ = **0.602** |
| B2 | Energy production | Regression | `Distribution Network (kWh)` | Random Forest | R² = **0.802** |

---

## Task A: Bank Marketing (classification)

**Data.** UCI Bank Marketing (Portuguese bank telemarketing campaigns): about 41k rows and 21 columns (client data, credit history, campaign context, socio-economic indicators). The goal is to predict whether a client subscribes to a term deposit.

### Traps found during exploration

1. **Data leakage: `duration`.** The call duration is strongly related to the outcome (calls of 0 seconds are always "no"), but it is only known *after* the call. It was **removed** from the pipeline.
2. **Class imbalance.** More than 88% of the rows are "no". Accuracy alone is misleading, so **Cohen's κ** and **recall on "yes"** (`yes (Actual) Accuracy`) were used to select models.
3. **Hidden missing values.** There are no nulls, but `job`, `marital`, `education`, `default` and other columns contain `"unknown"`. These values were converted to missing.
4. **`pdays` sentinel.** About 96% of the rows hold `999` ("never contacted before"). The values 0–20 look like outliers but are real and informative, so they were kept.

### Preparation decisions

Each decision was tested with a Decision Tree on a stratified 80/20 split.

| Experiment | Overall acc. | yes recall | κ |
|---|---|---|---|
| No normalization | 87.42% | 28.70% | 0.268 |
| With normalization | 87.41% | 28.59% | 0.267 |
| Missing: do nothing | 87.42% | 28.70% | 0.268 |
| Missing: most frequent | 87.49% | 28.06% | 0.264 |
| Missing: next value | 87.15% | 27.37% | 0.250 |
| **Missing: previous value** (chosen) | 87.49% | 29.29% | 0.271 |
| Missing: remove row | 86.00% | 33.07% | 0.292 |

- **No normalization:** it did not help tree models.
- **"Remove row" was rejected** even though it scored best. It dropped about 26% of the data, and the score went up only because removing those rows also reduced the class imbalance. "Previous value" was chosen instead.
- **SMOTE**, applied **to the training partition only**, so the test set keeps the real distribution. With a Decision Tree, recall on "yes" went from 29.29% to 38.01% and κ stayed the same (0.271).

### Modelling

**Model comparison** (stratified 80/20, with SMOTE):

| Model | Overall acc. | yes recall | κ |
|---|---|---|---|
| Decision Tree | 89.84% | 38.01% | 0.271 |
| Random Forest | 89.29% | 39.55% | 0.396 |
| **Gradient Boosted Trees** | 86.97% | 54.09% | **0.410** |

**Partitioning:** the 70/75/80/90% splits and 5- and 10-fold cross-validation all landed at κ between 0.397 and 0.420. The κ = 0.420 came from a stratified 70/30 split; 10-fold stratified CV gave κ = 0.418 with the best "yes" recall (55.91%).

**Feature Selection Loop**, optimizing κ, without CV or SMOTE inside the loop: κ = 0.382, worse than using all columns. It was dropped.

**Hyperparameter tuning (GBT):**

| Depth | Models | LR | Overall acc. | yes recall | κ |
|---|---|---|---|---|---|
| 4 | 100 | 0.1 | 86.95% | **55.91%** | 0.418 |
| 6 | 100 | 0.1 | 87.90% | 51.81% | **0.423** |
| 4 | 300 | 0.05 | 87.56% | 53.64% | 0.422 |
| 4 | 200 | 0.1 | 87.81% | 52.44% | **0.423** |

**Baseline to beat:** κ ≈ **0.42**, "yes" recall ≈ **52–56%**.

---

## Task B: Energy production (classification and regression)

**Data.** Dataset provided by the course teaching staff: 8,784 hourly rows and 24 columns. It contains energy production by source (cogeneration, wind, solar, hydro, other), the total injected into the distribution network, and weather variables (temperature, humidity, rain, cloud cover, wind speed and direction at 10 m and 100 m, radiation).

### Traps found during exploration

1. **Data leakage: production components.** `Distribution Network (kWh)` is the exact **sum** of the individual sources. Keeping them as features gives R² = 1, a useless model, because production is only known after it happens. **All production columns were removed**, so the models only use weather and time features, which simulates real forecasting.
2. **Corrupted categorical target.** The original `Productions` labels were inconsistent with the actual kWh values ("Medium" had a higher median than "High", and the ranges overlapped heavily), and they also had typos (e.g. "Alto"). The target was **discarded and rebuilt** by equal-frequency binning of `Distribution Network (kWh)` into 5 classes: Low, Medium-Low, Medium, Medium-High, High.
3. **Fake missing values.** `precipitation` and `rain` had more than 79% missing values, and their minimum was 0.1. Missing meant "no rain", so they were filled with **0.0**.
4. **Type errors.** `Cogeneration`, `Photovoltaics` and `diffuse_radiation` were read as strings and were converted to numbers.
5. **Invalid timestamps.** 6 rows (0.07%) had impossible dates and were removed.

### Preparation decisions

- **Feature engineering on `Date/Time`:** year, month, day, weekday and **hour**. Hour matters because solar production follows a clear daily cycle that peaks between 12h and 14h.
- **Recovering missing total production:** when `Distribution Network` was missing, it was filled with the sum of its components (deterministic, not an estimate).
- **Missing values**, after sorting chronologically: linear interpolation for continuous weather variables, previous value for categorical ones (wind direction), and 0.0 for rain and precipitation.
- **Dropped columns:** `RowID` and the original `Productions`.
- **Redundancy noted:** wind speed at 10 m and at 100 m are highly correlated.

### Modelling

**Baseline comparison** (80/20):

| Classification | Accuracy | κ |
|---|---|---|
| Decision Tree | 55.54% | 0.444 |
| **Random Forest** | 66.40% | 0.580 |
| Gradient Boosted Trees | 60.31% | 0.504 |

| Regression | R² | MAE | RMSE |
|---|---|---|---|
| **Random Forest** | 0.793 | 94,942 | 127,363 |
| Gradient Boosted Trees | 0.750 | 106,820 | 139,892 |
| Linear Regression | 0.589 | 140,123 | 179,427 |

**Tuning the Random Forest** with 10-fold CV (`~` = KNIME default; NS = minimum node size):

| Setup | Accuracy | κ | R² | MAE | RMSE |
|---|---|---|---|---|---|
| 80/20 split | 66.40% | 0.580 | 0.793 | 94,942 | 127,363 |
| 10-fold CV | 67.55% | 0.594 | 0.800 | 95,058 | 126,103 |
| **300 models, depth ~, NS ~** | **68.18%** | **0.602** | **0.802** | **94,733** | **125,489** |
| 300 models, depth 15, NS ~ | 57.61% | 0.470 | 0.768 | 105,171 | 135,831 |
| 300 models, depth 15, NS 5 | 58.16% | 0.477 | 0.749 | 110,013 | 141,370 |

**Baseline to beat:** R² = **0.802**, classification accuracy **68.18%**, κ = **0.602**.

---

## Known limitations of the baseline

Written in hindsight. These are good checks for the reviewer agent:

- **Random split on time-series data (Task B).** The energy data is hourly, and random partitioning and CV put neighbouring hours in both train and test. This probably makes the results optimistic. A chronological split, e.g. training on the first 10 months and testing on the last 2, would be more realistic.
- **Preprocessing before the split.** Interpolation, previous-value imputation and equal-frequency binning were computed on the full dataset before partitioning. This is a mild form of leakage.
- **No separate validation set for tuning.** The hyperparameters were chosen on the same data used to report results, so the final numbers are slightly optimistic.

## How agents will be compared

For each task, the agent gets the raw dataset and the business goal, with **no hints**. It is evaluated on:

1. Whether it **finds the traps on its own**: `duration` leakage, the production-sum leakage, the corrupted `Productions` labels, the `"unknown"` values, and rain/precipitation missing meaning zero.
2. Whether it **chooses suitable metrics** (κ and minority-class recall for Task A, not plain accuracy).
3. The **final metrics** compared with the tables above, using the same seed and split where possible, plus a chronological split for Task B.
4. **Cost and time per run.**