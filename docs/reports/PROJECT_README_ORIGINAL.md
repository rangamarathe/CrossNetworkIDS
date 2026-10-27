# Network Intrusion Detection from Flow Statistics

## Project overview

This capstone examines whether network-flow statistics can distinguish benign traffic from cyberattacks and classify attack families. It uses the improved CICIDS2017 and CSE-CIC-IDS2018 datasets, which correct several labeling problems in the original CIC releases.

## Research question

How effectively can network-flow statistics distinguish benign traffic from cyberattacks and separate attack families, and how much can evaluation design overstate that performance?

## Data

The local dataset contains 65,295,121 network flows across 15 CSV files:

| Dataset | Files | Rows | Effective attack rows | Effective attack share |
|---|---:|---:|---:|---:|
| CICIDS2017 improved | 5 | 2,099,976 | 505,431 | 24.07% |
| CSE-CIC-IDS2018 improved | 10 | 63,195,145 | 3,535,422 | 5.59% |

Rows ending in `- Attempted` represent connections without an effective malicious payload. The baseline modeling policy folds these rows into `BENIGN`. The alternative would be to remove them or model them as a separate class.

The data can be downloaded with [`00_download_data.ipynb`](../../00_download_data.ipynb). The source is the [Improved CICIDS2017 and CSE-CIC-IDS2018 dataset on Kaggle](https://www.kaggle.com/datasets/ernie55ernie/improved-cicids2017-and-csecicids2018).

Supporting documentation:

- [`DATA_DICTIONARY.md`](../analysis/DATA_DICTIONARY.md) explains the 91 columns and known quality issues.
- [`ATTACK_LABELS.md`](../analysis/ATTACK_LABELS.md) documents every label, attempted-category code, cross-year mapping, and measured attack signature.

## Methodology

### Exploratory analysis

[`01_eda.ipynb`](../../EDA/01_eda.ipynb) contains the core CICIDS2017 EDA and initial CSE-CIC-IDS2018 population inventory. It performs:

- schema, file-size, missing-value, infinite-value, constant-feature, and duplicate checks;
- class-balance analysis for both years;
- protocol and destination-port analysis by attack class;
- attack-volume analysis over time;
- feature-correlation screening;
- explicit IQR and extreme-quantile outlier analysis;
- categorical protocol and destination-port plots;
- class-level continuous-feature distributions; and
- PCA as an initial view of class separability.

[`01_eda_advanced_questions.ipynb`](../../EDA/01_eda_advanced_questions.ipynb) contains the hypothesis-driven analyses. It covers:

- attempted-attack label analysis;
- per-class feature-separation testing;
- hierarchical feature-redundancy clustering;
- single-feature and testbed-shortcut analysis;
- near-duplicate split-leakage analysis; and
- cross-year feature-distribution drift analysis.

CICIDS2017 is loaded in full for detailed analysis. CSE-CIC-IDS2018 is streamed for exact label counts because it is approximately 36 GB.

[`01_eda_2018_extended.ipynb`](../../EDA/01_eda_2018_extended.ipynb) contains the standalone extended CSE-CIC-IDS2018 analysis: an exact cached quality, anomaly, protocol, port, and file audit of all 63.2 million rows plus sample-based distributions, attack timing, class comparisons, feature-redundancy clustering, PCA, and generated findings.

### Cleaning and feature engineering

The modeling workflow:

1. maps every `- Attempted` label to `BENIGN`;
2. removes identifiers and leakage-prone fields (`id`, flow ID, IP addresses, source port, timestamp, and attempted-category code);
3. converts infinite rate values to missing and replaces them with the corresponding finite column maximum;
4. drops constant features;
5. removes exact duplicate feature/label rows; and
6. derives packet totals, payload totals, duration in seconds, bytes per packet, and direction ratios for interpretation; and
7. applies a signed `log1p` transform and standardization for Logistic Regression. Timestamp-derived hour and time bins are used only for EDA because they encode the scheduled attack window.

### Baseline and evaluation

[`02_logistic_regression_baseline.ipynb`](../../baseline/02_logistic_regression_baseline.ipynb) establishes the class-weighted multinomial Logistic Regression baseline under random and chronological splits. It also contains the binary cross-year Logistic Regression check moved from the extended EDA.

[`02_logistic_regression_tuning.ipynb`](../../advanced-modeling/02_logistic_regression_tuning.ipynb) tunes L2 regularization strength and class weighting on an early training period, selects by middle-period macro F1, refits the winner on the complete first 80%, and evaluates once on the untouched latest 20%. Binary tuning is performed separately and CSE-CIC-IDS2018 is used only after selection as an external check.

[`02_ensemble_models_cicids2017.ipynb`](../../advanced-modeling/02_ensemble_models_cicids2017.ipynb) compares that baseline with Random Forest, Extra Trees, HistGradientBoosting, XGBoost, and LightGBM.

[`03_mlp_experiment.ipynb`](../../advanced-modeling/03_mlp_experiment.ipynb) evaluates a two-hidden-layer multilayer perceptron using the same test splits and metrics. Because this scikit-learn MLP does not accept class weights, its training data use class-aware resampling while the full test sets remain unchanged.

[`04_histgb_fine_tuning.ipynb`](../../advanced-modeling/04_histgb_fine_tuning.ipynb) tunes the strongest time-aware model. It selects HistGradientBoosting parameters on a middle validation period, refits the winner on the full first 80% of each class, and evaluates once on the untouched final 20%.

[`05_all_model_tuning.ipynb`](../../advanced-modeling/05_all_model_tuning.ipynb) applies the same validation design to Random Forest, Extra Trees, XGBoost, LightGBM, and MLP, then compares all tuned and untuned results.

Two 80/20 evaluations are used:

- **Stratified random split:** comparable with common benchmark practice, but susceptible to traffic-burst leakage.
- **Within-class time-ordered split:** the earliest 80% of each class trains the model and the latest 20% tests it.

Macro F1 is the primary metric because it balances precision and recall within each class and then gives every class equal weight. This makes rare attacks visible in the aggregate score. Weighted F1 and accuracy are secondary because benign flows dominate.

## Results

### Exploratory findings

- Both datasets have the same 91-column schema and no missing cells.
- The data are severely imbalanced. Benign traffic represents 75.36% of CICIDS2017 and 93.92% of CSE-CIC-IDS2018.
- Several attack classes contain fewer than 50 effective examples, which makes their test metrics unstable.
- A small number of rate fields contain infinity when flow duration is zero.
- Attacks occur in scheduled bursts on particular days. Random row splitting can place near-duplicate flows from one burst into both training and test sets.
- Many numeric features are strongly correlated or mathematically redundant.
- IQR screening identifies many statistical outliers, but these include real flooding and slow-DoS signatures. They are retained unless a field violates a known constraint. Infinite rates and negative header lengths are handled as measurement anomalies.
- PCA and class distributions indicate that major flooding and scanning attacks are separable, while rare web and infiltration classes overlap more with benign or related attack traffic.

### Model results

| Model | Random-split macro F1 | Time-ordered macro F1 |
|---|---:|---:|
| Logistic Regression baseline | 0.7969 | 0.7278 |
| Tuned Logistic Regression | — | 0.7746 |
| Random Forest | 0.9342 | 0.8908 |
| Extra Trees | 0.9552 | 0.8924 |
| HistGradientBoosting | 0.9854 | **0.9236** |
| XGBoost | **0.9858** | 0.9108 |
| LightGBM | 0.9787 | 0.9182 |
| MLP (128, 64) | 0.8244 | 0.7312 |
| Tuned HistGradientBoosting | — | 0.9199 |

The required Logistic Regression baseline loses 0.0691 macro F1 under time-ordered evaluation. Every ensemble also declines. HistGradientBoosting produces the best time-ordered result at 0.9236 macro F1, while XGBoost leads only on the random split. This confirms that model rankings and reported performance depend on how the data are split.

Logistic Regression tuning selected unweighted L2 with `C=1.0` for the multiclass task. On the untouched chronological test period, macro F1 improved from 0.7278 to 0.7746 (+0.0468) and attack-class macro F1 improved from 0.7101 to 0.7598. The improvement came from higher macro precision (0.6868 to 0.7857) with lower macro recall (0.9457 to 0.7760), so the tuned model is more conservative rather than uniformly better for every attack. Separate binary tuning selected the original balanced `C=1.0` settings; with the same training-population policy, its chronological and 2018 results reproduce the baseline. Tuning therefore helps the secondary multiclass objective but does not improve the primary binary cross-network detector.

The MLP reaches 0.7312 time-ordered macro F1, only 0.0033 above Logistic Regression and 0.1924 below HistGradientBoosting. Its 10.3-second fit used 288,996 resampled training rows rather than the full training set, so the runtime is not directly comparable with the full-data models. The experiment does not support replacing the tree ensembles with this MLP.

Fine tuning selected `learning_rate=0.05`, `max_iter=250`, `max_leaf_nodes=63`, `min_samples_leaf=20`, and `l2_regularization=0.1`. The tuned model reached 0.9199 macro F1 on the untouched time-ordered test period, 0.0037 below the original HistGradientBoosting result. The original configuration therefore remains the preferred model. This also demonstrates why tuning decisions must be separated from final test evaluation: a validation improvement does not guarantee better future-period performance.

The broader tuning experiment produced the following time-ordered test results:

| Model | Untuned macro F1 | Tuned macro F1 | Change |
|---|---:|---:|---:|
| HistGradientBoosting | **0.9236** | 0.9199 | -0.0037 |
| LightGBM | **0.9182** | 0.9182 | 0.0000 |
| XGBoost | **0.9108** | 0.9053 | -0.0055 |
| Extra Trees | **0.8924** | 0.8924 | 0.0000 |
| Random Forest | **0.8908** | 0.8908 | 0.0000 |
| MLP | 0.7312 | **0.8075** | +0.0763 |

Only the MLP improved materially. Its selected `(256, 128)` architecture raised macro F1 by 0.0763, but it remained 0.1161 below the original HistGradientBoosting model. LightGBM, Random Forest, and Extra Trees selected their original candidate, while the selected XGBoost and HistGradientBoosting alternatives performed worse on the later test period. The original HistGradientBoosting model remains the overall recommendation.

The strongest models classify common DDoS, DoS, and brute-force traffic reliably. Results for Heartbleed, Infiltration, SQL Injection, and XSS should not be treated as stable because their time-ordered test sets contain only a few observations.

## Limitations

- CICIDS is a controlled testbed, not live production traffic.
- IP addresses, ports, operating-system fingerprints, and scheduled attack windows can encode the capture environment.
- Rare-class metrics have high variance.
- The current models are trained and evaluated within CICIDS2017. Cross-year generalization to CSE-CIC-IDS2018 has not yet been measured.
- The tuning search used one time-based validation period rather than repeated temporal folds, so its selected parameters may be sensitive to that period.

## Next steps

1. Evaluate class weighting or resampling for rare classes.
2. Repeat tuning with multiple temporal validation folds.
3. Map compatible 2017 and 2018 labels and run cross-year tests.
4. Add precision-recall curves and threshold analysis.
5. Use permutation importance or SHAP to identify testbed-specific shortcuts.

## Repository contents

| File | Purpose |
|---|---|
| `00_download_data.ipynb` | Optional dataset download and extraction |
| `EDA/01_eda.ipynb` | Core grading-aligned CICIDS2017 EDA and initial CSE-CIC-IDS2018 inventory |
| `EDA/01_eda_advanced_questions.ipynb` | Advanced Q1–Q6 analysis of labels, separability, redundancy, shortcuts, leakage, and cross-year drift |
| `EDA/01_eda_2018_extended.ipynb` | Standalone extended CSE-CIC-IDS2018 EDA with a full streamed audit and sample-based analysis |
| `docs/README_EDA.md` | Step-by-step explanation of every EDA stage and its business purpose |
| `docs/PROJECT_BUSINESS_GOAL.md` | Primary binary and secondary multiclass project goals, metrics, and success criteria |
| `MODEL_SELECTION_RATIONALE.md` | Why each baseline, ensemble, boosting, and neural-network model was selected |
| `baseline/02_logistic_regression_baseline.ipynb` | Standalone Logistic Regression baseline and cross-year check |
| `02_logistic_regression_tuning.ipynb` | Leakage-safe multiclass and binary Logistic Regression tuning with baseline comparison |
| `02_ensemble_models_cicids2017.ipynb` | Baseline and preliminary ensemble comparison |
| `03_mlp_experiment.ipynb` | MLP experiment and model comparison |
| `04_histgb_fine_tuning.ipynb` | Leakage-safe HistGradientBoosting tuning and base comparison |
| `05_all_model_tuning.ipynb` | Fine tuning and comparison of the remaining ensembles and MLP |
| `02_summary.csv` | Overall model metrics |
| `02_per_class_metrics.csv` | Per-class precision, recall, F1, and support |
| `baseline/02_logistic_baseline_summary.csv` | Standalone Logistic Regression baseline metrics |
| `baseline/02_logistic_baseline_per_class.csv` | Baseline per-class precision, recall, F1, and support |
| `baseline/02_logistic_cross_year_detection.csv` | Cross-year family detection and false-alarm rates |
| `02_logistic_tuning_search.csv` | Multiclass Logistic Regression validation search results |
| `02_logistic_binary_tuning_search.csv` | Binary Logistic Regression validation search results |
| `02_logistic_tuned_summary.csv` | Tuned multiclass chronological test summary |
| `02_logistic_tuned_per_class.csv` | Tuned multiclass per-class chronological test metrics |
| `02_logistic_tuned_cross_year.csv` | Tuned binary chronological and 2018 alarm/detection rates |
| `02_logistic_baseline_vs_tuned.csv` | Untuned-versus-tuned Logistic Regression comparison |
| `04_histgb_tuning_search.csv` | Validation results for every tuning candidate |
| `04_tuned_summary.csv` | Tuned model test metrics and selected parameters |
| `04_tuned_per_class_metrics.csv` | Tuned model per-class test metrics |
| `05_tuning_search_results.csv` | Validation metrics for all multi-model tuning candidates |
| `05_tuned_model_summary.csv` | Final test metrics and selected parameters for each tuned model |
| `05_tuned_per_class_metrics.csv` | Per-class metrics for the tuned model winners |
| `05_base_vs_tuned_comparison.csv` | Untuned-versus-tuned test comparison |
| `docs/DATA_DICTIONARY.md` | Column definitions and quality notes |
| `docs/ATTACK_LABELS.md` | Label definitions and attack signatures |

## Reproduction

Start Jupyter from the repository root. Run `EDA/01_eda.ipynb` first, followed by `EDA/01_eda_advanced_questions.ipynb` and `EDA/01_eda_2018_extended.ipynb`. Then continue with `baseline/02_logistic_regression_baseline.ipynb`, `advanced-modeling/02_logistic_regression_tuning.ipynb`, the ensemble, MLP, and remaining tuning notebooks. Python dependencies include `numpy`, `pandas`, `matplotlib`, `seaborn`, `scipy`, `scikit-learn`, `xgboost`, and `lightgbm`. The raw data directory is intentionally not duplicated in the repository deliverables because it is approximately 36 GB.
