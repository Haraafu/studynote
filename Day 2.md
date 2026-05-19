# Day 2 Daily Log — Baseline Model Study

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Table of Contents

- [Day 2 Daily Log — Baseline Model Study](#day-2-daily-log--baseline-model-study)
  - [Table of Contents](#table-of-contents)
  - [Daily Plans and Logs](#daily-plans-and-logs)
    - [2026/05/12](#20260512)
  - [Study Notes](#study-notes)
    - [Baseline Modeling Objective](#baseline-modeling-objective)
    - [Linear Regression as Baseline](#linear-regression-as-baseline)
    - [Random Forest as Nonlinear Baseline](#random-forest-as-nonlinear-baseline)
    - [Feature Engineering for Baseline Models](#feature-engineering-for-baseline-models)
    - [Evaluation Setup](#evaluation-setup)
  - [References](#references)

---

## Daily Plans and Logs

### 2026/05/12

**Short-term Goal**

- [x] Study baseline models for server temperature prediction

**Daily-logs**:

- `09:00–10:00`: Baseline Modeling Objective
- `10:00–11:30`: Linear Regression as Baseline
- `13:00–14:30`: Random Forest as Nonlinear Baseline
- `14:30–15:45`: Feature Engineering for Baseline Models
- `15:45–16:30`: Evaluation Setup

---

## Study Notes

### Baseline Modeling Objective

The baseline model is needed before implementing more complex algorithms.

Main purpose:

- Check whether the available variables are predictive.
- Create a minimum performance benchmark.
- Avoid using complex models before confirming the data quality.
- Provide interpretable early results for discussion with Prof. Ray.

Initial supervised learning setup:

```text
Input:
- cpu_usage
- power_consumption
- lag features
- rolling features
- temperature history, if available

Target:
- temperature 5 minutes ahead
```

---

### Linear Regression as Baseline

Linear Regression assumes that future temperature can be predicted as a weighted sum of input features.

Example formulation:

```text
temperature_t+5 = β0
                + β1(cpu_usage_t)
                + β2(power_t)
                + β3(cpu_rolling_mean_5min)
                + β4(power_rolling_mean_5min)
                + ε
```

Technical justification:

- It is simple and fast.
- It can show whether CPU usage and power consumption have a direct linear relationship with temperature.
- It is easy to explain because each coefficient represents the direction and strength of a feature.
- It gives a benchmark for more advanced models.

Expected limitation:

- Server temperature is usually nonlinear.
- Thermal behavior has delay.
- Same CPU usage can produce different temperatures depending on previous temperature, airflow, cooling state, and server condition.
- Linear Regression cannot automatically capture complex feature interactions.

Decision:

- Use Linear Regression as the **first baseline only**.
- Do not treat it as the final expected model.

---

### Random Forest as Nonlinear Baseline

Random Forest is a stronger baseline because it can model nonlinear relationships.

Technical mechanism:

- Builds many decision trees.
- Each tree learns from different subsets of the data.
- Final output is the average prediction of all trees.

Why it is technically useful:

- It can capture nonlinear relations between CPU usage, power, and temperature.
- It can learn feature interactions.
- It is robust for tabular sensor data.
- It does not require heavy feature scaling.

Example interaction:

```text
high CPU usage + high power + high previous temperature
→ higher predicted future temperature
```

Possible useful features:

| Feature | Purpose |
|---|---|
| `cpu_usage_t` | Current workload condition |
| `power_t` | Current heat generation indicator |
| `cpu_lag_1`, `cpu_lag_2` | Previous workload condition |
| `power_lag_1`, `power_lag_2` | Previous power condition |
| `temp_lag_1` | Previous thermal state |
| `cpu_rolling_mean_5min` | Short-term workload trend |
| `power_rolling_max_5min` | Recent power peak |

Limitations:

- Random Forest does not naturally understand time order.
- Time-series pattern must be represented manually through lag and rolling features.
- It may not extrapolate well outside the training data range.

Decision:

- Use Random Forest as the **nonlinear baseline** before XGBoost and LightGBM.

---

### Feature Engineering for Baseline Models

Raw data should not only be used as direct `cpu_usage` and `power_consumption`.

Required engineered features:

| Feature group | Example features | Technical purpose |
|---|---|---|
| Lag features | `cpu_lag_1`, `cpu_lag_2`, `power_lag_1`, `power_lag_2` | Captures previous condition |
| Rolling mean | `cpu_mean_5min`, `power_mean_5min` | Captures short-term average |
| Rolling max | `cpu_max_5min`, `power_max_5min` | Captures recent peak |
| Difference features | `cpu_t - cpu_t-1`, `power_t - power_t-1` | Captures sudden increase or decrease |
| Temperature history | `temp_lag_1`, `temp_lag_2` | Captures thermal inertia |
| Time features | `hour`, `minute` | Captures repeated workload pattern |

Important note:

- Temperature history is likely one of the strongest features.
- If temperature data is not available, the model can still be tested, but prediction quality may be limited.

---

### Evaluation Setup

Regression metrics should be used for temperature prediction.

| Metric | Technical meaning | Reason for use |
|---|---|---|
| MAE | Average absolute error | Easy to interpret in °C |
| RMSE | Penalizes large errors | Useful for detecting large overheating prediction mistakes |
| R² | Explained variance | Shows how much temperature variation is captured |

Primary metric:

```text
MAE
```

Reason:

- MAE is easy to explain.
- Example: `MAE = 1.5°C` means the model is wrong by around 1.5°C on average.

Validation strategy:

- Use time-based split, not random split.
- Training data should come from earlier timestamps.
- Test data should come from later timestamps.
- This prevents future data leakage.

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Breiman, L. (2001). *Random Forests*. Machine Learning.
3. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
