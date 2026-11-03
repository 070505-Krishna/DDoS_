# 03. Core Algorithms and Math

## 1. Problem formulation

Global objective over K clients:

```
min_w  F(w) = sum_{k=1..K} p_k * F_k(w)
F_k(w) = (1/n_k) * sum_{i in D_k} loss(w; x_i, y_i)
p_k = n_k / n,   n = sum n_k
```

Centralized training would minimize the loss over the pooled data. In FL, each `F_k` is evaluated only at client k.

## 2. FedSGD (baseline)

- Each selected client computes one gradient on its local data at `w_t` and sends it.
- Server: `w_{t+1} = w_t - eta * sum_k p_k * g_k`
- Equivalent to large-batch SGD. Very communication heavy (one round per gradient step).

## 3. FedAvg (the standard)

Client update (E local epochs, batch B, learning rate eta):
```
w_k <- w_t
for epoch in 1..E:
    for batch b in local data (size B):
        w_k <- w_k - eta * grad loss(w_k; b)
send w_k (or w_k - w_t) and n_k
```
Server aggregation:
```
w_{t+1} = sum_{k in S_t} (n_k / n_S) * w_k
```

Hyperparameters: `C` (fraction), `E`, `B`, `eta`, `T`.

Behaviour:
- IID data: works nearly as well as centralized with far fewer communication rounds.
- Non-IID data: **client drift** (local models move toward local optima), slow/unstable convergence, accuracy drop.

Tips: tune E carefully (large E + non-IID = drift), use learning-rate decay, consider server momentum (FedAvgM).

## 4. Handling heterogeneity: main algorithms

### 4.1 FedProx (Li et al., 2018)
Adds a proximal term to the local objective:
```
min_w  F_k(w) + (mu/2) * ||w - w_t||^2
```
- Keeps local models near the global one, tolerant to partial work (variable E).
- `mu` is tuned (0.001-1 typical). `mu = 0` equals FedAvg.

### 4.2 SCAFFOLD (Karimireddy et al., 2020)
Uses control variates `c` (server) and `c_k` (client) to correct drift:
```
local step: w <- w - eta * (g - c_k + c)
```
- Better on non-IID, fewer rounds, but doubles communication and requires client state (hard in cross-device).

### 4.3 FedNova
Normalizes updates by number of local steps to remove objective inconsistency when clients do different work.

### 4.4 FedAvgM / FedOpt (Reddi et al., 2020)
Treat the averaged update as a pseudo-gradient and apply server-side optimizers: FedAdam, FedYogi, FedAdagrad. Often a strong, simple improvement.

### 4.5 MOON, FedDyn, FedDC
Contrastive or dynamic regularization to align local and global representations.

### 4.6 FedBN
Keep BatchNorm layers local; helps with feature-shift non-IID.

## 5. Personalization

Goal: a good model **per client** instead of one global model.

| Method | Idea |
|---|---|
| Fine-tuning | Train global, then fine-tune locally |
| Local head (FedPer, LG-FedAvg) | Shared body, private final layers |
| Ditto | Train a personal model regularized toward global |
| pFedMe | Moreau-envelope based personalization |
| Per-FedAvg (MAML) | Meta-learn an initialization that adapts quickly |
| Clustered FL (IFCA, CFL) | Group similar clients, one model per cluster |
| Mixture / multi-task (MOCHA) | Learn related models jointly |

Cybersecurity relevance: each network has different "normal" traffic, so personalized anomaly detectors often beat one global model.

## 6. Aggregation rules (basic and robust)

