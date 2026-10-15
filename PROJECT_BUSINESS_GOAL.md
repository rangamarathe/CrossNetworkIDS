# Project Business Goal

## Project title

Cross-Network Machine Learning for Network Intrusion Detection

## Business problem

Organizations need to identify malicious network connections quickly while limiting false alerts that consume security-operations resources. Machine-learning models can learn patterns in network-flow statistics, but strong results inside one research dataset do not guarantee that a model will work in a different network environment.

Network configurations, operating systems, services, traffic patterns, attack tools, and data-collection procedures can change between environments. A model may also learn laboratory-specific shortcuts, such as destination ports or TCP-window values, instead of reusable attack behavior. The central business concern is therefore not only whether a model can detect attacks in its training environment, but whether its detection behavior transfers to new traffic.

## Overall project goal

Build and evaluate machine-learning models that use network-flow statistics to detect malicious traffic and identify attack families. The project uses the improved CICIDS2017 and CSE-CIC-IDS2018 datasets to measure both within-dataset predictive performance and cross-dataset generalization.

The project has two related objectives:

1. A primary binary attack-detection objective that supports alerting and security triage.
2. A secondary multiclass objective that supports attack identification and diagnostic analysis.

## Primary goal: Binary attack detection across datasets

Develop a binary classifier that predicts whether a network connection is:

- benign; or
- malicious.

The model is trained using CICIDS2017 and evaluated using CSE-CIC-IDS2018. This experiment measures whether a detector learned in one network-security test environment can flag malicious traffic in another.

### Primary business question

> Can a machine-learning model trained on CICIDS2017 reliably distinguish benign from malicious network traffic in CSE-CIC-IDS2018 while maintaining an acceptable false-alarm rate?

### Primary evaluation measures

The cross-dataset binary evaluation emphasizes:

- **Attack recall:** the share of effective attacks correctly flagged as malicious.
- **Benign false-alarm rate:** the share of genuine benign traffic incorrectly flagged as malicious.
- **Per-family detection rate:** attack recall calculated separately for each compatible attack family.
- **Attempted-attack alarm rate:** the share of unsuccessful attempted attacks flagged by the model even though the baseline labeling policy treats them as benign.
- **Cross-dataset performance change:** the difference between chronological CICIDS2017 performance and CSE-CIC-IDS2018 performance.

Overall accuracy on a class-capped or stratified sample is not treated as a primary cross-dataset measure because the sample does not preserve natural class prevalence.

### Primary business value

The binary detector represents the first decision in an operational security workflow: whether a connection should be allowed to pass without action or should be prioritized for investigation.

The cross-dataset experiment provides evidence about:

- the expected loss of detection performance when the network environment changes;
- which attack families transfer successfully;
- which attack families require new training data or different features;
- how much the benign alert volume may increase; and
- whether a benchmark model is appropriate for deployment beyond its original testbed.

## Secondary goal: Multiclass attack-family classification

Develop multiclass models that classify a network flow as benign or as a specific attack family. Examples include:

- DDoS;
- DoS Hulk;
- DoS GoldenEye;
- DoS Slowloris;
- DoS Slowhttptest;
- port scanning;
- SSH or FTP brute force;
- botnet activity;
- web attacks; and
- infiltration.

The secondary analysis compares Logistic Regression, Random Forest, Extra Trees, HistGradientBoosting, XGBoost, LightGBM, and a multilayer perceptron neural network within CICIDS2017.

### Secondary business question

> When malicious traffic is present, how accurately can machine-learning models identify the attack family, and which attack types are the most difficult to classify?

### Secondary evaluation measures

The multiclass evaluation uses:

- **Macro F1:** the primary metric because it gives every traffic class equal importance.
- **Per-class precision:** how often an alert for a particular class is correct.
- **Per-class recall:** how much of each attack class is detected.
- **Per-class F1:** the balance between precision and recall for each class.
- **Confusion matrices:** which attack families are mistaken for benign traffic or for another attack.
- **Weighted F1:** a secondary aggregate measure influenced by common classes.
- **Attack-only macro F1:** average F1 across attack classes without the benign class.
- **Random-versus-chronological change:** how much performance declines when the model is evaluated on later traffic.

### Secondary business value

Attack-family classification can help a security team:

- prioritize incidents by likely attack type;
- route alerts to the appropriate investigation process;
- understand which controls or services are being targeted;
- identify classes that require more training data; and
- diagnose why a binary detector succeeds or fails.

This is a supporting objective. A correct attack-family label is useful only after the system can reliably distinguish malicious from benign traffic.

## Research questions

The project addresses the following research questions:

