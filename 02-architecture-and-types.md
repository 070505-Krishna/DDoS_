# 02. Architecture, Types, and System Design

## 1. Topologies

### 1.1 Centralized (client-server) FL
- One server orchestrates; clients talk only to the server.
- Pros: simple, well-supported by frameworks, easy aggregation/monitoring.
- Cons: server is a single point of failure and a trust bottleneck.

```
   Client1   Client2   Client3
      \         |         /
       \        |        /
          [ Aggregator ]
```

### 1.2 Decentralized / peer-to-peer FL
- No central server; clients exchange updates with neighbors (gossip, ring, graph).
- Pros: no single point of failure, trust distributed.
- Cons: convergence analysis harder, more communication complexity.
- Related: blockchain-based FL / swarm learning (ledger records updates, smart contracts incentivize).

### 1.3 Hierarchical FL
- Clients -> edge aggregators -> cloud aggregator.
- Good for telecom, smart cities, multi-branch enterprises; reduces long-haul traffic.

```
Clients -> Edge Server A \
Clients -> Edge Server B  -> Cloud Server
Clients -> Edge Server C /
```

## 2. Settings: cross-device vs cross-silo

| Property | Cross-device | Cross-silo |
|---|---|---|
| Clients | Millions of phones/IoT | 2-100 organisations |
| Client reliability | Low (drop out often) | High (servers) |
| Data per client | Small | Large |
| Client state | Stateless typically | Stateful |
| Participation per round | Tiny fraction | Often all |
| Main bottleneck | Communication, availability | Trust, legal, heterogeneity |
| Security example | Mobile antivirus, IoT botnet detection | Banks, SOCs, hospitals, ISPs |

Most **cybersecurity research** uses cross-silo (several organisations) or IoT cross-device simulation.

## 3. Types by data partitioning

### 3.1 Horizontal FL (HFL)
Same feature space, different samples. Example: many companies each with netflow logs of the same schema.

### 3.2 Vertical FL (VFL)
Same samples (entities), different features. Example: a bank and a telecom both hold data on the same customers (bank: transactions, telecom: call patterns). Requires private entity alignment (PSI, private set intersection) and split-model training.

### 3.3 Federated Transfer Learning (FTL)
Different samples and different features with small overlap; use transfer learning to bridge.

| Type | Samples | Features | Typical use |
|---|---|---|---|
| Horizontal | different | same | IDS across organisations |
| Vertical | same | different | Fraud detection across institutions |
| Transfer | different | different | Cross-domain, small overlap |

## 4. Types by learning paradigm

- **Supervised FL** (classification of attacks) - most common.
- **Unsupervised / self-supervised FL** (autoencoder anomaly detection) - useful because attack labels are scarce.
- **Semi-supervised FL** - few labeled clients, many unlabeled.
- **Federated reinforcement learning** - adaptive defense policies.
- **Federated analytics** - statistics (counts, histograms) without models.
- **Federated unlearning** - removing a client's influence.
- **Federated LLM fine-tuning** - LoRA/adapters (security chatbots, log analysis).

## 5. System components

| Component | Responsibility |
|---|---|
| Orchestrator / coordinator | Schedules rounds, tracks state |
| Client selector | Chooses participants (random, resource-aware, loss-based) |
| Aggregator | Combines updates (FedAvg, robust, secure) |
| Model store / registry | Versioned global models |
| Communication layer | gRPC / HTTPS / MQTT, TLS or mTLS |
| Client runtime | Training loop, data loader, local privacy (clip/noise) |
| Privacy module | DP accountant, secure aggregation |
| Security module | Authentication, attestation, anomaly/poison detection |
| Monitoring | Metrics, drift detection, logs |
| Incentive/reputation | Contribution scoring (optional) |

## 6. Lifecycle of one round (detailed)

1. **Selection:** server samples `m = max(C*K, 1)` clients (or by eligibility).
2. **Broadcast:** server sends `w_t` (possibly compressed/quantized).
3. **Local training:** client runs `E` epochs of SGD with batch size `B`, learning rate `eta`.
4. **Local protection (optional):** clip update norm, add noise, mask for secure aggregation.
5. **Upload:** send update plus metadata (sample count).
6. **Aggregation:** server computes `w_{t+1}` (weighted average or robust variant).
7. **Evaluation:** on a held-out server set and/or client-side validation.
8. **Checkpoint & log**, repeat.

Sync vs async:
- **Synchronous:** wait for all selected clients (or a deadline). Simple, stable.
- **Asynchronous:** apply updates as they arrive (staleness weighting). Faster for stragglers, harder convergence.

## 7. Client selection strategies

- Uniform random (default).
- Resource-aware (battery, network, idle, charging).
- Loss-based / importance sampling (Power-of-Choice).
- Clustered / stratified (cover diverse data).
- Reputation-based (exclude suspected malicious).

## 8. Communication efficiency techniques

| Technique | Idea |
|---|---|
| More local computation | Larger E, fewer rounds (FedAvg itself) |
| Quantization | 32-bit -> 8/4/1 bit (signSGD, QSGD) |
| Sparsification | Send top-k entries of the update |
| Sketching | Count-sketch compressed gradients |
| Low-rank / LoRA | Send low-rank factors only |
| Knowledge distillation (FedDF, FedMD) | Exchange predictions/logits instead of weights |
| Partial participation | Sample fewer clients |
| Hierarchical aggregation | Aggregate at edge first |

## 9. Heterogeneity-handling designs

- **Data heterogeneity:** FedProx, SCAFFOLD, FedNova, FedBN, clustered FL, personalization, data sharing of small public sets.
- **Model heterogeneity:** FedMD/FedDF (distillation), HeteroFL (sub-networks), split learning.
- **Device heterogeneity:** adaptive local epochs, deadline-based rounds, asynchronous FL.

## 10. Trust models (who do you trust?)

| Model | Server | Clients | Typical protections |
|---|---|---|---|
| Honest-but-curious server | Follows protocol, tries to learn | Honest | Secure aggregation, DP, HE |
| Malicious server | May tamper, send bad models | Honest | Verifiable aggregation, TEEs, signatures |
| Malicious clients | Honest | Some Byzantine | Robust aggregation, anomaly detection |
| Colluding parties | Server + some clients | | Threshold crypto, DP |

State your trust model explicitly in any security paper.

## 11. Reference architecture for a cybersecurity FL system

```
[Org A: SIEM/IDS logs] -> local preprocess -> local trainer --\
[Org B: firewall flows] -> local preprocess -> local trainer ---> (mTLS) -> Secure Aggregator
[Org C: IoT gateway]    -> local preprocess -> local trainer --/            |
                                                                           v
                                                       Global model registry -> back to orgs
                                                       Monitoring / poison detection / audit
```

Design notes:
- Standardize the **feature schema** across orgs (e.g. CICFlowMeter features) or use encoders.
- Use mTLS and signed model artifacts; log all rounds for audit.
- Deploy the final model locally as IDS inference engine; consider drift monitoring.
