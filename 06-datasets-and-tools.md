# 06. Datasets, Frameworks, and Environment Setup

## 1. Frameworks and libraries

| Framework | Maintainer | Backend | Best for | Notes |
|---|---|---|---|---|
| **Flower (flwr)** | Flower Labs | PyTorch, TF, JAX, sklearn, any | Research and production, simulation + real deployment | Framework-agnostic, strategy API, popular for security research |
| **TensorFlow Federated (TFF)** | Google | TensorFlow | Algorithm research, simulation | Steeper learning curve |
| **PySyft** | OpenMined | PyTorch | Privacy-preserving data science, remote data | Focus on data governance and remote execution |
| **FedML** | FedML | PyTorch | Cross-device/silo, MLOps platform | Includes benchmarks |
| **NVIDIA FLARE** | NVIDIA | PyTorch, TF | Cross-silo, healthcare | Production-oriented |
| **FATE** | WeBank | Own | Industrial, vertical FL, HE | Strong enterprise features |
| **OpenFL** | Intel | PyTorch, TF | Cross-silo, TEEs (SGX) | Healthcare focus |
| **FLUTE** | Microsoft | PyTorch | Large-scale simulation | Research |
| **FedScale** | Univ. Michigan | PyTorch | Realistic scale and systems benchmark | Device traces |
| **LEAF** | CMU | TF | Benchmark datasets (FEMNIST, Shakespeare) | Older |
| **FedLab / FL-bench / PFLlib** | Community | PyTorch | Many algorithms implemented | Good for baselines |
| **Opacus** | Meta | PyTorch | DP-SGD | Pair with FL loop |
| **TensorFlow Privacy** | Google | TF | DP | |
| **TenSEAL / SEAL / OpenFHE** | Various | - | Homomorphic encryption | |
| **CrypTen** | Meta | PyTorch | MPC | |

Recommended starter stack for a cybersecurity research project:
- Python 3.10+, PyTorch, scikit-learn, pandas, numpy
- Flower for orchestration/simulation
- Opacus for DP
- Optional: `ray` (Flower simulation backend), `wandb` or `tensorboard`, `shap`

Always verify installation commands and API versions in the official documentation; Flower's API changed significantly between versions.

## 2. Environment setup

```bash
python -m venv fl-env
source fl-env/bin/activate        # Windows: fl-env\Scripts\activate
pip install torch scikit-learn pandas numpy matplotlib
pip install "flwr[simulation]"     # Flower with simulation engine
pip install opacus                 # differential privacy (optional)
pip install tensorboard shap       # logging, explainability (optional)
```

Hardware notes:
- Small tabular IDS models (MLP) run on CPU. Simulation of 10-50 clients is fine on a laptop.
- CNN/LSTM on large flows or images benefits from a GPU (Colab, Kaggle, university cluster).
- For real multi-machine deployments use Docker containers or Raspberry Pi/Jetson as clients.

## 3. Datasets for cybersecurity FL

### 3.1 Network intrusion datasets

| Dataset | Content | Size | Notes |
|---|---|---|---|
| **CIC-IDS2017** | Benign + DoS, DDoS, PortScan, Brute force, Web attacks, Botnet, Infiltration | ~2.8M flows | Widely used; fix known label/feature issues |
| **CSE-CIC-IDS2018** | Similar, larger, cloud testbed | ~16M | Large; good for client partitions |
| **UNSW-NB15** | 9 attack families | ~2.5M | Different features; cross-dataset tests |
| **NSL-KDD** | Classic KDD cleaned | ~150K | Outdated but good for teaching/exams |
| **KDD Cup 99** | Original | 5M | Very outdated; avoid for serious work |
| **TON_IoT** | IoT/IIoT telemetry, network, OS logs | Large | Multi-source |
| **Bot-IoT** | IoT botnet, DDoS, recon | ~73M records | Big, imbalanced |
| **N-BaIoT** | Mirai, BASHLITE on 9 IoT devices | ~7M | Device-level natural federation |
| **CIC-IoT2023 / CIC-IoMT2024** | IoT attack traces | Large | Newer |
| **Edge-IIoTset** | IIoT testbed | Large | Edge focus, FL friendly |
| **X-IIoTID** | Industrial IoT | Medium | |
| **ISCX-VPN/NonVPN, Tor** | Encrypted traffic classification | Medium | |
| **CTU-13** | Botnet captures | 13 scenarios | Pcap-based; natural scenario split |
| **CIC-DDoS2019** | DDoS variants | Large | |
| **LANL cyber dataset** | Auth/network logs, enterprise | Huge | Log/graph anomaly detection |

