# Server Thermal Prediction Study Notes and Daily Logs

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Daily Plans and Logs

### 2026/05/20

**Short-term Goal**

- [x] Design baseline experiment using Linear Regression and Random Forest

**Daily-logs**:

- `09:00–10:00`: Define Baseline Experiment Objective
- `10:00–11:30`: Prepare Baseline Dataset Configuration
- `13:00–14:00`: Design Linear Regression Experiment
- `14:00–15:00`: Design Random Forest Experiment
- `15:00–16:00`: Define Baseline Evaluation Procedure
- `16:00–17:00`: Summarize Day 6 Technical Decision

---

## Study Notes

## Design Baseline Experiment Using Linear Regression and Random Forest

### Define Baseline Experiment Objective

The objective of Day 6 is to design the first model experiment before using more complex algorithms.

The baseline experiment should answer three technical questions:

1. Can CPU usage and power consumption predict future server temperature?
2. Does adding time-series features improve the prediction result?
3. Is a nonlinear model better than a simple linear model?

The baseline models are:

| Model | Role |
|---|---|
| Linear Regression | Simple baseline |
| Random Forest Regressor | Nonlinear baseline |

Main prediction task:

```text
Predict temperature 5 minutes ahead.
```

Initial target:

```text
target_temp_5min
```

Initial model input:

```text
cpu_usage, power_consumption, lag features, rolling features, and temperature history if available
```

Technical reason:

- Linear Regression gives the simplest benchmark.
- Random Forest checks whether nonlinear relationships exist.
- These two models provide a fair starting point before testing XGBoost and LightGBM.

---

### Prepare Baseline Dataset Configuration

The dataset should be constructed as a time-series tabular dataset.

Minimum columns:

| Column | Role |
|---|---|
| `timestamp` | Time index |
| `server_id` | Server identifier |
| `cpu_usage` | Workload feature |
| `power_consumption` | Heat generation feature |
| `temperature` | Current temperature |
| `target_temp_5min` | Prediction target |

Example dataset structure:

| timestamp | server_id | cpu_usage | power_consumption | temperature | target_temp_5min |
|---|---|---:|---:|---:|---:|
| 2026-05-20 09:00 | server_1 | 61.5 | 218.4 | 38.2 | 39.1 |
| 2026-05-20 09:01 | server_1 | 66.0 | 225.8 | 38.5 | 39.4 |

The target is constructed by shifting the temperature column into the future.

If the sampling interval is 1 minute:

```text
target_temp_5min = temperature shifted by -5 rows
```

Example:

```text
feature row at 09:00 -> target temperature at 09:05
```

Technical note:

- Rows without future target values should be removed.
- Data must be sorted by `timestamp` before shifting.
- If multiple servers exist, target shifting must be applied per `server_id`.

---

### Design Linear Regression Experiment

Linear Regression is used to create the first performance benchmark.

#### Model formulation

```text
target_temp_5min = beta_0
                 + beta_1(cpu_usage)
                 + beta_2(power_consumption)
                 + beta_3(cpu_mean_5min)
                 + beta_4(power_mean_5min)
                 + beta_5(temp_lag_1)
                 + error
```

#### Experiment input

Recommended feature set:

| Feature | Purpose |
|---|---|
| `cpu_usage` | Current workload |
| `power_consumption` | Current heat generation |
| `cpu_lag_1` | Previous CPU usage |
| `power_lag_1` | Previous power consumption |
| `cpu_mean_5min` | Recent CPU workload average |
| `power_mean_5min` | Recent power average |
| `temperature_lag_1` | Previous thermal condition, if available |

#### Why Linear Regression is necessary

- It is easy to interpret.
- It checks the basic relationship between input features and target temperature.
- It gives a minimum performance standard.
- If Linear Regression already performs well, the problem may not require a complex model.
- If it performs poorly, nonlinear models are justified.

#### Expected limitation

Linear Regression may perform poorly because:

- Heat accumulation is delayed.
- Thermal behavior is not always linear.
- CPU and power interaction may affect temperature.
- Cooling conditions may change the relationship between power and temperature.

#### Output to record

| Output | Description |
|---|---|
| MAE | Average temperature prediction error |
| RMSE | Large-error sensitivity |
| R2 | Explained variance |
| Coefficients | Direction and strength of each feature |

Technical interpretation example:

```text
If beta_2 for power_consumption is positive, higher power consumption is associated with higher future temperature.
```

---

### Design Random Forest Experiment

Random Forest is used as the nonlinear baseline model.

#### Model mechanism

Random Forest builds multiple decision trees and averages the result.

```text
prediction = average(tree_1, tree_2, tree_3, ..., tree_n)
```

#### Experiment input

The feature set should be wider than Linear Regression.

