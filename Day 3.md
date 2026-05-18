# Day 3 Daily Log — Main Candidate Model Study

> [!NOTE]
> **Lab Hours**: `09:00` to `17:00` every working day

## Table of Contents

- [Day 3 Daily Log — Main Candidate Model Study](#day-3-daily-log--main-candidate-model-study)
  - [Table of Contents](#table-of-contents)
  - [Daily Plans and Logs](#daily-plans-and-logs)
    - [2026/05/13](#20260513)
  - [Study Notes](#study-notes)
    - [Boosting-Based Model Objective](#boosting-based-model-objective)
    - [XGBoost Regressor](#xgboost-regressor)
    - [LightGBM Regressor](#lightgbm-regressor)
    - [Neural and Sequence-Based Alternatives](#neural-and-sequence-based-alternatives)
      - [ANN / MLP](#ann--mlp)
      - [LSTM / GRU](#lstm--gru)
    - [Initial Technical Decision](#initial-technical-decision)
  - [References](#references)

---

## Daily Plans and Logs

### 2026/05/13

**Short-term Goal**

- [x] [Study main candidate algorithms for server thermal prediction](<replace-with-github-commit-url#boosting-based-model-objective>)

**Daily-logs**:

- `09:00–10:00`: [Boosting-Based Model Objective](<replace-with-github-commit-url#boosting-based-model-objective>)
- `10:00–11:45`: [XGBoost Regressor](<replace-with-github-commit-url#xgboost-regressor>)
- `13:00–14:30`: [LightGBM Regressor](<replace-with-github-commit-url#lightgbm-regressor>)
- `14:30–15:45`: [Neural and Sequence-Based Alternatives](<replace-with-github-commit-url#neural-and-sequence-based-alternatives>)
- `15:45–16:30`: [Initial Technical Decision](<replace-with-github-commit-url#initial-technical-decision>)

---

## Study Notes

### Boosting-Based Model Objective

After baseline models, the main candidate algorithms should be tree-based boosting models.

Reason:

- The gateway data is expected to be structured numerical sensor data.
- CPU usage and power consumption are tabular features.
- Lag and rolling features can convert time-series data into supervised tabular data.
- Boosting models are usually strong for this type of dataset.
- They are easier to validate than deep learning models.

Main candidates:

1. XGBoost Regressor
2. LightGBM Regressor

---

### XGBoost Regressor

XGBoost uses **gradient-boosted decision trees**.

Technical mechanism:

- Trees are trained sequentially.
- Each new tree tries to correct the error from previous trees.
- The final prediction is the sum of multiple weak learners.
- The objective function combines prediction loss and regularization.

Simplified idea:

```text
Final prediction = tree_1 + tree_2 + tree_3 + ... + tree_n
```

Why XGBoost is suitable:

- It performs well on structured data.
- It captures nonlinear relationships.
- It can learn interactions between CPU usage, power, and temperature.
- It supports feature importance analysis.
- It can work with medium-sized datasets.
- It is easier to implement than LSTM.

Important hyperparameters:

| Hyperparameter | Function |
|---|---|
| `n_estimators` | Number of boosting trees |
| `max_depth` | Controls tree complexity |
| `learning_rate` | Controls update step size |
| `subsample` | Uses part of the training samples for each tree |
| `colsample_bytree` | Uses part of the features for each tree |
| `reg_alpha`, `reg_lambda` | Regularization to reduce overfitting |

Technical risks:

- Can overfit if `max_depth` is too large.
- Needs tuning.
- Needs time-based validation.
- Needs good feature engineering for temporal behavior.

Decision:

- XGBoost should be the **main practical model** after baseline testing.

---

### LightGBM Regressor

LightGBM is also a gradient boosting decision tree model.

Technical mechanism:

- Uses boosting like XGBoost.
- Optimized for faster training.
- Efficient for larger tabular datasets.
- Can handle many rows from continuous IoT gateway logs.

Why LightGBM is suitable:

- The IoT gateway may generate large time-series sensor logs.
- Training speed can become important.
- It works well with lag features, rolling mean, rolling max, and trend features.
- It is suitable for repeated experiments and parameter tuning.

Important hyperparameters:

| Hyperparameter | Function |
|---|---|
| `num_leaves` | Controls tree complexity |
| `max_depth` | Limits tree depth |
| `learning_rate` | Controls update step size |
| `n_estimators` | Number of boosting rounds |
| `min_data_in_leaf` | Helps reduce overfitting |
| `feature_fraction` | Uses part of features per iteration |
| `bagging_fraction` | Uses part of data per iteration |

Technical risks:

- Can overfit small datasets if `num_leaves` is too large.
- Needs careful validation.
- Slightly less intuitive than Random Forest.

Decision:

- LightGBM should be compared with XGBoost as the **main model comparison**.

---

### Neural and Sequence-Based Alternatives

#### ANN / MLP

ANN can be used as an optional nonlinear model.

Technical mechanism:

- Receives numerical features.
- Passes the input through hidden layers.
- Learns nonlinear mapping from input features to temperature.

Possible advantage:

- Can model complex nonlinear behavior.
- Can combine CPU usage, power consumption, and temperature history.

Limitations:

- Needs more data than tree-based models.
- Requires feature scaling.
- Needs tuning of hidden layers, learning rate, batch size, epochs, and regularization.
- Less interpretable than XGBoost or LightGBM.

Decision:

- ANN is not the first priority.

#### LSTM / GRU

LSTM or GRU is useful for sequential data.

Technical mechanism:

- Input is a sequence window instead of a single row.
- The model receives several previous time steps.
- It can learn temporal dependency and thermal delay.

Example input shape:

```text
samples × time_window × features
```

Example setup:

```text
time_window = last 10 minutes
features = cpu_usage, power_consumption, temperature
target = temperature 5 minutes ahead
```

Possible advantage:

- Server temperature has time delay.
- Heat accumulation depends on previous states.
- LSTM/GRU can directly learn sequential patterns.

Limitations:

- Requires more data.
- More computationally expensive.
- More difficult to explain.
- May not outperform XGBoost or LightGBM if the dataset is small.

Decision:

- Keep LSTM/GRU as future work after tabular models.

---

### Initial Technical Decision

Recommended model sequence:

| Order | Model | Purpose |
|---|---|---|
| 1 | Linear Regression | Baseline benchmark |
| 2 | Random Forest | Nonlinear baseline |
| 3 | XGBoost | Main candidate |
| 4 | LightGBM | Main comparison |
| 5 | ANN / MLP | Optional nonlinear model |
| 6 | LSTM / GRU | Future sequential model |

Initial implementation setup:

```text
Input:
- CPU usage
- Power consumption
- Lag features
- Rolling statistics
- Temperature history, if available

Target:
- Temperature 5 minutes ahead
```

Main technical justification:

- Start with tabular regression before deep learning.
- XGBoost and LightGBM are strong because the data is structured sensor data.
- LSTM/GRU is promising but not efficient as the first implementation.
- The first milestone is to prove whether CPU usage and power consumption can predict future temperature with acceptable error.

---

## References

1. Lin, J., Lin, W., Huang, H., Lin, W., & Li, K. (2024). *Thermal Modeling and Thermal-Aware Energy Saving Methods for Cloud Data Centers: A Review*. IEEE Transactions on Sustainable Computing.
2. Chen, T., & Guestrin, C. (2016). *XGBoost: A Scalable Tree Boosting System*. Proceedings of the 22nd ACM SIGKDD International Conference on Knowledge Discovery and Data Mining.
3. Ke, G., Meng, Q., Finley, T., Wang, T., Chen, W., Ma, W., Ye, Q., & Liu, T. Y. (2017). *LightGBM: A Highly Efficient Gradient Boosting Decision Tree*. Advances in Neural Information Processing Systems.
4. Lin, J., Lin, W., Lin, W., Wang, J., & Jiang, H. (2022). *Thermal Prediction for Air-cooled Data Center using Data Driven-based Model*. Applied Thermal Engineering, 217, 119207.
5. Ilager, S., Ramamohanarao, K., & Buyya, R. (2020). *Thermal Prediction for Efficient Energy Management of Clouds using Machine Learning*. arXiv:2011.03649.
