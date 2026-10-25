# Model Selection Rationale

## Purpose

This document explains why the project evaluates Logistic Regression, Random Forest, Extra Trees, HistGradientBoosting, XGBoost, LightGBM, and a multilayer perceptron. Together, these models provide a structured comparison between an interpretable linear baseline, bagged tree ensembles, boosted tree ensembles, and a neural-network alternative.

The objective is not to include complexity for its own sake. Each model tests a specific hypothesis about the structure of network-flow data and whether additional complexity produces a measurable improvement under chronological evaluation.

## Characteristics of the modeling data

The network-flow data contain:

- highly skewed numeric variables;
- nonlinear relationships;
- threshold effects;
- interactions between packet, byte, flag, port, and duration fields;
- statistical outliers that may represent genuine attacks;
- redundant and correlated variables;
- severe class imbalance;
- rare attack families; and
- more than one million training observations.

An attack pattern may depend on several conditions simultaneously. For example:

```text
Destination port is 22
AND the flow contains repeated connection attempts
AND duration is short
AND packet count is small
```

Tree-based models represent these conditional rules naturally. A linear model requires transformations or explicit interaction terms to represent similar relationships.

## Models included in the experiment

The experiment compares the following model families:

| Category | Model | Role in the experiment |
|---|---|---|
| Linear baseline | Logistic Regression | Establish an interpretable minimum-performance benchmark |
| Bagging ensemble | Random Forest | Test nonlinear decision trees combined through bootstrap aggregation |
| Randomized bagging ensemble | Extra Trees | Test whether greater split randomization improves generalization and efficiency |
| Gradient boosting | HistGradientBoosting | Test efficient histogram-based boosting within the scikit-learn workflow |
| Gradient boosting | XGBoost | Test regularized boosting with detailed tree and sampling controls |
| Gradient boosting | LightGBM | Test a fast, memory-efficient boosting implementation for large datasets |
| Neural network | Multilayer Perceptron | Test whether learned nonlinear representations outperform linear and tree models |

The comparison follows this progression:

```text
Logistic Regression baseline
            ↓
Random Forest and Extra Trees
            ↓
HistGradientBoosting, XGBoost, and LightGBM
            ↓
Multilayer Perceptron
```

## Baseline: Logistic Regression

### Why it was selected

Logistic Regression provides a simple and interpretable baseline. It establishes the minimum performance that more complex models should improve upon.

The baseline uses:

- a signed `log1p` transformation for skewed values;
- feature standardization;
- balanced class weights; and
- the same random and chronological evaluation periods used by the comparison models.

### What it tests

Logistic Regression tests whether the classes can be separated using approximately linear combinations of transformed flow features.

### Advantages

- Relatively easy to interpret.
- Computationally practical.
- Supports class weighting.
- Produces class probabilities.
- Provides a clear benchmark for measuring the value of nonlinear models.

### Limitations

- Does not naturally represent complex threshold interactions.
- Coefficients become difficult to interpret when features are strongly correlated.
- May underfit attack patterns that depend on combinations of network conditions.

## Random Forest

### Why it was selected

Random Forest is a widely used general-purpose tree ensemble. It can model nonlinear relationships and feature interactions without requiring standardization.

Each tree is trained on a bootstrap sample and considers a random subset of features at each split. Combining many trees reduces the variance associated with a single decision tree.

### What it tests

Random Forest tests whether bagged decision trees substantially improve upon the linear baseline.

### Advantages

- Handles nonlinear relationships and interactions.
- Tolerates skewed variables and outliers.
- Requires little feature scaling.
- Works well with large tabular datasets.
- Provides feature-importance measures.

### Limitations

- Can require substantial memory.
- Feature importance may be divided among correlated variables.
- Large ensembles are less interpretable than Logistic Regression.
- May memorize laboratory-specific patterns.

## Extra Trees

### Why it was selected

Extra Trees is related to Random Forest but adds more randomness when selecting split thresholds. Random Forest searches for strong splits among sampled features, while Extra Trees uses more randomized candidate splits.

This additional randomization can reduce variance and training time, although it may introduce more bias.

### What it tests

Extra Trees tests whether greater split randomization improves generalization or reduces overfitting relative to Random Forest.

### Advantages

- Often trains faster than Random Forest.
- Handles nonlinear interactions.
- Can reduce variance through stronger randomization.
- Requires no feature scaling.

### Limitations

- Random splits can add bias.
- Model explanations remain distributed across many trees.
- Correlated and testbed-specific features can still dominate predictions.

## HistGradientBoosting

### Why it was selected

HistGradientBoosting is an efficient gradient-boosted tree implementation designed for large numeric datasets. Instead of evaluating every unique feature value, it groups continuous values into histogram bins.

Boosting builds trees sequentially. Each new tree focuses on correcting errors made by the existing ensemble.

### What it tests

HistGradientBoosting tests whether sequential error correction provides stronger chronological performance than independently trained bagged trees.

### Advantages

- Efficient for large datasets.
- Lower memory use than exact split-search methods.
- Models nonlinear thresholds and interactions.
- Supports balanced class weights.
- Integrates directly with the scikit-learn workflow.

### Limitations

- More sensitive to hyperparameters than bagged trees.
- Can overfit environment-specific shortcuts.
- Individual predictions are more difficult to explain than a linear baseline.

## XGBoost

### Why it was selected

XGBoost is a mature and widely used gradient-boosting implementation. It includes explicit regularization and detailed controls for tree depth, child-node size, row sampling, feature sampling, and learning rate.

