# Exploratory Data Analysis Guide

## Purpose

This document explains the core analysis in [`EDA/01_eda.ipynb`](../EDA/01_eda.ipynb), the hypothesis-driven Q1–Q6 analysis in [`EDA/01_eda_advanced_questions.ipynb`](../EDA/01_eda_advanced_questions.ipynb), and the detailed 2018 analysis in [`EDA/01_eda_2018_extended.ipynb`](../EDA/01_eda_2018_extended.ipynb), including why each step is performed and how it supports the network-intrusion-detection business case.

The project asks:

> How effectively can network-flow statistics distinguish benign traffic from cyberattacks and separate attack families, and how much can evaluation design overstate that performance?

The business objective is not only to obtain a high model score. A useful detector must recognize attacks that occur later or in another network, maintain a tolerable false-alarm rate, and avoid relying on identifiers or laboratory-specific shortcuts.

## Data used

| Dataset | Files | Flows | Role in the analysis |
|---|---:|---:|---|
| Improved CICIDS2017 | 5 | 2,099,976 | Detailed EDA and primary model-development data |
| Improved CSE-CIC-IDS2018 | 10 | 63,195,145 | Exact class counts and cross-year distribution comparison |

Both datasets contain the same 91-column schema. The complete 2017 dataset is loaded for detailed analysis. Because the 2018 data are approximately 35 GB, exact label counts are streamed from all files and detailed comparisons use a cached, stratified per-label sample.

The stratified 2018 sample does not preserve population class proportions. It is appropriate for comparing feature distributions within labels, but not for estimating overall accuracy or attack prevalence.

## EDA roadmap

The table below summarizes the complete exploratory workflow before the detailed explanations.

| Step | Analysis | What is done | Why it is done | **Business relevance** |
|---:|---|---|---|---|
| 1 | Source-file inventory | List every 2017 and 2018 file and measure its size. | Confirm that the expected data are present and understand processing requirements. | Missing capture files could create monitoring blind spots or misleading model results. |
| 2 | Schema compatibility | Compare columns and data structure across both years. | Verify that the datasets can support consistent preparation and cross-year comparison. | A detector cannot transfer reliably if feature definitions differ between training and deployment data. |
| 3 | Data-quality audit | Measure missing values, infinite values, constants, duplicates, data types, and memory use. | Identify conditions that could break estimators or distort analysis. | Reliable alerts require traceable and stable input data. |
| 4 | Cleaning policy | Define infinity replacement, constant removal, duplicate removal, and leakage-column exclusions. | Apply consistent and reproducible preparation without deleting legitimate attacks. | Prevents the model from learning record identifiers or laboratory-specific metadata. |
| 5 | Feature engineering | Derive packet totals, byte totals, duration, bytes per packet, direction ratios, and hour. | Convert raw measurements into interpretable traffic concepts. | Helps analysts connect model behavior with volume, duration, and traffic direction. |
| 6 | Outlier and anomaly analysis | Examine quantiles, IQR outliers, zero-duration flows, negative headers, and extreme rates. | Distinguish genuine attack behavior from measurement problems. | Avoids deleting extreme flows that may be the attacks the organization needs to detect. |
| 7 | Label balance | Count every benign, attempted, and effective-attack label in both datasets. | Measure class imbalance and identify rare classes. | Justifies macro F1 and prevents benign traffic from hiding poor attack detection. |
| 8 | Protocol and port analysis | Compare IP protocols and destination ports by traffic type and attack class. | Identify targeted services and categorical attack patterns. | Supports security interpretation while revealing possible service-layout shortcuts. |
| 9 | Attack timing | Plot benign and attack traffic in 10-minute intervals across capture days. | Identify scheduled attack bursts and temporal dependence. | Shows why random splitting can overstate future detection performance. |
| 10 | Correlation analysis | Measure feature relationships, attack correlations, and highly correlated pairs. | Detect redundant measurements and multicollinearity. | Supports simpler monitoring and prevents misleading coefficient or importance interpretation. |
| 11 | Class-level distributions | Compare log-scaled continuous features across benign traffic and attack families. | Identify visible attack signatures and overlapping classes. | Sets realistic expectations for which attack families will be easier or harder to detect. |
| 12 | PCA visualization | Project standardized, transformed features into two dimensions. | Obtain an initial visual assessment of class structure and overlap. | Reveals broad separability without claiming that the two-dimensional view is a production classifier. |
| 13 | Common cleaned table | Apply one shared label, feature, infinity, constant, and duplicate policy. | Ensure every extended analysis describes the same population used for modeling. | Makes conclusions traceable from EDA through model evaluation. |
| 14 | Stratified 2018 sample | Stream the large 2018 files and cache a bounded sample per label. | Make detailed cross-year comparisons practical without loading 35 GB repeatedly. | Enables efficient drift analysis while preserving honest limits on prevalence and accuracy estimates. |
| 15 | Attempted-attack policy | Compare benign, attempted, and effective attacks using medians, KS statistics, and distributions. | Test whether attempted attacks should be treated as benign. | Quantifies label ambiguity that can turn reasonable security alerts into apparent false positives. |
| 16 | Attack-feature separation | Calculate class-versus-benign KS statistics for every feature. | Identify the signals associated with each attack family independently of a fitted model. | Shows where detection is feasible and where weak separation may cause missed attacks. |
| 17 | Redundancy clustering | Group features using absolute Spearman correlation and select representatives. | Convert pairwise correlations into an interpretable reduced-feature design. | Can reduce collection cost, scoring complexity, and unstable explanations. |
| 18 | Shortcut testing | Fit shallow one-feature diagnostic rules using all features and environment-only features. | Determine whether apparent separation can come from laboratory hosts or service configuration. | Prevents deployment decisions based on a detector that recognizes the testbed rather than malicious behavior. |
| 19 | Near-duplicate leakage | Fingerprint approximately similar flows and compare random and chronological splits. | Measure whether train and test sets contain repeated flows from the same attack bursts. | Supports chronological evaluation as the more credible estimate of future performance. |
| 20 | Cross-year drift | Align compatible families and compare their 2017 and 2018 feature distributions. | Determine whether apparently identical attacks look different across environments. | Establishes whether within-year scores are likely to transfer to another network. |

