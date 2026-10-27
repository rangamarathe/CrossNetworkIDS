# Exploratory data analysis findings

## Executive summary

The exploratory analysis shows that network-flow statistics contain useful attack signals, but strong performance inside CICIDS2017 can overstate readiness for another network. The main risks are class imbalance, extreme and redundant measurements, scheduled attack bursts, laboratory-specific shortcuts, ambiguous attempted-attack labels, and substantial cross-year drift.

The data support building a binary detector for effective attacks and a secondary multiclass analysis of attack families. They do not support a claim that a model trained on these datasets will detect every attack in an unseen production environment.

## Data analyzed

| Dataset | Source files | Flows | Primary analytical role |
|---|---:|---:|---|
| CICIDS2017 improved | 5 | 2,099,976 | Core EDA and baseline model development |
| CSE-CIC-IDS2018 improved | 10 | 63,195,145 | Extended EDA and cross-network comparison |

Both datasets contain the same 91-column schema. CICIDS2017 can be analyzed in memory. The approximately 36 GB 2018 collection requires chunked processing, cached aggregates, and a 378,952-row stratified sample for distribution-heavy analysis.

The 2018 sample contains observations from 25 labels. It intentionally changes the natural class mixture and therefore cannot be used to estimate real prevalence, overall accuracy, or production precision.

## Findings at a glance

| Area | Supporting evidence | Finding | Business implication |
|---|---|---|---|
| Data compatibility | Both years use the same 91-column schema. | Common preparation and cross-year comparison are feasible. | Matching schemas reduce integration work, but matching column names do not guarantee stable feature distributions. |
| Class imbalance | Benign traffic is 75.36% of 2017 and 93.92% of 2018. Several 2017 attack classes contain fewer than 100 effective examples. | Accuracy can hide poor detection of rare attacks. | Use macro F1, attack recall, false-alarm rate, and per-class metrics. |
| Data quality | The audits found infinite rates, invalid negative header lengths, constants, and duplicate records. The 2018 full audit found 57 infinite rate values. | Explicit cleaning is required before modeling. | A reproducible preprocessing pipeline prevents failures and inconsistent scoring. |
| Heavy tails | The examined 2017 features have approximately 8%–21% IQR-defined outliers. | Many extreme values are legitimate attack behavior. | Retain plausible extremes; transform and scale them instead of deleting them automatically. |
| Protocol and service concentration | Effective 2017 attacks are 99.937% TCP. Many attack classes target one service port. | Protocol and destination port are strong signals inside the laboratory. | Treat them as context, not proof of maliciousness, and verify them across networks. |
| Scheduled events | All 158,468 DoS Hulk flows occur in one approximately 18-minute period. | Random splits can place closely related event flows in both training and testing. | Emphasize chronological and cross-network evaluation. |
| Redundant features | The 2017 sample has 72 numeric feature pairs with absolute Pearson correlation above 0.95. The 83 model features form 37 groups at absolute Spearman correlation of at least 0.90. | Several columns measure nearly the same behavior. | Use regularization or compact feature representatives before interpreting coefficients. |
| Attack separability | Every 2017 attack class has at least one class-versus-benign KS statistic of 0.87 or higher; 12 of 15 reach at least 0.95. | Individual attacks can appear easy to separate in the source testbed. | High within-dataset scores require shortcut and transfer checks. |
| Laboratory shortcuts | One feature achieves F1 of at least 0.95 for 11 of 15 classes. Environment-only fields achieve F1 of at least 0.90 for 10 classes. | Models can identify a laboratory host, port, or TCP configuration instead of attack behavior. | Validate feature dependence and performance on another network before deployment. |
| Attempted attacks | Median KS distance is 0.375 from benign and 0.390 from effective attacks; attempted flows are closer to attacks on 43 of 83 features. | Attempted flows are neither ordinary benign traffic nor equivalent to effective attacks. | The label policy affects apparent false positives and should be tested explicitly. |
| Cross-year drift | Benign KS values are approximately 0.11–0.29, while several compatible attack families show near-complete changes in important features. | The same named attack can look different in another year or network. | Strong within-year performance does not establish deployment readiness. |
| Low-dimensional structure | The first two PCA components explain about 55% of transformed 2017 variation and 51.3% of 2018 variation. | Some classes form visible structures, but benign and attack traffic overlap. | Multivariate modeling is justified, but PCA is not evidence of classifier accuracy. |