### What it tests

XGBoost tests whether a highly optimized, regularized boosting implementation improves rare-class and nonlinear classification.

### Advantages

- Strong predictive performance on structured data.
- Extensive regularization controls.
- Supports row and feature subsampling.
- Efficient histogram-based training.
- Frequently used as a benchmark for tabular machine learning.

### Limitations

- Requires careful tuning.
- Multiclass imbalance requires deliberate sampling or weighting design.
- Can learn testbed-specific shortcuts.
- More operational and configuration complexity than the scikit-learn baseline.

## LightGBM

### Why it was selected

LightGBM is designed for fast and memory-efficient gradient boosting on large datasets. Its leaf-wise tree-growth strategy can improve accuracy efficiently when properly constrained.

### What it tests

LightGBM tests whether a scalable boosting implementation can provide comparable or better performance with practical training time and memory use.

### Advantages

- Fast training on large datasets.
- Memory-efficient histogram binning.
- Supports class weighting.
- Strong performance on numeric tabular data.
- Flexible tree-complexity controls.

### Limitations

- Leaf-wise growth can overfit when unconstrained.
- Requires tuning of leaf count, depth, minimum child size, and regularization.
- Model behavior can depend on implementation-specific defaults.

## Multilayer Perceptron

### Why it was selected

A multilayer perceptron provides a non-tree nonlinear comparison. It learns layered combinations of standardized input features and tests whether a neural network can discover useful representations beyond tree ensembles.

The MLP also directly addresses the question of whether a neural network solves this structured network-flow problem more effectively.

### What it tests

The MLP tests whether learned nonlinear feature combinations outperform manually transformed linear models and tree-based ensembles.

### Advantages

- Represents complex nonlinear relationships.
- Learns interactions across many features.
- Supports flexible architectures.
- Provides a useful comparison with tree-based methods.

### Limitations

- Requires scaling and careful optimization.
- Scikit-learn's MLP does not support class weights.
- Requires class-aware resampling for this imbalanced dataset.
- Architecture, learning rate, regularization, and batch size affect results.
- Neural networks are not automatically superior for structured tabular data.

## Why multiple tree ensembles were included

The tree models represent two different ensemble strategies.

### Bagging

- Random Forest
- Extra Trees

Bagging trains trees independently and combines their predictions. Its main purpose is to reduce variance and improve stability over one decision tree.

### Boosting

- HistGradientBoosting
- XGBoost
- LightGBM

Boosting trains trees sequentially, with later trees focusing on errors made by earlier trees. This often provides stronger predictive performance but increases sensitivity to hyperparameters and data leakage.

Including both strategies answers whether the data benefit more from independent model averaging or sequential error correction.

## Why three boosting implementations were compared

The boosting models use related principles but differ in implementation and controls:

| Model | Main reason for inclusion |
|---|---|
| HistGradientBoosting | Efficient scikit-learn integration and direct balanced-class support |
| XGBoost | Strong regularization and detailed tree controls |
| LightGBM | Speed, memory efficiency, and scalability |

Comparing all three helps determine whether improvements arise from boosting generally or from one implementation.

## Model progression

The project follows a controlled progression:

```text
Interpretable linear baseline
            ↓
Bagged tree ensembles
            ↓
Boosted tree ensembles
            ↓
Neural-network alternative
```

This progression addresses the following questions:

1. Is nonlinear modeling necessary?
2. Do tree ensembles improve upon Logistic Regression?
3. Does boosting outperform bagging?
4. Which boosting implementation provides the strongest chronological result?
5. Does a neural network add measurable value?
6. Does hyperparameter tuning improve performance on a later, untouched test period?
7. Is additional model complexity justified by the observed improvement?

## Planned evaluation

Every model will use the same cleaned features, label policy, and evaluation periods. The initial comparison will report both a stratified random split and a within-class chronological split. The chronological result will be the primary basis for model comparison because it better represents prediction on later traffic.

The planned multiclass measures are:

- macro F1 as the primary metric;
- per-class precision, recall, and F1;
- attack-only macro F1;
- weighted F1 as secondary context;
- confusion matrices; and
- training and prediction time.

Hyperparameters will be selected using a validation period that occurs before the final chronological test period. The final test period will not be used to select models or parameters.

## Model selection criteria

No preferred model is declared before experimentation. The final model decision will consider:

- chronological macro F1;
- per-class recall and F1;
- cross-dataset attack detection;
- benign false-alarm rate;
- stability across attack families;
- reliance on environment-specific features;
- model complexity;
- training and inference cost; and
- interpretability requirements.

A more complex model will be preferred only when it produces a meaningful and repeatable improvement under the chronological evaluation. Random-split performance alone will not determine the winner.

Cross-dataset binary evaluation should be applied to the strongest practical candidates before making a production recommendation. A model with strong within-dataset multiclass performance may still be unsuitable if its attack recall collapses or its false-alarm rate rises substantially in CSE-CIC-IDS2018.

## Relationship to the project goals

The planned model comparison primarily supports the secondary goal of identifying attack families within CICIDS2017. The binary versions of suitable models will support the primary cross-dataset objective by training on CICIDS2017 and evaluating on CSE-CIC-IDS2018.

The design begins with Logistic Regression as the interpretable baseline. Bagged trees, boosted trees, and the MLP are then evaluated to determine whether their additional complexity improves chronological performance. The strongest practical binary candidates will subsequently be compared on cross-dataset attack recall and benign false-alarm rate.
