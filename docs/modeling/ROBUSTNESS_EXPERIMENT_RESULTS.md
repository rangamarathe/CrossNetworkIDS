# Feature-ablation robustness experiment results

## Objective

This experiment tests whether high-drift or laboratory-context features cause a binary intrusion-detection model to perform well in its source environment but transfer poorly to another network dataset.

The experiment uses the same mixed 2017/2018 training records, preprocessing, Logistic Regression configuration, and classification threshold for every run. Only the predictor set changes. Each model is evaluated on untouched chronological test partitions from both years.

## Feature sets

| Feature set | Purpose |
|---|---|
| All shared numeric features | Establish the reference result using all 83 predictors. |
| Without high-KS features | Test whether removing distributionally unstable features improves transfer. |
| Without laboratory-context features | Test reliance on ports, protocol, TCP-window values, and flag fields that may identify the test environment. |
| Without both groups | Test a deliberately drift-conscious predictor set. |

## Results

| Feature set | Test dataset | Features | Macro F1 | Attack recall | Attack F1 | False-alarm rate |
|---|---|---:|---:|---:|---:|---:|
| All features | 2017 | 83 | 0.963 | 0.986 | 0.943 | 2.98% |
| All features | 2018 | 83 | 0.968 | 0.978 | 0.964 | 3.92% |
| Without high-KS | 2017 | 83 | 0.963 | 0.986 | 0.943 | 2.98% |
| Without high-KS | 2018 | 83 | 0.968 | 0.978 | 0.964 | 3.92% |
| Without laboratory context | 2017 | 67 | 0.945 | 0.985 | 0.916 | 4.62% |
| Without laboratory context | 2018 | 67 | 0.943 | 0.957 | 0.937 | 6.62% |
| Without both groups | 2017 | 67 | 0.945 | 0.985 | 0.916 | 4.62% |
| Without both groups | 2018 | 67 | 0.943 | 0.957 | 0.937 | 6.62% |

## Findings

### 1. All-feature model

The model using all 83 shared numeric features produced the strongest results on both test datasets. Its macro F1 was 0.963 on 2017 and 0.968 on 2018, with attack recall above 0.97 in both environments.

### 2. Laboratory-context ablation

Removing 16 laboratory-context features did not improve the mixed-training Logistic Regression model:

- 2017 macro F1 declined from 0.963 to 0.945.
- 2018 macro F1 declined from 0.968 to 0.943.
- 2018 attack recall declined from 0.978 to 0.957.
- The 2017 false-alarm rate increased from 2.98% to 4.62%.
- The 2018 false-alarm rate increased from 3.92% to 6.62%.

These features contain environment information, but they also provide useful predictive information after the model has been trained on both environments. Their removal creates a measurable operational cost, particularly through higher false-alarm rates.

This finding does not prove that the features are universally safe. Both evaluation environments are represented in the mixed training data. Evaluation on a third, completely unseen network is still required to test broader generalization.

### 3. Current high-KS ablation

No feature met the current selection rule of:

```text
mean of benign KS and attack KS >= 0.50
```

Consequently, the `Without high-KS` feature set retained all 83 features and produced exactly the same results as the all-feature model. The `Without both` set was therefore also identical to the laboratory-context ablation.

The current threshold is too conservative because averaging benign and attack KS can conceal substantial attack-specific drift. For example:

| Feature | Benign KS | Attack KS | Mean KS |
|---|---:|---:|---:|
| `Fwd IAT Min` | 0.231 | 0.697 | 0.464 |
| `Total Length of Bwd Packet` | 0.130 | 0.659 | 0.395 |
| `Packet Length Std` | 0.121 | 0.655 | 0.388 |
| `Bwd Packet Length Max` | 0.119 | 0.653 | 0.383 |

Although these attack distributions differ substantially between years, their lower benign KS values bring the average below 0.50.

## Interpretation

The current evidence supports retaining all features for this mixed-data Logistic Regression configuration. Removing the full laboratory-context group reduced detection quality and increased false alarms.

However, the experiment has not yet established whether particular high-drift features create brittleness for individual attack families. Binary attack-versus-benign aggregation can combine attacks with very different behaviors and conceal family-specific drift.

The correct conclusion is therefore:

> Mixed-environment training with all features produced the strongest two-dataset results. Broad removal of laboratory-context features was harmful, while the current mean-KS rule selected no features and requires refinement before conclusions can be drawn about drift-based ablation.

## Recommended next experiment

The next experiment should calculate cross-year KS statistics separately for every compatible attack family and then repeat the ablation comparison.

Recommended steps:

1. Align attack-family names between CICIDS2017 and CSE-CIC-IDS2018.
2. Calculate each feature's 2017-versus-2018 KS statistic separately for every compatible family using training data only.
3. Define a feature-selection rule before evaluating the test data, such as attack-family KS of at least 0.50 for multiple families.
4. Create a focused high-drift feature group rather than removing every feature with one isolated high value.
5. Retrain the same controlled models with and without that group.
6. Compare macro F1, attack recall, per-family recall, and false-alarm rate on both held-out years.
7. Prefer a reduced feature set only if it improves worst-year or per-family transfer without creating an unacceptable false-alarm increase.
8. Validate the selected model on a third unseen network before making a production-generalization claim.

An attack-only rule such as `attack KS >= 0.50` can be used as a preliminary sensitivity analysis, but per-family KS is preferable because it preserves differences among DoS, brute-force, infiltration, scanning, and other attack behaviors.

## Experiment artifacts

- `robustness-validation/07_feature_ablation_robustness.ipynb` — executed experiment and visualizations.
- `robustness-validation/results/07_feature_ablation_results.csv` — evaluation metrics for each feature set and test dataset.
- `robustness-validation/results/07_training_only_ks_statistics.csv` — training-only binary-class KS statistics.