## Detailed findings

### 1. The datasets are structurally compatible but behaviorally different

The two datasets use the same 91 columns and the same flow-extraction approach. This permits a shared cleaning and feature-preparation pipeline. However, several feature distributions and label proportions differ substantially between years.

For example, `CWR Flag Count` is nonzero in approximately 0.04% of 2017 records and 29% of 2018 records. `ECE Flag Count` changes from approximately 0.03% to 29%. These changes likely reflect differences in hosts or network configuration rather than changes in maliciousness.

**Business implication:** Schema compatibility is necessary for transfer, but distribution monitoring and cross-network validation remain necessary.

### 2. Class imbalance makes accuracy insufficient

CICIDS2017 is 75.36% benign. Most effective attacks are concentrated in Portscan, DoS Hulk, DDoS, and Infiltration-Portscan, while Heartbleed, Infiltration, and several web attacks have very few effective examples.

CSE-CIC-IDS2018 is even more benign-dominated at 93.92%. Its label mixture also differs from 2017. A model can therefore achieve high accuracy by favoring benign traffic and common attack families.

**Modeling action:** Use macro F1 as the primary multiclass metric. Report attack recall, attack precision, false-alarm rate, and per-family results alongside it. Apply weighting or resampling only to training data.

### 3. Extreme values often contain the attack signal

Duration, packet counts, byte volumes, inter-arrival times, and packet rates span several orders of magnitude. The IQR method marks approximately 8%–21% of observations as outliers for the examined 2017 variables. The 2018 distributions have similarly substantial tails.

Floods, scans, and slow denial-of-service attacks can naturally produce extreme rates, volumes, or durations. Removing all statistical outliers would erase part of the behavior the detector needs to learn.

**Cleaning action:** Replace infinite values, isolate clearly invalid measurements, and use transformations and scaling. Do not delete plausible extremes solely because they exceed an IQR threshold.

### 4. Protocol and destination port are useful but risky

Effective CICIDS2017 attacks are almost entirely TCP: 505,115 flows, or 99.937%. Another 316 effective attack flows use ICMP, UDP, or other protocols, so the data do not support the claim that attacks occur only over TCP.

Destination ports align closely with the generated scenarios. Benign traffic frequently uses ports 53, 443, and 80. Several attacks target a single service, including HTTP on port 80, SSH on port 22, FTP on port 21, and web traffic on port 8080.

**Business implication:** These fields help identify affected services but may encode the research network's service layout. Production rules must not treat a port or protocol as sufficient evidence of an attack.

### 5. Scheduled attack bursts can inflate random-split results

Attacks occur in concentrated laboratory windows. For example, all 158,468 DoS Hulk flows occur during one approximately 18-minute period. An 80/20 random split can put roughly 126,000 flows from that event into training and 32,000 related flows into testing.

Removing timestamps does not eliminate the dependency. Neighboring flows still share rates, ports, TCP windows, and tool-specific behavior. Near-duplicate analysis confirms the concern: DoS Slowhttptest exposure falls from approximately 39% with a random split to 3% with a chronological split, while Slowloris falls from approximately 36% to 14%.

**Evaluation action:** Exclude schedule fields from predictors and emphasize within-class chronological evaluation. Use cross-year testing as the stronger available generalization check.

### 6. Feature redundancy complicates interpretation

The 2017 analysis contains 88 varying numeric variables and 72 pairs with absolute Pearson correlation above 0.95. After aligning the model feature set, 83 features collapse into 37 groups at absolute Spearman correlation of at least 0.90 and 22 groups at 0.70.

Some columns are exact or near-exact copies. `Fwd URG Flags` and `URG Flag Count` are identical, while `Fwd Packet Length Mean` and `Fwd Segment Size Avg` differ only through rounding. Derived variables can also duplicate dataset-provided measurements.

**Modeling action:** Use L2 regularization for the Logistic Regression baseline and test a compact representative feature set. Interpret coefficient groups rather than assigning causal importance to one correlated column.

### 7. Attack families have different signatures