| Rule | Formula / idea | Robust to |
|---|---|---|
| Mean (FedAvg) | weighted average | none |
| Coordinate-wise Median | median per parameter | few outliers (< 50%) |
| Trimmed mean | drop top/bottom beta fraction per coordinate, average rest | up to beta fraction Byzantine |
| Krum | choose update closest to its n-f-2 nearest neighbours | f Byzantine of n (n >= 2f+3) |
| Multi-Krum | average the m best Krum-scored updates | f Byzantine |
| Bulyan | Krum selection + trimmed mean | stronger, needs n >= 4f+3 |
| Geometric median (RFA) | minimize sum of distances | moderate |
| Norm clipping | clip update L2 norm to threshold | scaling/boosting attacks |
| FLTrust | server holds small clean dataset; trust score = cosine to server update | many malicious clients |
| FoolsGold | down-weight clients whose updates are too similar (Sybils) | Sybil collusion |

Krum score for update i: sum of squared distances to its closest `n - f - 2` neighbours; pick the minimum.

## 7. Federated optimization theory (what to know)

- Convergence for convex/strongly convex objectives holds with decaying learning rate; error includes terms due to **data heterogeneity** (bounded gradient dissimilarity) and **local steps** (client drift).
- With partial participation, variance grows as the participating fraction shrinks.
- Non-IID rate (rough intuition): FedAvg converges as `O(1/sqrt(KET))` for non-convex with sufficient assumptions, but heterogeneity adds a bias term proportional to `E` and gradient dissimilarity.
- You do not need proofs for a project; know the **intuition** and cite the standard analyses (Li et al. "On the Convergence of FedAvg on Non-IID Data", Karimireddy et al.).

## 8. Unsupervised and anomaly-detection oriented FL

- **Federated autoencoder:** each client trains an autoencoder on benign traffic; anomaly score = reconstruction error. Aggregate with FedAvg. Strong for zero-day detection (e.g. N-BaIoT style).
- **Federated one-class / isolation approaches:** share summary statistics or thresholds.
- **Federated clustering (K-means, DBSCAN variants):** share centroids.
- **Federated GANs:** generate synthetic attack traffic to balance classes.
- **Federated contrastive / self-supervised:** SimCLR-style with unlabeled logs.

## 9. Learning with limited labels / class imbalance (very common in security)

- Weighted loss, focal loss, class-aware sampling.
- Federated SMOTE / synthetic data from local generative models.
- Server-side small public/synthetic data for calibration.
- Evaluate with F1, recall per class, PR-AUC, not accuracy alone.

## 10. Federated distillation

- Clients and server exchange **logits on a shared unlabeled public set** rather than weights (FedMD, FedDF).
- Benefits: supports heterogeneous architectures and reduces communication; less direct weight leakage.

## 11. Federated analytics & threat-intel sharing

- Compute global statistics (e.g. frequency of malicious IPs, domain counts) using secure aggregation + DP (e.g. Count-Min sketches, heavy hitters via histograms).
- Practical for sharing indicators of compromise (IoCs) without exposing all internal traffic.

## 12. Algorithm selection guide

| Situation | Try first |
|---|---|
| Baseline, mild non-IID | FedAvg |
| Strong non-IID, stragglers | FedProx, FedAvgM/FedAdam |
| Strong non-IID, cross-silo, stateful clients | SCAFFOLD |
| Each client wants own best model | Fine-tuning, Ditto, FedPer |
| Few attack labels | Federated autoencoder |
| Suspect malicious clients | Median / Trimmed mean / Krum / FLTrust |
| Need formal privacy | DP-FedAvg (+ SecAgg) |
| Heterogeneous model sizes | FedMD / FedDF / HeteroFL |

## 13. Pseudocode: FedProx local step

```
for each batch (x, y):
    loss = CE(model(x), y) + (mu/2) * sum((p - p_global)^2 for p in params)
    loss.backward(); opt.step()
```

## 14. Exam-style points

- FedAvg = local SGD + weighted averaging; saves communication by doing more local work.
- Non-IID causes client drift; fixes: proximal term, control variates, server optimizers, personalization.
- Robust aggregation tradeoff: more robustness often means lower accuracy on benign non-IID data (benign heterogeneity looks like an attack).
