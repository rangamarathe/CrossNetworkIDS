# Model tuning strategy

## Untuned versus tuned models

| Aspect | Untuned model | Tuned model |
|---|---|---|
| Configuration | Uses one predefined configuration | Compares multiple hyperparameter configurations |
| Purpose | Establish a reproducible reference result | Test whether configuration changes improve the reference |
| Parameter selection | Selected before model comparison | Selected using validation performance |
| Training cost | Lower | Higher because several candidates are trained |
| Validation set | Not required for parameter selection | Required to select the winning candidate |
| Test-set use | Evaluates the fixed model | Evaluates only the selected and refitted winner |
| Main benefit | Simple, interpretable benchmark | Potentially better generalization |
| Main risk | Configuration may be suboptimal | Search may overfit the validation period |
| Project rule | Retained as the baseline | Replaces the baseline only if protected test results improve |
| Expected outcome | Provides a comparison point | Improvement is possible but not guaranteed |

An untuned model is not necessarily unconfigured. Its settings are chosen before comparing alternatives. For example, the Logistic Regression baseline uses L2 regularization, balanced class weights, and `C=1.0`, but those settings are fixed in advance rather than selected through a validation search.

## General tuning workflow

| Stage | Data period | Data used | Purpose |
|---:|---|---|---|
| 1. Search training | Earliest 60% or 70% | Class-aware sample | Fit candidate configurations efficiently |
| 2. Validation | Next 20% or 10% | Complete validation partition | Select the candidate with the highest macro F1 |
| 3. Final refit | Complete first 80% | Usually all available rows | Retrain the selected configuration with more historical data |
| 4. Final test | Latest 20% | Complete untouched test partition | Obtain the final protected performance estimate |
| 5. Comparison | Final test results | Tuned versus untuned | Determine whether tuning produced a meaningful improvement |

## Chronological design by notebook

| Notebook | Search training | Validation | Final refit | Final test |
|---|---|---|---|---|
| `advanced-modeling/02_logistic_regression_tuning.ipynb` | First 70%, capped at 50,000 rows per class | Next 10% | Complete first 80% | Latest 20% |
| `advanced-modeling/04_histgb_fine_tuning.ipynb` | First 60%, sampled at 2,000–30,000 rows per class | Next 20% | Complete first 80% | Latest 20% |
| `advanced-modeling/05_all_model_tuning.ipynb`: tree models | First 60%, sampled at 2,000–20,000 rows per class | Next 20% | Complete first 80% | Latest 20% |
| `advanced-modeling/05_all_model_tuning.ipynb`: MLP | First 60%, sampled at 2,000–20,000 rows per class | Next 20% | Sample of 5,000–50,000 rows per class from first 80% | Latest 20% |

Chronological periods are used because a random tuning split can place flows from the same scheduled attack burst into both training and validation. The chronological design asks whether a configuration trained on earlier traffic performs well on later traffic.

## Purpose of each partition

| Partition | What it does | Why it is separate |
|---|---|---|
| Search-training data | Learns model coefficients, tree rules or neural-network weights | Candidate configurations need data on which to fit |
| Validation data | Compares hyperparameter configurations | Prevents candidate selection from using the final test set |
| Final-refit data | Trains the winning configuration on more historical data | Uses the complete pre-test period after selection |
| Final test data | Measures the selected model's performance | Remains untouched until every tuning decision is complete |

## Tuning procedure

| Step | Action | Leakage control |
|---:|---|---|
| 1 | Rank observations chronologically within each class | Preserves later observations for validation and testing |
| 2 | Reserve the latest 20% as the final test set | Test data cannot influence candidate selection |
| 3 | Create an earlier search-training period and a later validation period | Measures performance on later traffic |
| 4 | Build a class-aware search sample | Controls computation while retaining rare-class influence |
| 5 | Train every candidate on the same search records | Keeps candidate comparisons fair |
| 6 | Evaluate candidates on the same complete validation partition | Prevents sampling differences from determining the winner |
| 7 | Select the highest validation macro F1 | Gives every class equal weight |
| 8 | Refit the winner on the complete first 80% where practical | Uses more historical data after selection |
| 9 | Evaluate once on the untouched latest 20% | Produces the final protected result |
| 10 | Compare the tuned result with the untuned baseline | Establishes whether tuning actually helped |

## Model-specific hyperparameters

| Model | Hyperparameters tuned | What they control |
|---|---|---|
| Logistic Regression | `C` and class weighting | Regularization strength and rare-class influence |
| Random Forest | Number of trees, maximum depth, minimum leaf size, feature sampling and class weighting | Ensemble size, tree complexity and imbalance handling |
| Extra Trees | Number of trees, maximum depth, minimum leaf size, feature sampling and class weighting | Randomization, complexity and imbalance handling |
| HistGradientBoosting | Learning rate, iterations, leaf count, maximum depth, minimum leaf size and L2 regularization | Boosting speed, model complexity and overfitting control |
| XGBoost | Estimators, depth, learning rate, row sampling, column sampling, child weight and regularization | Boosting capacity, sampling and overfitting control |
| LightGBM | Estimators, learning rate, leaf count, depth, child size, sampling and regularization | Tree complexity, training speed and overfitting control |
| MLP | Hidden layers, `alpha`, learning rate and batch size | Network capacity, regularization and optimization |

