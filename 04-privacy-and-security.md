# 04. Privacy and Security in Federated Learning

FL is **not private by default**. This file is the heart of a cybersecurity research project: threat model, attacks, defenses, and trade-offs.

## 1. Threat model template (write this in every paper)

Specify:
1. **Adversary identity:** malicious client(s), curious server, external eavesdropper, colluding set.
2. **Adversary goal:** degrade accuracy (untargeted), implant behaviour (targeted/backdoor), learn private data (privacy), free-ride, evade detection.
3. **Adversary knowledge:** white-box (knows model/aggregator), gray-box, black-box.
4. **Adversary capability:** fraction of compromised clients `f`, data control, model-update control, number of rounds active, collusion, Sybil creation.
5. **Setting:** cross-device vs cross-silo, synchronous, participation rate.

## 2. Attack taxonomy

```
Attacks on FL
|-- Integrity (training-time)
|   |-- Data poisoning (label flipping, clean-label, dirty-label)
|   |-- Model poisoning (scaling, sign flipping, gaussian noise, crafted updates)
|   |-- Backdoor (trigger-based, semantic, distributed, edge-case)
|   |-- Sybil attacks
|   `-- Free-riding
|-- Privacy (inference-time or update-time)
|   |-- Gradient/model inversion (DLG, iDLG, Inverting Gradients)
|   |-- Membership inference
|   |-- Property inference
|   `-- Attribute / label inference
|-- Availability
|   |-- DoS (stalling rounds, huge updates)
|   `-- Straggler / resource exhaustion
|-- Evasion (test-time)
|   `-- Adversarial examples vs the deployed global model
`-- Server-side / infrastructure
    |-- Malicious server (model tampering, trap weights)
    `-- Communication attacks (MITM, replay)
```

## 3. Integrity attacks in detail

### 3.1 Data poisoning
- **Label flipping:** attacker flips labels (e.g. malicious -> benign) in local data. Simple, no model access needed.
- **Dirty-label vs clean-label:** clean-label keeps labels correct but crafts inputs.
- Effect in security: IDS misses a target attack class.

### 3.2 Model poisoning
- **Scaling / boosting:** multiply the malicious update by factor gamma so it dominates the average.
- **Sign flipping:** send `-delta` to reverse progress.
- **Gaussian noise updates:** random updates to destroy convergence.
- **Optimized attacks (Fang et al., local model poisoning; "A Little Is Enough"):** craft updates just inside the benign variance to bypass defenses; **min-max / min-sum** attacks.

### 3.3 Backdoor attacks
- Goal: model behaves normally on clean data but misclassifies inputs with a **trigger** into attacker's target class.
- Security example: a specific packet-size/flag pattern causes malicious traffic to be classified as benign.
- Techniques: model replacement (Bagdasaryan et al.), constrain-and-scale, distributed backdoor attack (DBA, split trigger across clients), edge-case backdoors (rare in-distribution inputs), neurotoxin / durable backdoors.
- Metrics: **Main task accuracy (MTA)** and **Attack Success Rate (ASR)**.

### 3.4 Sybil and collusion
- Attacker creates many fake clients. Defenses: FoolsGold, identity/attestation, rate limits, reputation.

### 3.5 Free-riding
- Client submits no useful update but benefits from the global model. Defenses: contribution evaluation (Shapley approximations), verification.

## 4. Privacy attacks in detail

### 4.1 Gradient / model inversion
- Given shared gradients, reconstruct training inputs: **Deep Leakage from Gradients (DLG, Zhu et al. 2019)**, iDLG, Inverting Gradients (Geiping et al.), GradInversion.
- Works best with small batches, few local steps, and unprotected gradients.
- A malicious server can improve leakage by sending **crafted models** ("trap weights", "model inconsistency").

### 4.2 Membership inference (MIA)
- Decide whether a specific record was in a client's training set (use loss/confidence, shadow models, update analysis).
- Security impact: reveal that a specific host or user appeared in logs.

### 4.3 Property inference
- Learn aggregate properties of other clients' data (e.g. "this org has lots of DNS tunneling traffic").

### 4.4 Label and attribute inference
- Infer labels from last-layer gradients (embedding-layer gradients also leak words in text models).

## 5. Defense overview (mapping)

| Threat | Defenses |
|---|---|
| Update leakage | Secure aggregation, DP, HE, MPC, TEEs, larger batches/local steps, compression |
| Poisoning | Robust aggregation, norm clipping, anomaly detection, FLTrust, client reputation, data validation |
| Backdoors | Clipping + noise, pruning, CRFL, FLAME, RLR (robust LR), neural cleanse adaptation |
| Sybil | FoolsGold, attestation, identity management |
| Malicious server | TEEs, verifiable aggregation, signatures, decentralized FL |
| Network attacks | TLS/mTLS, authentication, replay protection |
| DoS | Rate limiting, update size caps, deadlines |
| Evasion | Adversarial training (federated), input sanitization |

## 6. Differential privacy (DP)

**Definition:** A randomized mechanism M is (epsilon, delta)-DP if for neighbouring datasets D, D' and any output set S:
`P[M(D) in S] <= e^epsilon * P[M(D') in S] + delta`.
Smaller epsilon = stronger privacy.

### Levels in FL
| Level | What is protected | How |
|---|---|---|
| Record-level (local DP-SGD) | Individual training samples | Per-sample clipping + noise in local SGD (Opacus) |
| Client-level (central DP-FedAvg) | Entire client's contribution | Clip each client update to norm C, add Gaussian noise to the sum at server (McMahan et al. 2018) |
| Local DP | Even from the server | Noise added on client before upload (needs far more noise, worse utility) |
| Distributed DP | Server cannot see raw updates | Noise shares + SecAgg |

