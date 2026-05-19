# Server Thermal Prediction Study Notes and Daily Logs

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Daily Plans and Logs

### 2026/05/19

**Short-term Goal**

- [x] Design preprocessing and feature engineering strategy for server temperature prediction

**Daily-logs**:

- `09:00–10:00`: Review Dataset Requirement from Day 4
- `10:00–11:30`: Define Data Cleaning Strategy
- `13:00–14:30`: Design Time-Series Feature Engineering
- `14:30–15:30`: Define Target Construction for Multi-Horizon Prediction
- `15:30–16:30`: Prepare Validation Strategy for Time-Series Data
- `16:30–17:00`: Summarize Day 5 Technical Decision

---

## Study Notes

## Design Preprocessing and Feature Engineering Strategy for Server Temperature Prediction

### Review Dataset Requirement from Day 4

The expected dataset should be constructed from IoT gateway logs.

Minimum required columns:

| Column | Type | Role |
|---|---|---|
| `timestamp` | datetime | Time index for sequence construction |
| `server_id` | categorical/string | Identifies the monitored server |
| `cpu_usage` | numerical | Main workload feature |
| `power_consumption` | numerical | Main heat generation feature |
| `temperature` | numerical | Prediction label |

Optional but useful columns:

| Column | Role |
|---|---|
| `gpu_usage` | Additional workload indicator if the server uses GPU |
| `fan_speed` | Useful for explaining cooling response |
| `rack_id` | Useful if multiple servers are monitored |
| `ambient_temperature` | Helps separate server heat from room condition |
| `workload_type` | Helps explain workload-specific thermal behavior |

Technical assumption:

```text
CPU usage and power consumption are input features.
Temperature is the supervised learning target.
```

Main issue:

- If temperature is not available, the model cannot be trained as supervised temperature prediction.
- If only CPU usage and power consumption are available, the first task should be limited to data exploration or proxy risk scoring.

---

### Define Data Cleaning Strategy

The raw IoT data should not be directly used for model training.

#### 1. Timestamp alignment

Sensor data must have consistent time intervals.

Example target interval:

```text
1 row per minute
```

If logs arrive irregularly, resampling is needed.

Example:

```text
09:00:05 → 09:00
09:01:12 → 09:01
09:02:48 → 09:02
```

Reason:

- Time-series models and lag features require fixed intervals.
- Irregular timestamps can make previous values inconsistent.

#### 2. Missing value handling

Possible missing data cases:

| Missing column | Suggested handling |
|---|---|
| `cpu_usage` | Forward fill if gap is short; remove row if gap is long |
| `power_consumption` | Interpolate if sensor gap is short |
| `temperature` | Do not blindly fill if used as target |
| `timestamp` | Remove invalid rows |

Recommended rule:

```text
If missing gap <= 2 minutes: interpolate or forward fill.
If missing gap > 2 minutes: mark as invalid segment.
```

Reason:

- Short gaps may be caused by sensor delay.
- Long gaps can break the time-series structure.

#### 3. Outlier detection

Possible abnormal values:

| Feature | Invalid example |
|---|---|
| `cpu_usage` | `< 0%` or `> 100%` |
| `power_consumption` | negative value |
| `temperature` | physically impossible value |
| `timestamp` | duplicated or unordered time |

Basic rule:

```text
0 <= cpu_usage <= 100
power_consumption >= 0
temperature should be within realistic server/rack range
```

Technical note:

- Do not remove all high temperature values.
- High temperature may represent important overheating behavior.
- Only remove values that are physically impossible or clearly caused by sensor error.

#### 4. Duplicate record handling

If multiple rows have the same timestamp and server ID:

```text
group by timestamp and server_id
use mean for numerical columns
```

Reason:

- Duplicate readings can bias the model.
- Aggregation keeps one consistent reading per time step.

---

### Design Time-Series Feature Engineering

Raw features are not enough because temperature has thermal delay.

The model should use historical features.

#### 1. Lag features

Lag features represent previous sensor values.

Example:

| Feature | Meaning |
|---|---|
| `cpu_lag_1` | CPU usage 1 minute before |
| `cpu_lag_5` | CPU usage 5 minutes before |
| `power_lag_1` | Power consumption 1 minute before |
| `power_lag_5` | Power consumption 5 minutes before |
| `temp_lag_1` | Temperature 1 minute before |
| `temp_lag_5` | Temperature 5 minutes before |

Technical justification:

- Temperature does not respond instantly to CPU or power changes.
- Previous workload and power values may explain future temperature better than current values only.

