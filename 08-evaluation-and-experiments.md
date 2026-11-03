# 08. Evaluation, Experimental Design, and Reproducibility

## 1. What to measure

### 1.1 Model quality (security-appropriate)
| Metric | Why |
|---|---|
| Accuracy | Only meaningful on balanced data |
| Precision, Recall, F1 (macro, weighted, per class) | Handles imbalance; per-class recall shows missed attacks |
| False Positive Rate (FPR) | Alert fatigue in SOCs |
| TPR @ fixed FPR (e.g. 1%, 0.1%) | Operational comparison |
| ROC-AUC, PR-AUC | Threshold-independent; PR-AUC better for imbalance |
| Confusion matrix | Which attacks confuse |
| Matthews correlation coefficient | Single balanced score |
| Detection delay | Time to detect after attack starts |

### 1.2 FL-specific
| Metric | Meaning |
|---|---|
| Rounds to target accuracy | Convergence speed |
| Communication cost | Total bytes up/down (model size x participants x rounds, minus compression) |
| Computation per client | FLOPs, wall-clock time, energy |
| Fairness | Variance of per-client accuracy; worst-client accuracy |
| Personalization gain | Local fine-tuned vs global per client |
| Participation robustness | Accuracy under dropout/stragglers |

### 1.3 Security and privacy
| Metric | Meaning |
|---|---|
| Main-task accuracy (MTA) under attack | Utility remaining |
| Attack Success Rate (ASR) | Fraction of triggered/target inputs misclassified as attacker wants |
| Accuracy drop | Untargeted poisoning damage |
| Backdoor persistence | ASR after attacker stops (rounds) |
| Detection rate of malicious clients | TPR/FPR of the defense's filter |
| Reconstruction quality (PSNR, SSIM, MSE) | Gradient inversion leakage |
| MIA AUC / advantage | Membership leakage (0.5 AUC = no leakage) |
| (epsilon, delta) | DP guarantee with accountant specifics |
| Overhead of SecAgg/HE | Time and bytes versus plain FedAvg |

## 2. Experimental design

### 2.1 Baselines (always include)
1. **Centralized** (upper bound).
2. **Local-only** (each client alone; lower bound, shows FL value).
3. **FedAvg** (standard).
4. Strong FL baselines relevant to your claim (FedProx, FedAdam, SCAFFOLD; Krum/Median/Trimmed mean/FLTrust if doing security).
5. Recent state-of-the-art from the last 2-3 years in your sub-area.

### 2.2 Factors to vary (ablations)
| Factor | Typical values |
|---|---|
| Number of clients K | 5, 10, 50, 100 |
| Participation C | 0.1, 0.3, 1.0 |
| Local epochs E | 1, 2, 5, 10 |
| Non-IID alpha (Dirichlet) | 0.1, 0.5, 1, 10, 100 |
| Rounds T | until convergence (plot curves) |
| Malicious fraction | 0, 10, 20, 30, 40% |
| Attack strength | scale factor, poison rate, trigger size |
| DP noise multiplier | 0, 0.3, 0.5, 1, 2 |
| Compression | none, 8-bit, top-k 1-10% |
| Model size/architecture | MLP vs CNN vs LSTM |

Change one factor at a time first, then run important interactions.

### 2.3 Data splits (avoid leakage)
- Train / validation / test; the test set is never seen by any client or hyperparameter search.
- Prefer **temporal** or **scenario-based** splits for security data.
- Hold out entire attack types or devices for "unseen attack" generalization tests.
- Deduplicate between train and test; the most common source of unrealistically high IDS accuracy.

### 2.4 Statistical rigor
- 3-5 seeds minimum (10 if cheap); report mean +/- std or confidence intervals.
- Use paired tests (Wilcoxon signed-rank) or bootstrap CIs when claiming improvement.
- Keep hyperparameter tuning budget equal across methods.
- Report the full config, hardware, and wall-clock times.

## 3. Simulating non-IID properly