Port scanning tends to produce short flows, few responses, small packets, and high forward packet rates. Slowloris and Slowhttptest produce long durations, larger inter-arrival times, and lower rates. Flooding and denial-of-service families show different combinations of packet volume, response size, and rate.

Every 2017 attack class has at least one feature with a KS statistic of 0.87 or more against benign traffic, and 12 of 15 classes reach at least 0.95. Backward payload measurements are strong general separators because attacks elicit characteristic server responses. Destination port is the strongest separator for FTP-Patator, SSH-Patator, Botnet, and Infiltration.

**Modeling action:** Use multivariate models and report per-family performance. Do not assume that one feature or one threshold detects every attack type.

### 8. Apparent separability often reflects the laboratory

A one-feature threshold reaches F1 of at least 0.95 for 11 of 15 attack classes. More importantly, an environment-only feature reaches F1 of at least 0.90 for 10 classes. `Bwd Init Win Bytes` often identifies the victim server's TCP configuration. Destination port separates Botnet and Infiltration, while an attacker TCP-window field nearly identifies the internal port scan by itself.

These signals may recognize which host or service was attacked rather than the behavior of the attack.

**Validation action:** Compare models with and without environment-related features, inspect permutation importance, and evaluate on an unseen network. A lower within-lab score may be acceptable if external recall becomes more stable.

### 9. Attempted attacks create label ambiguity

Flows labeled as attempted attacks are about equally distant from benign and effective attacks: median KS is 0.375 versus benign and 0.390 versus effective attacks. They are closer to attacks on 43 of 83 features and share attacker ports, TCP windows, and SYN patterns, but often contain no payload.

The current baseline folds attempted attacks into benign traffic because the project target is effective malicious activity. This policy creates attack-shaped records inside the benign class. In 2018, all 298,874 FTP brute-force flows are marked attempted, making the policy especially consequential.

**Future experiment:** Compare three policies: fold attempted flows into benign, remove them, or model them as a separate class. Evaluate how each policy changes attack recall and false alarms.

### 10. Compatible attacks drift across years

Benign traffic changes moderately between years, with selected KS values of approximately 0.11–0.29. Several compatible attacks change much more:

- SSH brute-force flows retain their port and similar packet sizes, but 2018 flows are about 30 times shorter, producing near-complete timing drift.
- DoS Hulk and DoS GoldenEye retain similar forward requests, with forward-byte KS no greater than 0.06, but their server-response sizes have KS of 1.00 because the victim web content changed.
- DDoS changes in duration, packet counts, and server responses while retaining the attacker's TCP-window signature.
- DoS Slowloris and Infiltration-Portscan are among the more stable compatible families.

**Business implication:** A model can lose attack recall or create excessive benign alerts after moving to another customer network. Training diversity, feature-drift monitoring, and controlled retraining are deployment requirements.

## Implications for the baseline model

The EDA supports the following baseline design:

1. Predict effective attack versus benign or attempted traffic for the primary binary task.
2. Exclude identifiers, timestamps, day, and derived schedule variables from predictors.
3. Replace infinite values and handle invalid measurements explicitly.
4. Preserve plausible heavy-tailed behavior and apply a signed-log transformation.
5. Standardize numeric features and use L2 regularization to manage correlated predictors.
6. Use chronological results as the main within-year estimate.
7. Report macro F1 for multiclass analysis and attack recall plus false-alarm rate for binary detection.
8. Treat cross-year testing as required evidence, not an optional extension.

## What the EDA establishes

The EDA establishes that the datasets contain learnable attack patterns and are suitable for a controlled cross-network experiment. It also establishes that several patterns are tied to particular hosts, ports, tools, schedules, and server responses.

The EDA does not establish universal attack coverage, production false-alarm rates, or generalization to an unseen third network. Those claims require representative customer traffic, prospective evaluation, and continued drift monitoring.

## Source notebooks

- `EDA/01_eda.ipynb` contains the core grading-aligned EDA and programmatic findings.
- `EDA/01_eda_advanced_questions.ipynb` contains label ambiguity, feature separation, redundancy, shortcut, leakage, and cross-year drift analysis.
- `EDA/01_eda_2018_extended.ipynb` contains the detailed CSE-CIC-IDS2018 audit and sampled distribution analysis.