Recommended feature groups:

| Feature group | Example |
|---|---|
| Current features | `cpu_usage`, `power_consumption`, `temperature` |
| Lag features | `cpu_lag_1`, `cpu_lag_3`, `cpu_lag_5` |
| Power lag features | `power_lag_1`, `power_lag_3`, `power_lag_5` |
| Temperature lag features | `temp_lag_1`, `temp_lag_3`, `temp_lag_5` |
| Rolling mean | `cpu_mean_5min`, `power_mean_5min` |
| Rolling max | `cpu_max_5min`, `power_max_5min` |
| Difference features | `cpu_diff_1`, `power_diff_1`, `temp_diff_1` |

#### Why Random Forest is relevant

- It can capture nonlinear relationships.
- It can model feature interaction.
- It does not require strong feature scaling.
- It is suitable for structured sensor data.
- It can provide feature importance for initial analysis.

Example nonlinear relationship:

```text
High CPU usage alone may not increase temperature significantly.
High CPU usage + high power consumption + high previous temperature may increase future temperature significantly.
```

#### Initial hyperparameter setup

| Hyperparameter | Initial value | Reason |
|---|---:|---|
| `n_estimators` | 100 | Enough trees for stable baseline |
| `max_depth` | None or 10 | Compare unrestricted and limited complexity |
| `min_samples_leaf` | 2 or 5 | Reduces overfitting |
| `random_state` | fixed value | Reproducibility |

#### Expected output

| Output | Description |
|---|---|
| MAE | Main comparison metric |
| RMSE | Large-error checking |
| R2 | Explained variance |
| Feature importance | Identifies influential features |

Technical interpretation example:

```text
If temperature_lag_1 has the highest importance, the previous thermal state is more predictive than current CPU usage alone.
```

---

### Define Baseline Evaluation Procedure

The evaluation must follow time-series logic.

Random train-test split should not be used.

Recommended split:

```text
Train: earliest 70%
Validation: next 15%
Test: latest 15%
```

Reason:

- The model should be tested on future unseen data.
- Random splitting may cause information leakage.
- Time-based splitting is closer to real deployment.

Evaluation metrics:

| Metric | Meaning | Role |
|---|---|---|
| MAE | Average absolute error | Main metric |
| RMSE | Square-root average squared error | Penalizes large errors |
| R2 | Explained variance | Measures overall fit |

Primary metric:

```text
MAE
```

Reason:

- It is directly interpretable in degree Celsius.
- It is easy to report to Prof. Ray.
- Example: MAE = 1.5 degree Celsius means the model is wrong by around 1.5 degree Celsius on average.

Comparison table format:

| Model | Feature set | MAE | RMSE | R2 | Notes |
|---|---|---:|---:|---:|---|
| Linear Regression | Basic features | TBD | TBD | TBD | Baseline |
| Linear Regression | Engineered features | TBD | TBD | TBD | Checks effect of feature engineering |
| Random Forest | Engineered features | TBD | TBD | TBD | Nonlinear baseline |

Expected analysis:

- If engineered features improve Linear Regression, thermal history is important.
- If Random Forest improves over Linear Regression, nonlinear behavior is present.
- If both models perform poorly, more variables may be needed, such as fan speed, ambient temperature, or rack position.

---

### Summarize Day 6 Technical Decision

Day 6 conclusion:

- The first experiment should compare Linear Regression and Random Forest.
- Linear Regression is used as the simplest benchmark.
- Random Forest is used to test nonlinear relationships.
- The first target should remain `temperature_t+5min`.
- Time-based train-validation-test splitting should be used.
- MAE should be the primary metric because it is easy to interpret.
- Feature engineering should be tested by comparing basic features and engineered features.

Final baseline experiment design:

```text
Experiment 1:
Model = Linear Regression
Features = cpu_usage + power_consumption
Target = temperature_t+5min

Experiment 2:
Model = Linear Regression
Features = cpu_usage + power_consumption + lag + rolling features
Target = temperature_t+5min

Experiment 3:
Model = Random Forest
Features = cpu_usage + power_consumption + lag + rolling + difference features
Target = temperature_t+5min
```

Next technical step:

```text
Design the main model experiment using XGBoost and LightGBM.
```

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
3. Breiman, L. (2001). *Random Forests*. Machine Learning, 45, 5-32.
4. Pedregosa, F., Varoquaux, G., Gramfort, A., Michel, V., Thirion, B., Grisel, O., Blondel, M., Prettenhofer, P., Weiss, R., Dubourg, V., Vanderplas, J., Passos, A., Cournapeau, D., Brucher, M., Perrot, M., & Duchesnay, E. (2011). *Scikit-learn: Machine Learning in Python*. Journal of Machine Learning Research, 12, 2825-2830.