## Step-by-step analysis

### 1. Inventory the source files

**What is done**

- Locate every 2017 and 2018 CSV file.
- Record each filename and file size.
- Calculate the number of files and total storage by dataset.

**Why it is done**

This verifies that all expected source files are present and establishes the scale of the analysis. It also explains why 2017 can be loaded in full while 2018 requires streaming and sampling.

**Business relevance**

A production data pipeline must know its expected inputs and volume. Missing capture days or incomplete files could create blind spots or misleading model performance.

### 2. Verify schema compatibility

**What is done**

- Read column names from both datasets.
- Confirm that 2017 and 2018 have the same 91-column schema.
- Display the available identifiers, traffic measurements, flags, timing fields, and label columns.

**Why it is done**

Cross-year analysis requires compatible variables. Schema validation prevents later comparisons from silently matching the wrong fields.

**Business relevance**

Stable feature definitions are required if a detector trained on one capture period will be applied to another.

### 3. Audit data quality

**What is done**

- Measure dataset shape and memory usage.
- Identify numeric and nonnumeric columns.
- Count missing values and infinite values.
- identify constant columns.
- Count duplicate traffic records while ignoring identifiers and timestamps.

**Why it is done**

Machine-learning estimators cannot safely consume infinite values, and constant fields contain no predictive information. Identifiers can make otherwise identical flows appear different, so they are excluded from the duplicate key.

**Business relevance**

Data-quality failures can produce an unstable baseline, incorrect alerts, or misleading evaluation results. Explicit checks make the processing pipeline auditable.

### 4. Define the cleaning policy

**What is done**

- Convert infinite rate values to missing and replace them with the largest finite value for that feature.
- Remove constant fields before modeling.
- Remove exact duplicate feature-and-label rows in the modeling data.
- Exclude flow IDs, IP addresses, source port, timestamp, record ID, and attempted-category code from model features.
- Preserve unusual but valid traffic values unless they violate a known field constraint.

**Why it is done**

Zero-duration flows produce infinite byte and packet rates. Replacing infinity with a finite extreme retains the idea of a very high rate without passing infinity to an estimator. Identifiers and capture fields are removed because they can reveal the experimental environment rather than attack behavior.

