# 05. Federated Learning in Cybersecurity: Applications and Design Patterns

## 1. Why cybersecurity is a strong fit

- Logs/traffic contain PII, business secrets, and vulnerabilities. Sharing is restricted.
- Attack campaigns hit many targets; collective visibility improves detection of rare and new attacks.
- Defenders have asymmetric data: each sees only part of the threat landscape.
- Edge devices (IoT, routers, endpoints) generate data that is costly to centralize.

## 2. Application map

| Application | Data | Model | FL value |
|---|---|---|---|
| Network Intrusion Detection (NIDS) | Flow features (CIC-IDS, UNSW-NB15) | MLP, CNN, LSTM, AE | Cross-org attack coverage |
| IoT / IIoT anomaly detection | Device traffic (N-BaIoT, TON_IoT) | Autoencoder, GRU | Edge training, low bandwidth |
| Botnet / DDoS detection | Flow stats, packet sequences | CNN/LSTM | Collaborative early warning |
| Malware detection (Android, PE) | API calls, permissions, opcodes, byte images | CNN, GNN, MLP | Phones keep app data private |
| Phishing / spam / malicious URL | URLs, email text, HTML | Text models, transformers | Emails are private |
| Fraud / financial crime | Transactions | GBDT, NN, graph | Banks cannot share data |
| Host-based IDS / EDR | System calls, process trees | Sequence models | Endpoint telemetry privacy |
| Log anomaly detection | System/app logs | LSTM, transformer (DeepLog-style) | Cross-tenant logs |
| Threat intelligence sharing | IoCs, indicators | Federated analytics | Share patterns not logs |
| Insider threat / UEBA | User behaviour | Autoencoders | HR/privacy sensitivity |
| Vulnerability / code analysis | Source code | Code models, LLMs | Proprietary code stays local |
| Smart grid / ICS-SCADA security | Sensor streams | AE, LSTM | Critical infrastructure secrecy |
| Vehicular (CAN bus) intrusion | CAN frames | CNN/LSTM | Fleet learning |
| 5G/6G network security | Slice and core metrics | Various | Operator confidentiality |
| Biometric/authentication | Behavioural biometrics | Metric learning | Biometrics privacy |

## 3. Deep dive: Federated NIDS (recommended project)

### 3.1 Pipeline
1. **Collect:** each org has flows labeled benign/attack.
2. **Preprocess locally:** drop identifiers (IPs, timestamps/IDs as features cause leakage and overfitting), handle inf/NaN, scale (use global or federated statistics, not per-client min-max if possible), encode categoricals.
3. **Partition:** by organisation or by attack type (non-IID).
4. **Train locally:** MLP/1D-CNN/LSTM/autoencoder.
5. **Aggregate:** FedAvg/FedProx etc.
6. **Evaluate:** global test set not in any client; also per-client and unseen attack types.
7. **Deploy:** inference at network edge.

### 3.2 Federated feature normalization
- Compute global mean/variance via secure aggregation of sums and counts, or use fixed domain-based scaling, or normalization layers (careful with BatchNorm under non-IID; consider GroupNorm/LayerNorm).

### 3.3 Typical non-IID splits in NIDS
- **By attack type:** client 1 sees DoS + benign; client 2 sees PortScan + benign. Realistic: orgs face different attacks.
- **By label skew with Dirichlet(alpha):** alpha = 0.1 (extreme) to 100 (near IID).
- **By source/topology:** different subnets or testbeds.
- **Quantity skew:** vastly different sizes.

### 3.4 Pitfalls specific to IDS datasets
- Duplicates and leakage in CIC-IDS2017/2018 (known labeling errors; use corrected versions, e.g. Engelen et al. fixes).
- Temporal leakage: random splits inflate accuracy; use time-based splits where possible.
- Extreme class imbalance and "too-easy" features (e.g. Flow ID, Destination Port artifacts).
- Report F1/recall/FPR, not accuracy only.
- Test on **unseen attacks** to show generalization.

## 4. Deep dive: Federated IoT anomaly detection