### 3.2 Malware and phishing

| Dataset | Domain |
|---|---|
| EMBER (v1/v2/2018) | Windows PE static features |
| SOREL-20M | Windows PE, large-scale |
| BODMAS | PE with timestamps (drift) |
| Drebin, AndroZoo, CIC-AndMal2017, CICMalDroid2020, AndroCT | Android |
| Malimg | Malware as images |
| Microsoft Malware Classification (BIG 2015) | Kaggle, bytes + asm |
| PhishTank, OpenPhish, UCI Phishing Websites, PhiUSIIL | Phishing URLs/websites |
| Enron, SpamAssassin, TREC | Spam/email |

### 3.3 Logs and others
| Dataset | Domain |
|---|---|
| HDFS, BGL, Thunderbird (Loghub) | System log anomaly detection |
| CERT Insider Threat | Insider threat |
| CAN-intrusion datasets (Car-Hacking, ROAD) | Vehicular |
| SWaT, WADI, HAI | ICS/SCADA anomaly detection |
| Credit Card Fraud (Kaggle, ULB), IEEE-CIS Fraud | Fraud detection |

### 3.4 General FL benchmarks
MNIST, FEMNIST (LEAF), CIFAR-10/100, Shakespeare, Sent140, StackOverflow (TFF), FedScale datasets. Use these first to validate that your FL code works before moving to security data.

## 4. Data preprocessing checklist (security data)

1. Remove identifiers (IP addresses, flow IDs, timestamps) from features unless justified.
2. Drop duplicates; handle NaN/inf; remove constant columns.
3. Consistent label mapping across clients (e.g. "DoS Hulk" -> "DoS").
4. Scale: StandardScaler with global statistics (or log-transform + clipping).
5. Encode categoricals consistently (shared vocabularies).
6. Handle imbalance (class weights, focal loss, resampling **only on training data**).
7. Split train/val/test **before** augmentation; prefer temporal splits.
8. Save the processed shards per client to disk (`client_0.csv` ... or `.npz`).

## 5. Partitioning data across clients

| Strategy | Code idea | Models |
|---|---|---|
| IID | Random shuffle, split equally | Baseline |
| Label skew (Dirichlet) | For each class, sample proportions ~ Dir(alpha) over clients | Typical non-IID |
| Shard-based (pathological) | Sort by label, give each client s shards | Extreme non-IID |
| Quantity skew | Dirichlet on sizes | Unbalanced clients |
| Feature skew | Partition by feature/source (subnet, device) | Domain shift |
| Natural | One client per device/org/scenario (N-BaIoT devices, CTU-13 scenarios) | Most realistic |

## 6. Experiment management

- Fix random seeds (numpy, torch, python, DataLoader workers); report mean +/- std over 3-5 seeds.
- Config files (YAML/JSON, Hydra) for hyperparameters.
- Log per-round metrics: global accuracy/F1, loss, per-client metrics, ASR (if attack), epsilon (if DP), bytes transferred, round time.
- Version data and code (git, DVC).
- Save plots: convergence curves, confusion matrices, per-class recall.

## 7. Simulation vs real deployment

| Aspect | Simulation (single machine) | Real deployment |
|---|---|---|
| Speed | Fast | Slow, network dependent |
| Realism | Models only | Real latency, failures, security |
| Use | Algorithm and attack research | Systems papers, demos, testbeds |
| Tools | Flower simulation (Ray), FedML simulator | Flower gRPC clients on Raspberry Pi/Docker, FATE, NVIDIA FLARE |

A good research project often does large-scale simulation plus a small real testbed (3-5 containers or devices) to measure real communication and latency.

## 8. Useful extra libraries

- **Adversarial/security ML:** ART (Adversarial Robustness Toolbox), CleverHans, Foolbox, SecML, `torchattacks`.
- **Explainability:** SHAP, Captum, LIME.
- **Imbalance:** imbalanced-learn.
- **Networking features:** CICFlowMeter, Zeek, nfstream, Scapy (pcap -> features).
- **Metrics:** scikit-learn (`classification_report`, `roc_auc_score`, `precision_recall_curve`).
- **DP accounting:** `opacus.accountants`, `dp-accounting` (Google), `autodp`.