#### 2. Rolling statistics

Rolling statistics summarize recent behavior.

Recommended rolling windows:

```text
3 minutes, 5 minutes, 10 minutes
```

Example features:

| Feature | Meaning |
|---|---|
| `cpu_mean_5min` | Average CPU usage in the last 5 minutes |
| `cpu_max_5min` | Maximum CPU usage in the last 5 minutes |
| `power_mean_5min` | Average power in the last 5 minutes |
| `power_max_5min` | Maximum power in the last 5 minutes |
| `temp_mean_5min` | Average temperature in the last 5 minutes |

Technical justification:

- Rolling mean captures stable workload pressure.
- Rolling max captures recent workload spikes.
- Rolling temperature captures thermal accumulation.

#### 3. Difference features

Difference features capture change direction.

Example:

```text
cpu_diff_1 = cpu_usage_t - cpu_usage_t-1
power_diff_1 = power_t - power_t-1
temp_diff_1 = temperature_t - temperature_t-1
```

Why this matters:

- Sudden CPU increase may indicate upcoming temperature rise.
- Sudden power increase may indicate higher heat generation.
- Temperature difference indicates whether the server is heating or cooling.

#### 4. Time features

Possible time-based features:

| Feature | Purpose |
|---|---|
| `hour` | Captures daily workload pattern |
| `minute` | Helps sequence representation |
| `day_of_week` | Useful if weekday workload differs |
| `is_working_hour` | Marks active working period |

Technical note:

- Time features are useful if workload follows daily patterns.
- They may be less useful if the server workload is random or experimental.

---

### Define Target Construction for Multi-Horizon Prediction

The target should be shifted into the future.

Recommended target:

```text
temperature_t+5min
```

Possible future targets:

| Target | Meaning | Difficulty |
|---|---|---|
| `temperature_t+5min` | Predict temperature 5 minutes ahead | Low |
| `temperature_t+10min` | Predict temperature 10 minutes ahead | Medium |
| `temperature_t+15min` | Predict temperature 15 minutes ahead | Higher |

Example target construction:

```text
target_temp_5min = temperature shifted by -5 rows
```

If the data interval is 1 minute:

```text
row at 09:00 uses target temperature at 09:05
```

Technical justification:

- 5-minute prediction is suitable for the first experiment.
- It gives early warning while keeping the prediction horizon realistic.
- Longer horizons should be tested only after the 5-minute model works.

Optional classification target:

| Class | Rule example |
|---|---|
| Normal | temperature < warning threshold |
| Warning | temperature close to threshold |
| Critical | temperature exceeds threshold |

Technical note:

- Classification should be built after regression.
- Regression gives more detailed output.
- Risk category can be derived from predicted temperature.

---

### Prepare Validation Strategy for Time-Series Data

The model should not use random train-test split.

Recommended split:

```text
Train: earliest 70%
Validation: next 15%
Test: latest 15%
```

Reason:

- The model should be evaluated on future unseen data.
- Random splitting may leak future patterns into training.
- Time-based splitting is more realistic for deployment.

Example:

```text
May 1–10   → training data
May 11–12  → validation data
May 13–14  → test data
```

Evaluation metrics:

| Metric | Function |
|---|---|
| MAE | Average prediction error in °C |
| RMSE | Penalizes large errors |
| R² | Measures explained variance |

Primary metric:

```text
MAE
```

Reason:

- It is easy to explain.
- It directly shows average error in temperature units.
- Example: MAE = 1.5°C means the model is wrong by about 1.5°C on average.

---

### Summarize Day 5 Technical Decision

Day 5 conclusion:

- The raw gateway logs need preprocessing before training.
- Timestamp alignment is necessary for time-series modeling.
- Missing values should be handled based on gap length.
- Physically impossible values should be removed.
- High temperature values should not be removed automatically because they may represent real overheating events.
- Lag features and rolling statistics are necessary to model thermal delay.
- The first target should be `temperature_t+5min`.
- The dataset should use time-based train-validation-test splitting.
- The next step is to design the baseline experiment using Linear Regression and Random Forest.

Final technical direction:

```text
Build a clean time-series tabular dataset first.
Then train baseline models using engineered lag, rolling, and difference features.
```

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
3. Lin, J., Lin, W., Lin, W., Wang, J., & Jiang, H. (2022). *Thermal Prediction for Air-cooled Data Center using Data Driven-based Model*. Applied Thermal Engineering, 217, 119207.