- Each device or gateway trains an autoencoder on benign data only (N-BaIoT: Mirai/BASHLITE attacks).
- Aggregate weights; threshold from reconstruction-error statistics (e.g. mean + 3 std on validation benign data), possibly personalized per device.
- Challenges: constrained devices (use small models, quantization, TinyML), intermittent connectivity, device heterogeneity.
- Federated approaches known in literature: DIoT (Nguyen et al., federated self-learning anomaly detection), federated autoencoders on N-BaIoT.

## 5. Deep dive: Federated malware detection

- Android: features from permissions, API calls, intents (Drebin, AndroZoo, CIC-AndMal2017), or graphs (function call graphs + GNN).
- Windows PE: EMBER dataset features (static), BODMAS, SOREL-20M.
- Challenges: concept drift (malware evolves), adversarial malware (problem-space attacks), class imbalance.
- FL angle: phones/enterprises upload only updates; add DP to prevent leaking installed-app lists.

## 6. Deep dive: Federated phishing / spam

- Text-based classifiers, URL character CNNs, transformer fine-tuning with LoRA (reduces communication).
- Datasets: Enron/SpamAssassin, Nazario phishing corpus, PhishTank/OpenPhish URL feeds, UCI Phishing Websites.
- Concern: text models leak tokens through embedding gradients; use DP/SecAgg.

## 7. Federated threat intelligence sharing

- Instead of models, use **federated analytics**: heavy-hitter detection of malicious IPs/domains, joint frequency estimation with secure aggregation + DP.
- Alternative: federated learning of classifiers over IoC features, then share only the model.
- Related standards and platforms: STIX/TAXII, MISP; FL can complement these by enabling learning without publishing raw sightings.

## 8. Federated intrusion detection across SOCs (cross-silo scenario design)

- Parties: banks, ISPs, universities, hospitals.
- Governance: data-sharing agreement, audit, schema alignment, onboarding with attestation.
- Technical: mTLS, secure aggregation or TEE aggregator, contribution logging, drift monitoring, periodic re-training, model rollback.
- Evaluate business value: detection of attacks that an individual org never saw locally ("coverage gain").

## 9. FL with blockchain for security

- Use ledger to record model hashes, contributions, rewards; smart contracts for client admission and aggregator election.
- Cost: latency, throughput, storage; do not store full models on-chain (store hashes/IPFS pointers).
- Good for auditability research, but justify necessity.

## 10. Hybrid architectures

- **FL + split learning** for weak devices.
- **FL + TEEs** for server trust.
- **FL + knowledge distillation** for model heterogeneity.
- **FL + continual learning** for concept drift (new attacks): use replay, EWC, or periodic global refresh.
- **FL + XAI:** explain detections (SHAP/LIME locally; federated feature importance) for SOC analysts.
- **FL + LLMs:** federated fine-tuning (LoRA) for log analysis or threat report triage.

## 11. Concept drift and continual FL

- Attacks and benign traffic change over time. Strategies: sliding windows, drift detectors (ADWIN, DDM), periodic retraining, personalization layers, replay buffers of past attacks (stored locally).
- Evaluate with **temporal splits** across rounds (task-incremental scenarios).

## 12. Cyber-specific evaluation notes

- Detection metrics: Precision, Recall, F1 (macro and per-class), FPR (very important for SOC alert fatigue), ROC-AUC, PR-AUC, detection rate at fixed FPR (e.g. TPR@1%FPR).
- Latency and resource usage for inference at line rate.
- Robustness: adversarial and poisoning evaluation (file 04).
- Generalization: leave-one-attack-out, cross-dataset (train on CIC-IDS2017, test on UNSW-NB15 after feature mapping).

## 13. Project ideas matched to this file

1. Federated NIDS with attack-type non-IID split; compare FedAvg, FedProx, FedAdam.
2. Backdoor and label-flip attacks on federated NIDS plus robust aggregation comparison.
3. DP-FedAvg for IoT botnet detection: privacy-utility curves per attack class.
4. Federated autoencoder for zero-day detection, leave-one-attack-out.
5. Federated phishing URL classifier with membership-inference evaluation.
6. FL + secure aggregation + lightweight anomaly checks (norm proofs).
7. Personalized federated IDS for heterogeneous networks.
8. Federated malware detection with concept drift handling.