- Label skew: Dirichlet partition (see file 07); report alpha.
- Quantity skew: client sizes follow lognormal/Dirichlet.
- Feature skew: apply client-specific scaling/shift, or partition by source subnet/device.
- Temporal skew: clients see different time windows.
- Report a **heterogeneity measure** (e.g. average KL divergence of client label distributions from global, or gradient dissimilarity).
- Show results at multiple heterogeneity levels, not one.

## 4. Evaluating attacks and defenses

1. Fix the threat model (file 04).
2. Run attack without defense: demonstrate damage (baseline).
3. Run each defense without attack: measure the **cost on benign performance**, especially under non-IID.
4. Run each defense under attack: report MTA and ASR.
5. Run an **adaptive attack** against each defense (attacker aware of the rule: e.g. stay within clipping bound, mimic benign statistics, optimize against Krum selection).
6. Vary attacker fraction, participation of attackers per round (always vs occasionally), and attack timing (early vs late rounds; attacks late in training often persist more).
7. Report overhead and failure cases.

Visualizations: ASR/MTA vs round; heatmap of defense vs attack; line plot ASR vs malicious fraction; update-space t-SNE/PCA showing benign vs malicious clusters; per-client contribution weights.

## 5. Evaluating privacy

- Run actual attacks (DLG, MIA) with and without defenses.
- DP: plot **utility vs epsilon** curve (accuracy/F1 and per-class recall as epsilon varies, include epsilon = infinity baseline).
- State the DP unit (record-level or client-level), clipping norm, sampling rate, number of rounds, accountant type.
- Warn about **rare-class degradation** under DP, which matters for rare attacks.

## 6. Systems evaluation (for deployment-oriented projects)

- Measure round latency breakdown: broadcast, compute, upload, aggregate, SecAgg/HE overhead.
- Use real devices (Raspberry Pi, Jetson, Docker, cloud VMs) with network shaping (`tc netem`) for latency and bandwidth.
- Report model size, memory footprint, inference latency and throughput (packets/flows per second), CPU/GPU utilization, energy.
- Test failures: client dropout rate 10-50%, stragglers, reconnects.

## 7. Common mistakes (reviewers notice)

1. Reporting only accuracy on imbalanced IDS data.
2. Test-set leakage (duplicates, identifiers, random splits of temporally correlated flows).
3. Evaluating on IID-only splits and claiming robustness.
4. Comparing against weak or untuned baselines.
5. Non-adaptive attacks only.
6. No threat model or unrealistic fraction of malicious clients (e.g. > 50% without justification).
7. Claiming "privacy-preserving" without DP/SecAgg quantification.
8. Single seed results.
9. Ignoring communication/computation overhead.
10. Tuning hyperparameters on the test set.

## 8. Reproducibility checklist

- [ ] Code and configs public (GitHub), tagged version
- [ ] Dataset version, preprocessing script, partition indices/seed
- [ ] Software versions (`pip freeze`), hardware, OS
- [ ] Seeds and number of runs
- [ ] Hyperparameter search space and selection method
- [ ] Raw results (CSV/JSON) and plotting scripts
- [ ] Ethics: data licenses, responsible disclosure for attack code

## 9. Reporting template (results table)

| Method | Alpha | Mal% | Acc | Macro-F1 | FPR | ASR | Rounds-to-90% | MB/round | eps |
|---|---|---|---|---|---|---|---|---|---|
| Centralized | - | 0 | | | | | | | inf |
| Local-only | 0.3 | 0 | | | | | | | - |
| FedAvg | 0.3 | 0 | | | | | | | inf |
| FedAvg | 0.3 | 20 | | | | | | | inf |
| Median | 0.3 | 20 | | | | | | | inf |
| Ours | 0.3 | 20 | | | | | | | |

## 10. Ethics, responsible research, and legal notes

- Use public or synthetic datasets; do not run attacks against real systems or others' data without authorization.
- Attack code in papers: follow responsible disclosure; describe defenses alongside.
- Document data licenses (some datasets restrict commercial use).
- When using real organisational data, get written approvals and anonymization review.