**Business relevance**

The objective is to learn reusable traffic patterns. A detector that memorizes laboratory hosts or row identifiers may score well in testing but fail in another network.

### 5. Engineer interpretable variables

**What is done**

Derived variables include:

- total packets;
- total payload bytes;
- duration in seconds;
- bytes per packet;
- forward-to-backward packet ratio; and
- hour of day.

**Why it is done**

The raw data split related measurements across directions and units. Derived totals and ratios make flow scale, direction, and intensity easier to interpret.

**Business relevance**

Security analysts reason about concepts such as traffic volume, packet asymmetry, connection duration, and response size. Interpretable variables help connect statistical findings with network behavior.

Time-derived fields are used only for EDA. They are excluded from modeling because the attacks were scheduled and hour-of-day would leak the experiment timetable.

### 6. Analyze outliers and measurement anomalies

**What is done**

- Calculate minimum, maximum, median, and extreme quantiles for important continuous features.
- Apply the 1.5-IQR rule as an outlier screen.
- Count zero-duration flows, negative header lengths, and infinite rates.
- Plot log-scaled distributions for highly skewed variables.

**Why it is done**

Network-flow variables are extremely skewed. Many statistical outliers are genuine flooding, scanning, or slow-DoS events. The IQR rule is therefore used to identify unusual observations, not automatically remove them.

**Business relevance**

Deleting statistical outliers indiscriminately could remove the exact attacks the detector is intended to find. Only known measurement anomalies receive corrective handling.

### 7. Measure label balance

**What is done**

- Count every 2017 label and its share of traffic.
- Display class counts on a logarithmic scale.
- Stream all 2018 files to obtain exact label counts.
- Compare label occurrence across capture days.

**Why it is done**

Benign traffic represents most observations, and several attacks have very few examples. Ordinary accuracy could therefore appear excellent even when rare attacks are never detected.

**Business relevance**

Rare attacks may still carry high operational cost. The imbalance analysis justifies using macro F1 as the primary multiclass measure because it gives every class equal importance.

### 8. Examine protocol and destination-port patterns

**What is done**

- Cross-tabulate attack labels with IP protocol.
- Identify the most common destination ports for each class.
- Compare protocol and port distributions for effective attacks versus benign or attempted traffic.

**Why it is done**

Protocol and destination port are categorical network characteristics that can provide legitimate context, such as SSH attacks targeting port 22.

**Business relevance**

The analysis identifies services exposed to each attack type. It also raises a generalization concern: a model may identify the laboratory service layout instead of malicious behavior.

### 9. Analyze attack timing

**What is done**

- Aggregate benign and effective-attack flows into 10-minute intervals.
- Plot traffic across each capture day.
- Calculate the start, end, and number of flows for each attack label.

**Why it is done**

The attacks occur in scheduled bursts. A random row split can distribute almost identical flows from one burst across training and testing.

**Business relevance**

Random evaluation can overstate the ability to detect future attacks. This finding motivates the chronological evaluation used by the Logistic Regression baseline.

### 10. Measure feature relationships and multicollinearity

**What is done**

- Calculate feature correlation with binary attack status.
- Display correlations among the 20 features most associated with attacks.
- Identify feature pairs with absolute correlation above 0.95.

**Why it is done**

The flow generator produces multiple variables that describe the same packet or timing behavior. Highly correlated columns carry overlapping information.

**Business relevance**

Redundancy can increase storage and monitoring cost and makes Logistic Regression coefficients harder to interpret because related columns divide the same signal.

### 11. Compare continuous variables across attack classes

**What is done**

- Fold `- Attempted` labels into `BENIGN`, matching the baseline policy.
- Sample up to 5,000 observations per class.
- Plot log-scaled distributions for duration, packet counts, packet sizes, rates, and TCP-window fields.

**Why it is done**

Class-level plots reveal whether an attack has a distinct flow signature and where different attacks overlap.

**Business relevance**

Clearly separated classes should be easier to detect. Overlapping web and infiltration classes require more cautious expectations and more detailed error analysis.

### 12. Visualize class separability with PCA

**What is done**

- Apply a signed `log1p` transformation.
- Standardize the sampled numeric features.
- project the data into two principal components.
- Color observations by traffic class.

