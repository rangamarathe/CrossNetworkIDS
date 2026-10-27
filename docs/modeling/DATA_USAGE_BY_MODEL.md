# Data usage by model and experiment

## Purpose

This document identifies which project models use complete cleaned data partitions and which use capped, sampled, or resampled training data. It also distinguishes the complete CSE-CIC-IDS2018 population from the cached stratified sample used by the modeling notebooks.

## Definitions

| Term | Meaning in this project |
|---|---|
| Full cleaned 2017 partition | Every eligible CICIDS2017 row assigned to that training, validation, or test partition after cleaning and duplicate handling. |
| Full 2018 sample partition | Every eligible row assigned to that partition from the cached 2018 stratified sample. It does not mean the complete 63.2-million-row population. |
| Partial training | A capped, sampled, controlled-budget, or resampled subset of an available training partition. |
| Full 2018 population | All 63,195,145 CSE-CIC-IDS2018 rows. No model has yet trained or tested on this full population. |

## Summary

| Notebook and model | Training data | Evaluation data | Classification |
|---|---|---|---|
| `baseline/02_logistic_regression_baseline.ipynb`: multiclass Logistic Regression | Complete 2017 random or chronological training partition | Corresponding complete cleaned 2017 test partition | Full cleaned 2017 training and testing |
| `baseline/02_logistic_regression_baseline.ipynb`: binary cross-year Logistic Regression | All effective 2017 training attacks plus at most 400,000 benign rows | Complete 2017 chronological test and complete cached 2018 sample | Partial training; full 2017 test; sampled 2018 test |
| `advanced-modeling/02_ensemble_models_cicids2017.ipynb`: preliminary models | Complete 2017 random or chronological training partition | Corresponding complete cleaned 2017 test partition | Full cleaned 2017 training and testing |
| `advanced-modeling/02_logistic_regression_tuning.ipynb`: tuning search | First 70% of 2017, capped at 50,000 rows per class | Complete middle 10% validation period | Partial tuning training; full validation |
| Same notebook: final tuned Logistic Regression | Complete first 80% of cleaned 2017 | Complete latest 20% | Full cleaned 2017 training and testing |
| `advanced-modeling/03_mlp_experiment.ipynb`: initial MLP | Per-class resample of 5,000 to 50,000 rows | Complete random or chronological 2017 test partition | Partial training; full test |
| `advanced-modeling/04_histgb_fine_tuning.ipynb`: tuning search | Per-class sample of 2,000 to 30,000 rows from the first 60% | Complete middle 20% validation period | Partial tuning training; full validation |
| Same notebook: final HistGradientBoosting | Complete first 80% of cleaned 2017 | Complete latest 20% | Full cleaned 2017 training and testing |
| `advanced-modeling/05_all_model_tuning.ipynb`: tuning search | Per-class sample of 2,000 to 20,000 rows from the first 60% | Complete middle 20% validation period | Partial tuning training; full validation |
| Same notebook: final RF, Extra Trees, XGBoost and LightGBM | Complete first 80% of cleaned 2017 | Complete latest 20% | Full cleaned 2017 training and testing |
| Same notebook: final tuned MLP | Per-class sample of 5,000 to 50,000 rows from the first 80% | Complete latest 20% of cleaned 2017 | Partial training; full test |
| `advanced-modeling/06_cross_dataset_experiments.ipynb` | Controlled samples from 2017 and/or the cached 2018 sample | Complete held-out 2017 partition and complete held-out 2018-sample partition | Partial training; full partition tests; 2018 remains sampled |
| `robustness-validation/07_feature_ablation_robustness.ipynb` | Controlled mixed 2017 and cached-2018-sample training | Complete held-out 2017 and 2018-sample partitions | Partial training; full partition tests; 2018 remains sampled |

## 1. Logistic Regression baseline

### Multiclass baseline

The random and chronological multiclass models use every cleaned CICIDS2017 row assigned to their respective 80% training and 20% testing partitions.

The executed chronological baseline used approximately:

```text
Training: 1,473,071 rows
Testing:    368,276 rows
```

The slight difference from counts in later notebooks comes from differences in model-aligned duplicate and preparation rules.

**Classification:** full cleaned 2017 training and testing.

### Binary cross-year baseline

The binary baseline uses every effective attack in the 2017 chronological training period plus at most 400,000 benign rows. The executed model used:

```text
Training: 725,111 rows
```

It is evaluated on the complete 2017 chronological test partition and the complete 378,952-row cached 2018 sample.

**Classification:** partial 2017 training, full 2017 test, and sampled 2018 test.

## 2. Preliminary ensemble comparison

The preliminary Logistic Regression, Random Forest, Extra Trees, HistGradientBoosting, XGBoost, and LightGBM models use every cleaned CICIDS2017 row assigned to the random or chronological training partition. They are evaluated on every row in the corresponding test partition.

