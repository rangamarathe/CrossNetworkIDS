# Robustness validation

This directory contains experiments that test whether apparently strong intrusion-detection performance depends on cross-year drift or laboratory-specific shortcuts.

## Experiment

Run `robustness-validation/07_feature_ablation_robustness.ipynb` after downloading the datasets and running the EDA workflow that creates `Data/sample_2018_stratified.csv.gz`. Generated CSV reports are written to `robustness-validation/results/`.

The notebook:

1. loads and harmonizes CICIDS2017 and the stratified CSE-CIC-IDS2018 sample;
2. creates independent chronological training and test partitions;
3. calculates per-class, cross-year KS statistics using training data only;
4. identifies high-drift features using a declared threshold;
5. defines known laboratory-context features separately;
6. trains the same Logistic Regression configuration on four feature sets;
7. evaluates every model on untouched 2017 and 2018 test partitions; and
8. compares macro F1, attack recall, attack F1, and benign false-alarm rate.

## Feature-set comparison

| Feature set | Purpose |
|---|---|
| All shared numeric features | Reference result |
| Exclude high-KS features | Test whether distribution drift drives transfer failure |
| Exclude lab-context features | Test reliance on ports, protocol, TCP windows, and related environment signals |
| Exclude both groups | Test a deliberately drift-conscious predictor set |

## Decision rule

A reduced feature set is considered more robust only when it improves or stabilizes performance across both test datasets without creating an unacceptable increase in false alarms. A small reduction in within-year score may be acceptable if cross-network attack recall improves materially.

KS is a diagnostic rather than an automatic deletion rule. The experiment reports the selected columns and their results so that removal decisions remain auditable.

## Limitations

- The cached 2018 data are stratified and do not preserve natural deployment prevalence.
- Results establish robustness across two represented research environments, not universal generalization.
- A truly unseen third network is still required for stronger deployment claims.