1. How accurately can network-flow statistics distinguish benign from malicious traffic?
2. How well does a binary detector trained on CICIDS2017 transfer to CSE-CIC-IDS2018?
3. Which attack families remain detectable across datasets, and which experience the largest decline?
4. How does the benign false-alarm rate change across network environments?
5. How accurately can multiclass models identify individual attack families?
6. Which attack families are most frequently confused or missed?
7. How much does a random train/test split overstate performance compared with chronological evaluation?
8. Do near-duplicate flows allow models to memorize attack bursts?
9. Do models depend on laboratory-specific fields such as destination port or TCP-window values?
10. Do ensemble models or a neural network improve upon the Logistic Regression baseline?
11. Does hyperparameter tuning improve performance on an untouched chronological test period?

## Data strategy

### CICIDS2017

CICIDS2017 is used for:

- detailed exploratory analysis;
- baseline development;
- random and chronological evaluation;
- multiclass model comparison; and
- model tuning.

### CSE-CIC-IDS2018

CSE-CIC-IDS2018 is used for:

- cross-year feature-distribution analysis;
- compatible attack-family comparisons; and
- testing a binary detector trained on CICIDS2017.

The two datasets share the same flow-feature schema but were generated in different network environments. This makes them appropriate for studying transfer behavior while still requiring careful label-family mapping.

## Evaluation design

### Random split

A stratified random 80/20 split provides a conventional benchmark. It preserves class proportions but can place related or near-duplicate flows from the same scripted attack burst in both training and testing.

### Chronological split

Within each class, the earliest 80% of flows are used for training and the latest 20% for testing. This is the primary within-dataset evaluation because the model must predict later traffic that was not available during training.

### Cross-dataset evaluation

The model is trained on CICIDS2017 and evaluated on compatible traffic from CSE-CIC-IDS2018. This is the primary generalization test and the closest project approximation to applying the detector in a different network environment.

## Modeling strategy

### Baseline model

Class-weighted Logistic Regression is used as the baseline because it is:

- easy to interpret;
- computationally practical;
- capable of producing class probabilities;
- suitable for identifying whether nonlinear models add value; and
- a clear minimum benchmark for later experiments.

### Comparison models

The secondary multiclass analysis evaluates:

- Random Forest;
- Extra Trees;
- HistGradientBoosting;
- XGBoost;
- LightGBM; and
- a multilayer perceptron.

The strongest models are then considered for hyperparameter tuning using a time-aware validation period.

## Success criteria

The project is successful if it provides a defensible answer to both objectives, even if cross-dataset performance is lower than within-dataset performance.

### Primary success criteria

- Produce a reproducible binary model trained on CICIDS2017.
- Measure attack-family recall and benign false alarms on both the 2017 chronological test period and 2018 traffic.
- Identify which attack families transfer and which fail.
- Quantify how the false-alarm rate changes across datasets.
- Explain major performance changes using feature drift, label policy, or environment-specific shortcuts.

No universal numerical deployment threshold is assumed because acceptable recall and false-alarm rates depend on organizational risk tolerance, network volume, and analyst capacity.

### Secondary success criteria

- Establish a Logistic Regression multiclass baseline.
- Compare nonlinear ensembles and a neural network using the same data preparation and test periods.
- Report macro F1 and per-class performance.
- Identify the strongest chronological model.
- Determine whether tuning improves the untouched chronological test result.
- Document classes whose results are unreliable because of small sample sizes.

## Important limitations

- CICIDS2017 and CSE-CIC-IDS2018 are controlled research testbeds rather than live production networks.
- Some fields describe the laboratory hosts and services rather than attack behavior.
- The two datasets do not contain exactly the same attack labels, so cross-dataset family mapping is required.
- Several attack classes contain very few observations.
- Attempted attacks are attack-shaped connections without an effective malicious payload. Treating them as benign introduces label ambiguity.
- A stratified 2018 sample supports per-family recall and false-alarm analysis but does not support population-level accuracy unless natural prevalence is restored.
- Cross-year success does not prove that a model will generalize to every production network.

## Expected project outcome

The project will produce:

1. A consolidated EDA describing data quality, imbalance, leakage, shortcuts, and cross-year drift.
2. An interpretable Logistic Regression baseline.
3. A binary CICIDS2017-to-CSE-CIC-IDS2018 generalization experiment.
4. A multiclass comparison of linear, ensemble, boosting, and neural-network models.
5. Time-aware tuning experiments for selected models.
6. A documented recommendation that distinguishes benchmark performance from expected cross-network behavior.

## Final decision framework

The recommended model should not be selected from random-split accuracy alone. The decision should consider:

1. chronological macro F1 for within-dataset multiclass performance;
2. cross-dataset attack recall;
3. benign false-alarm rate;
4. stability across attack families;
5. dependence on environment-specific features;
6. model complexity and interpretability; and
7. the operational cost of missed attacks and unnecessary alerts.

This framework keeps the project aligned with the business need: detecting malicious traffic reliably enough to support security operations when network conditions differ from the training environment.