**Classification:** full cleaned 2017 training and testing.

## 3. Tuned Logistic Regression

Parameter selection uses the earliest 70% of every class, capped at 50,000 rows per class. Candidates are evaluated on the complete next 10% validation period.

After parameter selection, the winning configuration is refitted on the complete first 80% and evaluated once on the complete latest 20%.

```text
Parameter-search training: partial
Validation:                full middle 10%
Final refit:               full first 80%
Final test:                full latest 20%
```

## 4. Initial MLP experiment

The MLP uses a class-aware training sample containing at least 5,000 and at most 50,000 rows per class. Rare classes with fewer than 5,000 available rows are sampled with replacement. Large classes are capped at 50,000.

The random and chronological test partitions are not resampled.

**Classification:** partial training and full cleaned 2017 testing.

## 5. HistGradientBoosting tuning

The tuning search uses between 2,000 and 30,000 rows per class from the first 60% period. Candidate models are evaluated on the complete middle 20%.

The selected configuration is then refitted on the complete first 80% and evaluated on the complete latest 20%.

```text
Parameter-search training: partial
Validation:                full middle 20%
Final refit:               full first 80%
Final test:                full latest 20%
```

## 6. Other model tuning

Random Forest, Extra Trees, XGBoost, LightGBM, and MLP candidate searches use between 2,000 and 20,000 rows per class from the first 60% period. All candidates use the complete middle 20% validation period.

After selection:

- Random Forest, Extra Trees, XGBoost, and LightGBM use the complete first 80% for their final refit.
- MLP continues to use a class-aware sample of 5,000 to 50,000 rows per class.
- Every final model is evaluated on the complete latest 20% of cleaned 2017.

## 7. Controlled cross-dataset experiment

The available chronological partitions are:

| Dataset | Available training rows | Held-out test rows |
|---|---:|---:|
| Complete cleaned 2017 | 1,469,282 | 367,333 |
| Cleaned cached 2018 sample | 268,654 | 67,174 |

The notebook sets the common training budget to the smallest of the configured 300,000-row maximum and the two available training partitions:

```text
minimum of 300,000, 1,469,282 and 268,654 = 268,654
```

The controlled strategies therefore use:

| Strategy | 2017 training rows | 2018-sample training rows | Total |
|---|---:|---:|---:|
| 2017 only | 268,654 | 0 | 268,654 |
| 2018 only | 0 | 268,654 | 268,654 |
| Mixed equal budget | 134,327 | 134,327 | 268,654 |
| Mixed expanded budget | 268,654 | 268,654 | 537,308 |

Every model is evaluated on all 367,333 held-out 2017 rows and all 67,174 held-out rows from the cleaned cached 2018 sample.

**Classification:** controlled partial training, complete held-out 2017 testing, and complete held-out sample testing for 2018.

## 8. Robustness feature ablation

The robustness experiment uses:

```text
268,654 selected 2017 training rows
268,654 selected cached-2018 training rows
537,308 total mixed training rows
```

All feature-set variants use the same training records. They are evaluated on the complete held-out 2017 partition and the complete held-out partition of the cached 2018 sample.

**Classification:** partial controlled training; full held-out partitions relative to the data sources used.

## Models using complete cleaned 2017 training partitions

- Multiclass Logistic Regression baseline
- Preliminary Logistic Regression and ensemble models
- Final tuned Logistic Regression
- Final tuned HistGradientBoosting
- Final tuned Random Forest
- Final tuned Extra Trees
- Final tuned XGBoost
- Final tuned LightGBM

## Models using partial or resampled training

- Binary cross-year Logistic Regression baseline
- Logistic Regression parameter-search candidates
- Initial MLP
- HistGradientBoosting parameter-search candidates
- Multi-model parameter-search candidates
- Final tuned MLP
- Controlled cross-dataset models
- Robustness feature-ablation models

## Complete 2018 population status

The complete CSE-CIC-IDS2018 population has been processed for streamed EDA counts and data-quality audits. It has not been used for model fitting or complete-population inference.

Current 2018 model evaluations use either:

- the complete 378,952-row cached stratified sample in the binary baseline; or
- the 67,174-row held-out portion of the cleaned cached sample in controlled experiments.

The correct project statement is:

> Final 2017 models generally use complete cleaned train and test partitions, except the MLP and controlled binary experiments. All 2018 modeling currently uses a stratified sample rather than the complete 63.2-million-row dataset.

## Recommended next step

Keep the trained models fixed and run chunked inference over the complete 63,195,145-row CSE-CIC-IDS2018 population. Accumulate confusion counts, per-family recall, false-alarm rates, and total alert volume without loading the full dataset into memory.
