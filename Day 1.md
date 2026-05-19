# Day 1 Daily Log — Algorithm Candidate Extraction

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Table of Contents

- [Day 1 Daily Log — Algorithm Candidate Extraction](#day-1-daily-log--algorithm-candidate-extraction)
  - [Table of Contents](#table-of-contents)
  - [Daily Plans and Logs](#daily-plans-and-logs)
    - [2026/05/11](#20260511)
  - [Study Notes](#study-notes)
    - [Project Framing as ML-Based Thermal Prediction](#project-framing-as-ml-based-thermal-prediction)
    - [Extract Algorithm Candidates from the Review Paper](#extract-algorithm-candidates-from-the-review-paper)
    - [Technical Scope for Initial Implementation](#technical-scope-for-initial-implementation)
  - [References](#references)

---

## Daily Plans and Logs

### 2026/05/11

**Short-term Goal**

- [x] Extract candidate algorithms from the thermal modeling review paper

**Daily-logs**:

- `09:00–10:15`: Project Framing as ML-Based Thermal Prediction
- `10:15–12:00`: Extract Algorithm Candidates from the Review Paper
- `13:00–15:00`: Technical Scope for Initial Implementation
- `15:00–16:30`: Summarized relevant algorithm categories and excluded non-priority control algorithms.
---

## Study Notes

### Project Framing as ML-Based Thermal Prediction

The project can be framed as a **supervised time-series regression** problem.

Available IoT gateway variables:

- `timestamp`
- `cpu_usage`
- `power_consumption`

Required target variable:

- `temperature`

Initial prediction formulation:

```text
temperature_t+5min = f(cpu_usage_history, power_consumption_history, temperature_history)
```

Technical interpretation:

- `cpu_usage` represents workload intensity.
- `power_consumption` represents heat generation potential.
- `temperature` is required as the prediction label.
- Historical values are important because server temperature does not change instantly.
- The first task should be prediction, not automatic cooling control.

---

### Extract Algorithm Candidates from the Review Paper

The review paper groups thermal modeling methods into three categories.

| Category | Technical meaning | Relevance to this project |
|---|---|---|
| White-box model | Uses thermodynamic equations and physical parameters | Hard to implement first because it requires airflow, thermal resistance, cooling layout, and detailed physical assumptions |
| Black-box model | Uses ML or data-driven models | Most suitable because the gateway already collects sensor data |
| Gray-box model | Combines simplified physical model and ML | Useful later if rack layout, airflow, or cooling parameters are available |

Relevant algorithm candidates found from the paper:

- Linear Regression
- Support Vector Regression
- Gaussian Process Regression
- Artificial Neural Network
- XGBoost
- LightGBM
- LSTM

Technical conclusion from this extraction:

- The initial implementation should focus on **black-box regression models**.
- The project should first test whether CPU usage and power consumption can predict future temperature.
- Control methods such as MPC, PID, Q-learning, and reinforcement learning are not suitable as the first implementation because they require a validated prediction model or direct control environment.

---

### Technical Scope for Initial Implementation

The initial technical scope should be limited to temperature prediction.

Recommended first target:

```text
Predict temperature 5 minutes ahead.
```

Reason:

- Short prediction horizon is easier to validate.
- It is useful for early overheating warning.
- It avoids the complexity of long-term forecasting.
- It can be extended later to 10-minute or 15-minute prediction.

Initial feature groups to prepare:

| Feature group | Example | Purpose |
|---|---|---|
| Current features | `cpu_usage_t`, `power_t` | Represents current server condition |
| Lag features | `cpu_lag_1`, `power_lag_1` | Represents recent workload and power history |
| Rolling features | `cpu_mean_5min`, `power_max_5min` | Captures short-term trend and peak |
| Temperature history | `temp_lag_1`, `temp_lag_2` | Represents previous thermal state |
| Time features | `hour`, `minute` | Captures possible workload pattern by time |

Models to prioritize in the next study:

1. Linear Regression
2. Random Forest
3. XGBoost
4. LightGBM
5. ANN or LSTM only as later comparison

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