**Why it is done**

PCA provides a compact visual summary of whether the feature space contains broad class structure. It is exploratory and is not used as the final classifier.

**Business relevance**

The visualization shows that major flooding and scanning attacks form clearer groups, while rare web and infiltration classes overlap more heavily.

### 13. Create a common cleaned analysis table

**What is done**

- Apply the baseline feature exclusions.
- replace infinite values with finite maxima.
- remove constant features.
- fold attempted attacks into the baseline `BENIGN` label.
- remove duplicate feature-and-label records.

**Why it is done**

The extended analyses must describe the same observations and features used by the Logistic Regression baseline. A single preparation step avoids inconsistent populations across notebook sections.

**Business relevance**

Consistent definitions make findings traceable from EDA through modeling and prevent apparently conflicting results caused by different cleaning rules.

### 14. Build the stratified 2018 sample

**What is done**

- Stream 2018 files in chunks when the cached sample is absent.
- Use random reservoir-style keys to retain a uniform sample per label.
- Keep up to 200,000 benign flows and 20,000 flows for each other label.
- Cache the result as a compressed CSV for reproducible reruns.

**Why it is done**

Loading all 63 million 2018 rows for every distributional comparison would be unnecessarily expensive. Per-label sampling preserves sufficient observations for class-conditional comparisons.

**Business relevance**

The approach makes cross-year diagnostic analysis practical while explicitly preventing the sample from being misused for population prevalence or overall accuracy.

### 15. Evaluate the attempted-attack labeling policy

**What is done**

- Compare benign, attempted, and effective-attack flow medians.
- Use two-sample Kolmogorov-Smirnov statistics to compare the three groups across all features.
- Plot empirical distributions for representative variables.

**Why it is done**

The baseline treats unsuccessful attack attempts as benign. The analysis tests whether those rows actually resemble ordinary benign traffic.

**Finding**

Attempted flows are approximately equally distant from benign and effective-attack traffic. They retain attacker characteristics such as target ports, SYN behavior, and TCP-window values but often contain no payload.

**Business relevance**

Treating attempted attacks as benign creates label noise and can count reasonable security alerts as false positives. Future work should compare three policies: fold into benign, remove the rows, or model attempted activity separately.

### 16. Identify features that separate each attack from benign traffic

**What is done**

- Sample up to 20,000 flows per attack class and 50,000 benign flows.
- Calculate a KS statistic for every class-feature pair.
- Display the 20 features with the strongest average separation.

**Why it is done**

This measures which traffic characteristics provide detection signal without depending on a particular machine-learning model.

**Finding**

Every attack class has at least one feature with KS of at least 0.87 against benign traffic. Server-to-client payload measurements are especially strong general separators.

**Business relevance**

The results show that flow data contain useful detection information, while identifying slow-DoS and web attacks as more difficult and potentially less stable classes.

### 17. Cluster redundant features

**What is done**

- Compute absolute Spearman correlation on a 100,000-flow sample.
- Cluster features using distance `1 - |correlation|`.
- Cut the hierarchy at absolute correlation of 0.9.
- Select one representative feature per group using average benign-versus-attack separation.

**Why it is done**

Spearman correlation is appropriate for skewed variables and captures common rank behavior. Clustering reveals groups rather than listing thousands of pairwise correlations.

**Finding**

The 83 model features collapse to 37 groups at absolute correlation of at least 0.9. Several variables are exact or near-exact copies.

**Business relevance**

A reduced feature set could lower collection and scoring cost, simplify explanations, and reduce dependence on redundant measurements.

### 18. Test single-feature and environment shortcuts

**What is done**

- Train depth-2 decision trees using one feature at a time for each attack-versus-benign problem.
- Repeat the comparison using only environment-related features: destination port, TCP initial-window values, minimum forward segment size, and protocol.

**Why it is done**

If one laboratory-specific field produces near-perfect classification, a complex model may achieve excellent scores without learning general attack behavior.

**Finding**

One feature reaches F1 of at least 0.95 for 11 of 15 attack classes. Environment features alone reach F1 of at least 0.90 for 10 classes.

**Business relevance**

