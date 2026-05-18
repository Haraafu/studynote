# Day 4 Daily Log — Prediction Target and Dataset Requirement

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Table of Contents

- [Day 4 Daily Log — Prediction Target and Dataset Requirement](#day-4-daily-log--prediction-target-and-dataset-requirement)
  - [Table of Contents](#table-of-contents)
  - [Daily Plans and Logs](#daily-plans-and-logs)
    - [2026/05/14](#20260514)
  - [Study Notes](#study-notes)
    - [Prediction Target Options](#prediction-target-options)
    - [Selected Initial Prediction Target](#selected-initial-prediction-target)
    - [Input and Output Formulation](#input-and-output-formulation)
    - [Dataset Requirement](#dataset-requirement)
    - [Dataset Schema Design](#dataset-schema-design)
    - [Technical Notes for Implementation](#technical-notes-for-implementation)
    - [Clarification Questions for Supervisor](#clarification-questions-for-supervisor)
  - [References](#references)

---

## Daily Plans and Logs

### 2026/05/14

**Short-term Goal**

- [x] [Define prediction target and dataset requirement for ML-based server thermal prediction](<replace-with-github-commit-url#define-prediction-target-and-dataset-requirement-for-ml-based-server-thermal-prediction>)

**Daily-logs**:

- `09:00–10:00`: [Prediction Target Options](<replace-with-github-commit-url#prediction-target-options>)
- `10:00–11:00`: [Selected Initial Prediction Target](<replace-with-github-commit-url#selected-initial-prediction-target>)
- `11:00–12:00`: [Input and Output Formulation](<replace-with-github-commit-url#input-and-output-formulation>)
- `13:00–14:30`: [Dataset Requirement](<replace-with-github-commit-url#dataset-requirement>)
- `14:30–15:30`: [Dataset Schema Design](<replace-with-github-commit-url#dataset-schema-design>)
- `15:30–16:30`: [Technical Notes for Implementation](<replace-with-github-commit-url#technical-notes-for-implementation>)
- `16:30–17:00`: [Clarification Questions for Supervisor](<replace-with-github-commit-url#clarification-questions-for-supervisor>)

---

## Study Notes

### Prediction Target Options

The first technical decision is to define what the machine learning model should predict.

Possible prediction targets:

| Target | Type | Description | Technical Priority |
|---|---|---|---|
| `temperature_t+5min` | Regression | Predict server or rack temperature 5 minutes ahead | High |
| `temperature_t+10min` | Regression | Predict server or rack temperature 10 minutes ahead | Medium |
| `temperature_t+15min` | Regression | Predict server or rack temperature 15 minutes ahead | Medium to low |
| `temperature_trend` | Classification | Predict whether temperature will increase, decrease, or remain stable | Medium |
| `overheating_risk` | Classification | Classify condition into normal, warning, or critical | Medium |

Technical interpretation:

- Regression is more suitable for the first implementation because it gives a direct numerical temperature output.
- Classification can be developed later by converting the predicted temperature into risk categories.
- Short-horizon prediction is more realistic for early implementation because the model does not need to forecast too far into the future.
- The target must be matched with the available data frequency. For example, if the sampling interval is 1 minute, then a 5-minute prediction horizon means shifting the temperature label by 5 rows.

---

### Selected Initial Prediction Target

The selected initial target is:

```text
temperature_t+5min
```

This means the model will predict the server or rack temperature 5 minutes ahead using current and historical sensor values.

Justification:

- A 5-minute horizon is short enough to reduce forecasting uncertainty.
- It is still useful for early overheating warning.
- It can be evaluated clearly using regression metrics such as MAE, RMSE, and R².
- It can be extended later into 10-minute or 15-minute prediction after the first model is validated.
- It can also be converted into risk classification after numerical prediction works.

The initial task should be:

```text
Predict future temperature 5 minutes ahead using CPU usage and power consumption history.
```

Risk classification can be added after the regression model is stable.

Example rule for future classification:

```text
if predicted_temperature < warning_threshold:
    status = "normal"
elif predicted_temperature < critical_threshold:
    status = "warning"
else:
    status = "critical"
```

This approach is safer than building classification directly because the numerical temperature prediction provides more detailed information.

---

### Input and Output Formulation

The model input should be constructed from gateway sensor data and historical features.

Basic formulation:

```text
temperature_t+5min = f(cpu_usage_t, power_t, cpu_history, power_history, temperature_history)
```

Minimum input features:

| Feature | Description | Function |
|---|---|---|
| `timestamp` | Time of data collection | Maintains time order |
| `cpu_usage_t` | Current CPU usage | Represents workload intensity |
| `power_t` | Current power consumption | Represents heat generation potential |
| `cpu_lag_1` | Previous CPU usage | Captures recent workload state |
| `power_lag_1` | Previous power consumption | Captures recent power state |
| `temperature_t` | Current temperature | Represents current thermal condition |

Output variable:

| Target | Description |
|---|---|
| `target_temp_5min` | Temperature value 5 minutes after the current timestamp |

If the sampling interval is 1 minute:

```text
target_temp_5min = temperature shifted by -5 rows
```

Example:

```text
temperature at 09:05 becomes the target for input features at 09:00
```

If the sampling interval is 10 seconds, the label should be shifted by 30 rows to represent 5 minutes ahead.

```text
5 minutes = 300 seconds
300 seconds / 10 seconds = 30 rows
```

Therefore, the prediction horizon must be adjusted based on the real sampling interval.

---

### Dataset Requirement

The dataset must contain both input features and the target label.

Required variables:

| Column | Required Status | Role | Notes |
|---|---|---|---|
| `timestamp` | Required | Time index | Needed for time-series ordering |
| `server_id` | Recommended | Entity identifier | Needed if data comes from multiple servers |
| `rack_id` | Optional | Location identifier | Useful if rack-level prediction is needed |
| `cpu_usage` | Required | Input feature | Already monitored by the gateway |
| `power_consumption` | Required | Input feature | Already monitored by the gateway |
| `temperature` | Required | Target label | Must be confirmed |
| `fan_speed` | Optional | Input feature | Can improve thermal prediction |
| `ambient_temperature` | Optional | Input feature | Useful if room condition affects server temperature |
| `cooling_status` | Optional | Input feature | Useful if cooling behavior changes over time |

Most critical requirement:

```text
Historical temperature data is required to train a supervised temperature prediction model.
```

Technical reason:

- `cpu_usage` and `power_consumption` are input features.
- `temperature` is the label or target.
- Without historical temperature data, the model cannot learn the mapping between server activity and thermal response.

If temperature data is not available, there are two possible alternatives:

| Alternative | Description | Limitation |
|---|---|---|
| Add temperature sensor | Collect temperature data from rack, server inlet, or server internal sensor | Requires additional monitoring setup |
| Predict proxy risk | Use CPU and power thresholds to estimate potential heating risk | Not a true supervised temperature prediction model |

Recommended approach:

- Confirm whether temperature data is available from the gateway, server sensors, BMC/IPMI, or another monitoring system.
- If temperature exists, use it as the main label.
- If temperature does not exist, propose adding temperature monitoring before model training.

---

### Dataset Schema Design

The dataset should be structured as a time-series table.

Minimum raw dataset schema:

| timestamp | server_id | cpu_usage | power_consumption | temperature |
|---|---|---:|---:|---:|
| 2026-05-14 09:00 | server_01 | 62.0 | 210.0 | 38.2 |
| 2026-05-14 09:01 | server_01 | 68.0 | 225.0 | 38.5 |
| 2026-05-14 09:02 | server_01 | 71.0 | 235.0 | 38.7 |

After target shifting, the supervised ML dataset can be structured as:

| timestamp | server_id | cpu_usage | power_consumption | temperature | cpu_lag_1 | power_lag_1 | temp_lag_1 | target_temp_5min |
|---|---|---:|---:|---:|---:|---:|---:|---:|
| 2026-05-14 09:00 | server_01 | 62.0 | 210.0 | 38.2 | 60.0 | 205.0 | 38.0 | 39.0 |
| 2026-05-14 09:01 | server_01 | 68.0 | 225.0 | 38.5 | 62.0 | 210.0 | 38.2 | 39.4 |
| 2026-05-14 09:02 | server_01 | 71.0 | 235.0 | 38.7 | 68.0 | 225.0 | 38.5 | 39.8 |

Feature generation plan:

| Feature group | Example | Purpose |
|---|---|---|
| Current features | `cpu_usage`, `power_consumption`, `temperature` | Represents current system state |
| Lag features | `cpu_lag_1`, `power_lag_1`, `temp_lag_1` | Captures previous system state |
| Rolling mean | `cpu_mean_5min`, `power_mean_5min` | Captures short-term average condition |
| Rolling max | `cpu_max_5min`, `power_max_5min` | Captures recent workload or power peak |
| Difference features | `cpu_diff_1`, `power_diff_1`, `temp_diff_1` | Captures sudden changes |
| Time features | `hour`, `minute` | Captures repeated workload pattern |

---

### Technical Notes for Implementation

The first implementation should avoid random train-test split.

Recommended validation:

```text
Train set: earlier timestamp data
Test set: later timestamp data
```

Reason:

- This matches real prediction conditions.
- The model should be tested on future data, not randomly mixed data.
- Random splitting may cause data leakage in time-series data.

Recommended model setup:

| Stage | Model | Purpose |
|---|---|---|
| Baseline | Linear Regression | Check simple relationship |
| Nonlinear baseline | Random Forest | Capture nonlinear pattern |
| Main model | XGBoost | Main practical implementation |
| Main comparison | LightGBM | Fast boosting comparison |

Recommended evaluation metrics:

| Metric | Purpose |
|---|---|
| MAE | Measures average error in °C |
| RMSE | Penalizes large prediction errors |
| R² | Measures explained temperature variance |

Primary metric:

```text
MAE
```

Reason:

- It is easy to interpret.
- Example: `MAE = 1.5°C` means the model prediction is wrong by around 1.5°C on average.

Technical success condition:

```text
The model is useful if the prediction error is small enough for early overheating warning.
```

The exact acceptable error should be discussed with the supervisor because it depends on the operational safety threshold.

---

### Clarification Questions for Supervisor

Questions to confirm before implementation:

1. Does the IoT gateway collect temperature data, or only CPU usage and power consumption?
2. If temperature data is available, what type of temperature is recorded, such as CPU temperature, server inlet temperature, server outlet temperature, or rack temperature?
3. What is the sampling interval of the gateway data?
4. Is historical data already available for model training?
5. For the first prototype, should the output be numerical temperature prediction or overheating risk warning?

These questions are important because the model design depends on the available target label, data frequency, historical data availability, and expected prototype output.

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
3. Chen, T., & Guestrin, C. (2016). *XGBoost: A Scalable Tree Boosting System*. Proceedings of the 22nd ACM SIGKDD International Conference on Knowledge Discovery and Data Mining.
4. Ke, G., Meng, Q., Finley, T., Wang, T., Chen, W., Ma, W., Ye, Q., & Liu, T. Y. (2017). *LightGBM: A Highly Efficient Gradient Boosting Decision Tree*. Advances in Neural Information Processing Systems.