### DP-FedAvg recipe (client-level)
1. Each selected client computes `delta_k`.
2. Clip: `delta_k <- delta_k * min(1, C / ||delta_k||)`.
3. Server aggregates, adds noise: `N(0, sigma^2 C^2 I)`.
4. Track cumulative privacy with an **accountant** (moments accountant / RDP / PRV).

### Trade-offs
- Utility drops as epsilon shrinks; larger numbers of clients per round help (noise averages out).
- Typical reported epsilons range 1 to 10 in practice; justify your choice.
- DP also gives some resistance to backdoors and MIA (but reduces accuracy for rare classes, which matters for rare attacks!).

## 7. Secure aggregation (SecAgg)

Goal: server learns only the **sum** of client updates, not individual ones.

Intuition (pairwise masking, Bonawitz et al. 2017):
- Each pair of clients (i, j) agree on a shared random mask `s_ij` (via Diffie-Hellman key agreement).
- Client i uploads `x_i + sum_{j>i} PRG(s_ij) - sum_{j<i} PRG(s_ij)`.
- Masks cancel when all uploads are summed.
- Dropout handling uses secret sharing (Shamir) of seeds.

Properties and costs:
- Protects against honest-but-curious server; O(n^2) communication/computation (improved by SecAgg+, LightSecAgg).
- **Conflict:** SecAgg hides individual updates, so the server **cannot** run per-update robust aggregation or anomaly detection. This is a core research tension. Approaches: secure computation of robust statistics, zero-knowledge proofs on update norms, client-side checks, TEE-based aggregation, ELSA, RoFL, etc.

## 8. Homomorphic encryption (HE) and MPC

- **HE (Paillier, CKKS, BFV):** server aggregates ciphertexts without decrypting. Libraries: Microsoft SEAL, TenSEAL, OpenFHE, Pyfhel.
- Cost: large ciphertext expansion and compute overhead; best for small models or selective-layer encryption.
- **MPC / secret sharing:** split updates among several non-colluding servers. Libraries: CrypTen, MP-SPDZ, PySyft (some).
- **Trusted Execution Environments (TEEs):** Intel SGX/TDX, AMD SEV, ARM TrustZone, confidential VMs; aggregation inside an enclave with remote attestation. Beware side-channels.

## 9. Robust aggregation under attack: practical notes

- Defenses are evaluated against specific attacks; adaptive attackers (who know the defense) often break them. **Always test adaptive attacks** for credibility.
- Non-IID benign clients look like outliers, so robust rules can hurt benign accuracy or exclude minority-attack-type clients (a fairness concern in security: rare attack classes can be treated as "malicious").
- Combine defenses in layers: clipping + median/trimmed mean + DP noise + client reputation.

### Defense details
| Defense | Mechanism | Weakness |
|---|---|---|
| Norm clipping | Bound each update's L2 norm | Low-norm stealthy backdoors |
| Krum / Multi-Krum | Select closest updates | Fails under high non-IID, adaptive attack |
| Median / Trimmed mean | Coordinate-wise robustness | Variance high, slow with non-IID |
| FLTrust | Cosine similarity to server reference update, normalized | Needs clean root dataset |
| FoolsGold | Penalize similar updates | Needs update visibility; fails when attackers are diverse |
| FLAME | Clustering + clipping + noise | Complexity, hyperparameters |
| RLR | Flip learning rate for coordinates with sign disagreement | Hurts convergence |
| CRFL | Clipping + smoothing for certified robustness | Utility loss |

## 10. Defenses against privacy attacks: practical notes

- Increase local batch size and local epochs (more averaging = harder inversion).
- Add DP noise; compress/quantize; share only subsets of layers.
- Use SecAgg so individual gradients are unavailable.
- Evaluate with actual attacks (DLG, MIA) and report reconstruction metrics (PSNR/SSIM for images, token overlap for text, AUC for MIA).

## 11. Evasion attacks on the deployed global model (cyber-specific)

- Network IDS models are vulnerable to adversarial perturbations on features, but features must remain **valid and functional** (protocol constraints). Use problem-space attacks, not unconstrained L-inf perturbations.
- Defenses: federated adversarial training, input validation, ensemble/hybrid detection, feature robustness.

## 12. Verifiability, auditing, accountability

- Sign model updates and aggregated models; keep hash-chained logs (optionally on a blockchain).
- **Proof of correct aggregation / zero-knowledge proofs** for update validity (norm bounds).
- Contribution/credit auditing; model provenance and SBOM-style records for ML.

## 13. Federated unlearning and the right to be forgotten

- Remove a client or data influence from the global model: retraining from checkpoints, FedEraser, gradient ascent, influence-based methods.
- Relevant for GDPR compliance and removing detected-malicious clients.

## 14. Regulatory view

GDPR (data minimization, purpose limitation), HIPAA, India's DPDP Act, CCPA, EU AI Act. FL helps data minimization but does not alone guarantee compliance; DP and governance agreements are needed.

## 15. Research tension map (open problems you can target)

1. Robustness vs privacy (SecAgg hides updates needed by robust aggregation).
2. Robustness vs non-IID/fairness (benign outliers vs attackers).
3. Privacy vs utility on rare attack classes.
4. Defenses vs adaptive attackers.
5. Backdoors persisting after defense removal.
6. Practical overhead of HE/MPC for deep IDS models.
7. Attacks and defenses for federated LLMs.

## 16. Checklist before writing results

- [ ] Threat model stated precisely
- [ ] Attack strength varied (fraction of malicious clients, scale factors)
- [ ] Defense evaluated under IID and non-IID
- [ ] Adaptive attack tested
- [ ] Privacy budget reported with accountant details
- [ ] Overhead (time, communication) reported