The diagnostic rules show that a detector can identify which laboratory host or service was attacked. This can fail when IP stacks, services, or network architecture change. The baseline should therefore be repeated without environment fields and compared with the original specification.

### 19. Measure near-duplicate leakage

**What is done**

- Signed-log transform and round every model feature into approximate 10% bins.
- Hash the rounded feature vector and label into a flow fingerprint.
- Calculate the share of test flows with a matching training fingerprint.
- Compare random and within-class chronological splits.

**Why it is done**

Scripted attacks generate many almost identical flows. Near-duplicate exposure measures how much a model can benefit from seeing the same burst pattern during training.

**Finding**

For DoS Slowhttptest, near-duplicate test exposure falls from approximately 39% under a random split to 3% under a chronological split. For Slowloris it falls from approximately 36% to 14%.

**Business relevance**

Chronological evaluation better represents detection of later activity. Random-split performance should be treated as a benchmark, not the primary deployment estimate.

### 20. Measure cross-year feature drift

**What is done**

- Map compatible 2017 and 2018 labels into common attack families.
- Compare each family across years using feature-level KS statistics.
- Plot representative empirical distributions.

**Why it is done**

An attack label can remain the same while duration, packet counts, response sizes, operating systems, and victim content change.

**Finding**

Benign traffic shifts moderately, while several attack families shift strongly. SSH brute force changes in timing and duration; Hulk and GoldenEye retain similar request sizes but receive different server responses.

**Business relevance**

Within-year test performance does not establish production readiness. Cross-year or cross-network testing is necessary to measure generalization.

## Decisions produced by the EDA

The consolidated analysis establishes the following modeling decisions:

1. Fold attempted attacks into `BENIGN` for the initial baseline, but identify the resulting label ambiguity as a limitation.
2. Exclude identifiers, IP addresses, source port, timestamps, and attempted-category codes.
3. Convert infinite rates to finite values and remove constant features.
4. Remove exact feature-and-label duplicates before splitting.
5. Retain legitimate statistical outliers.
6. Apply a signed `log1p` transformation and standardization for Logistic Regression.
7. Report both random and chronological evaluations, with chronological results as the headline.
8. Use macro F1 as the primary multiclass metric.
9. Treat rare-class results cautiously.
10. Include cross-year model evaluation before making generalization claims.
11. Test reduced-feature and environment-feature-free Logistic Regression baseline variants.

## How the EDA addresses the business case

The analysis supports the business case in four ways:

### Detection feasibility

Network-flow statistics clearly contain information that separates many attacks from benign activity. This supports continued development of an automated flow-based detector.

### Operational risk

Class imbalance, rare attacks, attempted-attack labels, and measurement anomalies show where false alerts or missed attacks may arise.

### Evaluation credibility

Scheduled bursts and near-duplicate flows demonstrate why random splitting can produce optimistic results. Chronological evaluation provides a more credible estimate of future-period performance.

### Deployment readiness

Environment shortcuts and strong cross-year drift show that good CICIDS2017 scores do not guarantee success in another network. Cross-year recall and false-alarm analysis are required before deployment claims.

## Relationship to the modeling workflow

The EDA and baseline notebooks should be run in this order:

1. `EDA/01_eda.ipynb` — complete the core, grading-aligned EDA and initial 2018 inventory.
2. `EDA/01_eda_advanced_questions.ipynb` — investigate label ambiguity, separability, redundancy, shortcuts, leakage, and cross-year drift.
3. `EDA/01_eda_2018_extended.ipynb` — perform the detailed CSE-CIC-IDS2018 audit and EDA.
4. `baseline/02_logistic_regression_baseline.ipynb` — establish an interpretable minimum benchmark and cross-year check.
5. `advanced-modeling/02_logistic_regression_tuning.ipynb` — test whether leakage-safe regularization and weighting choices improve the baseline.

## Reproduction

Start Jupyter from the project root and open the notebooks under `EDA/` so relative data and cache paths resolve correctly. Required Python packages include `numpy`, `pandas`, `matplotlib`, `seaborn`, `scipy`, and `scikit-learn`.

The notebook reads raw CSV files without modifying them. Generated caches contain label counts and the stratified 2018 sample, allowing later runs to avoid rescanning the full 2018 dataset.