## Logistic Regression search

| Search area | Values tested | Interpretation |
|---|---|---|
| L2 regularization `C` | 0.03, 0.10, 0.30, 1.00 and 3.00 | Smaller values apply stronger regularization |
| Class weighting | Balanced and unweighted | Balanced weighting gives rare classes more influence |
| Selection metric | Validation macro F1 | Gives every class equal importance |
| Final refit | Complete first 80% | Uses more data after configuration selection |
| Final evaluation | Latest 20% | Measures performance on later traffic |

The random-split tuned result is calculated for presentation consistency. It does not influence parameter selection. Binary Logistic Regression is tuned separately because binary detection and multiclass attack-family classification have different objectives.

## HistGradientBoosting search

| Hyperparameter | Values or alternatives explored | Effect |
|---|---|---|
| Learning rate | Approximately 0.03–0.10 | Controls how much each boosting iteration changes the model |
| Iterations | Approximately 125–300 | Controls the number of boosting updates |
| Maximum leaf nodes | 15, 31 and 63 | Controls tree complexity |
| Maximum depth | Limited or unlimited | Controls interaction depth |
| Minimum samples per leaf | 20, 50 and 100 | Prevents highly specialized leaves |
| L2 regularization | 0.0, 0.1 and 1.0 | Penalizes excessive complexity |
| Class weighting | Balanced | Gives rare classes more influence |

## Random Forest and Extra Trees search

| Hyperparameter | Alternatives explored | Effect |
|---|---|---|
| Number of trees | 100–250 | More trees can stabilize predictions but increase runtime |
| Maximum depth | Limited or unlimited | Controls tree complexity |
| Minimum samples per leaf | 1 or 2 | Reduces highly specific rules |
| Feature sampling | Square root or 50% | Controls diversity among trees |
| Class weighting | None, balanced or balanced subsample | Changes the influence of rare attack classes |

## XGBoost and LightGBM search

| Hyperparameter area | Alternatives explored | Effect |
|---|---|---|
| Number of estimators | Different boosting-round counts | Controls ensemble size |
| Learning rate | Lower and higher rates | Controls how quickly the model learns |
| Tree depth or leaves | Different complexity levels | Controls nonlinear interactions |
| Row sampling | Full or partial | Can reduce overfitting |
| Feature sampling | Full or partial | Can reduce dependence on a small feature group |
| Child or leaf size | Different minimum sizes | Prevents overly specialized regions |
| L2 regularization | Different penalty strengths | Controls overfitting |

## MLP search

| Hyperparameter | Alternatives explored | Effect |
|---|---|---|
| Hidden layers | `(128, 64)`, `(256, 128)`, `(128, 64, 32)` and `(256, 128, 64)` | Controls neural-network capacity |
| L2 penalty `alpha` | `0.0001` and `0.001` | Penalizes large neural-network weights |
| Initial learning rate | `0.001` and `0.0005` | Controls optimization step size |
| Batch size | 1,024 and 2,048 | Controls rows processed per update |
| Early stopping | Enabled | Stops training when internal validation stops improving |

## Candidate-selection metrics

| Metric | Role in tuning |
|---|---|
| Validation macro F1 | Primary candidate-selection metric |
| Attack macro F1 | Checks attack-family performance without benign traffic |
| Weighted F1 | Shows performance weighted by class support |
| Macro precision | Measures average class-level prediction correctness |
| Macro recall | Measures average class-level detection coverage |
| Fit time | Measures training cost |
| Prediction time | Measures inference cost |

## Tuning guardrails

| Guardrail | Reason |
|---|---|
| Use chronological periods | Reduces leakage between related attack bursts |
| Keep the final 20% untouched | Preserves a protected final evaluation |
| Tune on a smaller class-aware sample | Controls cost and ensures rare classes affect selection |
| Use complete validation and test partitions | Preserves the evaluation distributions |
| Select using macro F1 | Prevents benign and common classes from dominating |
| Refit only after selection | Uses additional history without contaminating candidate selection |
| Treat random-split results as secondary | Random splitting can overestimate later-event performance |
| Do not select using 2018 external results | Prevents adapting the model to the external evaluation data |

## Interpreting tuning outcomes

| Outcome | Interpretation |
|---|---|
| Validation and final-test performance improve | The configuration change likely improved generalization |
| Validation improves but final test does not | The candidate may have fit validation-specific behavior |
| Tuned and untuned results are nearly identical | The baseline configuration was already adequate |
| Tuned model performs worse | The selected complexity or settings did not transfer |
| Macro F1 improves but false alarms increase | The improvement may not be operationally acceptable |
| Attack macro F1 improves but weighted F1 declines | Rare-attack performance improved at some cost to common classes |

Tuning is an experiment rather than a guarantee. The tuned configuration should replace the untuned model only when it improves the protected final-test metrics and remains acceptable on attack recall, false alarms, runtime and cross-network behavior.
