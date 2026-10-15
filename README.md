# Cross-Network Intrusion Detection: EDA and Baseline

## Project goal

This project investigates whether network-flow statistics can identify malicious traffic and whether a detector trained on one research network can generalize to another.

The primary goal is binary detection:

> Is this network connection benign or an effective attack?

The secondary goal is multiclass identification:

> If the connection is malicious, which attack family does it most closely resemble?

The analysis uses CICIDS2017 for exploration and baseline development and CSE-CIC-IDS2018 as an external-network comparison. Labels ending in `- Attempted` are treated as benign for the baseline because they did not deliver an effective malicious payload; the advanced EDA documents the limitations of that policy.

## Data

> **Data setup required:** The raw CICIDS2017 and CSE-CIC-IDS2018 data files are not included in this repository because of their size. Before running any EDA or modeling notebook, run [`00_download_data.ipynb`](00_download_data.ipynb) to download and prepare the datasets in the expected `Data/` directory.

| Dataset | Files | Flows | Role |
|---|---:|---:|---|
| Improved CICIDS2017 | 5 | 2,099,976 | Core EDA and baseline development |
| Improved CSE-CIC-IDS2018 | 10 | 63,195,145 | Detailed EDA and external generalization check |

Both datasets use the same 91-column schema. CICIDS2017 is analyzed in memory. The approximately 36 GB CSE-CIC-IDS2018 collection is audited in chunks, with cached summaries and a stratified per-label sample used for distribution-heavy analysis.

## EDA notebooks

### [`01_eda.ipynb`](01_eda.ipynb)

Core, grading-aligned exploratory analysis:

- file and schema inventory;
- missing, infinite, invalid, constant, and duplicate-value checks;
- feature engineering;
- outlier and anomaly analysis;
- class imbalance;
- protocol and destination-port analysis;
- attack timing and temporal-leakage risk;
- correlation and multicollinearity;
- class-level continuous distributions;
- PCA; and
- initial CSE-CIC-IDS2018 population inventory.

Each applicable section includes why the analysis is performed, its business relevance, and a results-based takeaway.

### [`01_eda_advanced_questions.ipynb`](01_eda_advanced_questions.ipynb)

Hypothesis-driven analyses separated from the initial EDA:

- Q1: attempted-label ambiguity;
- Q2: per-class feature separation;
- Q3: feature redundancy;
- Q4: laboratory and environment shortcuts;
- Q5: near-duplicate split leakage; and
- Q6: cross-year distribution drift.

### [`01_eda_2018_extended.ipynb`](01_eda_2018_extended.ipynb)

Standalone CSE-CIC-IDS2018 analysis containing:

- an exact cached audit of all 63.2 million rows;
- data-quality and anomaly checks;
- protocol and destination-port analysis;
- sampled outlier and distribution analysis;
- attack timing;
- class-level comparisons;
- feature-redundancy clustering;
- PCA; and
- programmatically generated findings.

For a step-by-step explanation of the EDA, see [`README_EDA.md`](README_EDA.md).

## Key EDA findings

- The two datasets share a 91-column schema but differ substantially in traffic and attack distributions.
- Benign traffic represents 75.36% of CICIDS2017 and 93.92% of CSE-CIC-IDS2018.
- Several attack classes contain very few examples, so accuracy alone is misleading; macro F1 and per-class metrics are required.
- Network-flow variables are strongly skewed. Many statistical outliers are legitimate attack behavior and should not be deleted automatically.
- CICIDS2017 attacks occur in scheduled bursts. Random splitting can place related flows from the same event in both training and testing.
- Numerous variables are mathematically or behaviorally redundant, complicating coefficient and feature-importance interpretation.
- Ports, protocols, and TCP-window fields can reveal the laboratory environment rather than general malicious behavior.
- Compatible attack families drift between 2017 and 2018, so strong within-year performance does not establish deployment readiness.

## Baseline notebooks

### [`02_logistic_regression_baseline.ipynb`](02_logistic_regression_baseline.ipynb)

The fixed reference model uses:

- signed-log transformation;
- feature standardization;
- L2 Logistic Regression with `C=1.0`;
- balanced class weights;
- random and within-class chronological splits;
- macro F1 as the primary multiclass metric; and
- a separate binary CICIDS2017-to-2018 generalization check.

The untuned baseline is intentionally retained as a simple, reproducible comparison point.

## Baseline results

| Model | Random-split macro F1 | Chronological macro F1 | Attack macro F1 |
|---|---:|---:|---:|
| Untuned Logistic Regression | 0.8241 | 0.7278 | 0.7101 |

The binary baseline’s CSE-CIC-IDS2018 results also show substantial variation by family: DDoS and Slowloris transfer well, while Botnet, DoS Hulk, web attacks, and Infiltration transfer poorly. The 2018 benign false-alarm rate is 3.26%, compared with 0.54% on the 2017 chronological test.

## Evaluation rationale

Macro F1 is the primary multiclass metric because it calculates F1 separately for every class and gives rare classes equal weight. Weighted F1 and accuracy are secondary because benign traffic and common attacks dominate the dataset.

For binary detection, attack recall and benign false-alarm rate are reported together. High recall is not operationally useful if benign alerts overwhelm security analysts.

Chronological results are emphasized over random-split results because scheduled bursts and near-duplicate flows can make random evaluation optimistic. Cross-year performance is the strongest available test of network generalization.

## Reproduction

Run the notebooks from the project root in this order:

1. `00_download_data.ipynb` — required data download and setup
2. `01_eda.ipynb`
3. `01_eda_advanced_questions.ipynb`
4. `01_eda_2018_extended.ipynb`
5. `02_logistic_regression_baseline.ipynb`

Required packages include `numpy`, `pandas`, `matplotlib`, `seaborn`, `scipy`, and `scikit-learn`.

## Relevant outputs

| File | Purpose |
|---|---|
| `02_logistic_baseline_summary.csv` | Untuned baseline summary |
| `02_logistic_baseline_per_class.csv` | Untuned per-class metrics |
| `02_logistic_cross_year_detection.csv` | Baseline cross-year alarm and detection rates |

Supporting documentation includes [`PROJECT_BUSINESS_GOAL.md`](PROJECT_BUSINESS_GOAL.md), [`DATA_DICTIONARY.md`](DATA_DICTIONARY.md), and [`ATTACK_LABELS.md`](ATTACK_LABELS.md).

## Phase 2 (TODO)

Model tuning, ensemble and neural-network experiments, comparison outputs
