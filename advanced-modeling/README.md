# Advanced Modeling: Experiments and Tuning

This directory contains work beyond the Phase 1 EDA and untuned Logistic Regression baseline:

- tuned Logistic Regression;
- Random Forest, Extra Trees, HistGradientBoosting, XGBoost, and LightGBM experiments;
- the multilayer perceptron experiment;
- fine-tuning notebooks and search results; and
- model comparison outputs and presentation material.

The notebooks are configured to run from this directory. They read the raw datasets and baseline outputs using `../Data` and `../baseline/02_logistic_baseline_*.csv` paths.

Suggested order:

1. `02_logistic_regression_tuning.ipynb`
2. `02_ensemble_models_cicids2017.ipynb`
3. `03_mlp_experiment.ipynb`
4. `04_histgb_fine_tuning.ipynb`
5. `05_all_model_tuning.ipynb`
6. `06_cross_dataset_experiments.ipynb`

## Cross-dataset experiment

`06_cross_dataset_experiments.ipynb` compares 2017-only, 2018-only, equal-budget mixed-year, and expanded mixed-year binary Logistic Regression models. Every strategy is evaluated separately on untouched chronological partitions from both years.

| Training strategy | 2017 test macro F1 | 2018 test macro F1 |
|---|---:|---:|
| 2017 only | 0.9900 | 0.6936 |
| 2018 only | 0.7945 | 0.9851 |
| Mixed, equal total budget | 0.9617 | 0.9667 |
| Mixed, expanded budget | 0.9629 | 0.9675 |

The single-year transfer losses provide evidence of substantial and directional domain shift. Mixed training is much more stable across the two represented environments, and nearly identical fixed-budget and expanded-budget results suggest that environmental diversity—not just additional row count—drives most of the improvement. Because both environments appear in mixed training, this demonstrates robustness across known domains rather than generalization to an unseen third network.

Generated outputs include `06_cross_dataset_summary.csv`, `06_cross_dataset_confusion_counts.csv`, and `06_cross_dataset_family_detection.csv`.

## Deployment architecture

[`DEPLOYMENT_ARCHITECTURE.md`](DEPLOYMENT_ARCHITECTURE.md) translates the experimental findings into a privacy-preserving hybrid deployment design covering centralized training, on-premises and edge inference, sensitive-data restrictions, real-time processing, LLM explanations, model governance, monitoring, security, and phased production rollout.

[`SCALABILITY_ARCHITECTURE.md`](SCALABILITY_ARCHITECTURE.md) defines how traffic ingestion, active-flow state, feature extraction, inference, training, storage, customer growth, alert handling, LLM explanations, availability, and cost can scale from a single-site prototype to a multi-customer deployment.

[`MLOPS_AIOPS_OPERATING_MODEL.md`](MLOPS_AIOPS_OPERATING_MODEL.md) defines the responsibilities and integration of MLOps, AIOps, and SecOps, including data and model lineage, validation gates, deployment automation, monitoring, platform event correlation, analyst feedback, governance boundaries, and staged implementation.

[`TECHNOLOGY_COMPONENTS.md`](TECHNOLOGY_COMPONENTS.md) identifies the concrete research, collection, streaming, inference, storage, MLOps, observability, AIOps, SecOps, privacy, security, CI/CD, and LLM components required for the minimum viable and production-scale platforms.

## Whitepaper

[`WHITEPAPER.md`](WHITEPAPER.md) consolidates the project’s business problem, datasets, EDA, evaluation methodology, baseline and nonlinear model results, tuning experiments, cross-dataset findings, neural-network and LLM options, real-time data considerations, deployment architecture, limitations, and recommended next work.

See `MODEL_SELECTION_RATIONALE.md` for the pre-experiment model-selection rationale and `README_original.md` for the earlier comprehensive project narrative.
