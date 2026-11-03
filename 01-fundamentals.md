# 01. Fundamentals of Federated Learning

## 1. What is federated learning?

**Definition:** A distributed machine learning approach where multiple parties (clients) collaboratively train a model under the coordination of a server (or peer-to-peer), while the training data stays on each client's device or in its own data silo.

Core idea: **"Bring the code to the data, not the data to the code."**

Term origin: introduced by Google (McMahan et al., 2016/2017, "Communication-Efficient Learning of Deep Networks from Decentralized Data") for improving keyboard prediction (Gboard) without uploading what users type.

## 2. Why federated learning? (the motivation)

| Problem with centralized ML | How FL helps |
|---|---|
| Data privacy laws (GDPR, HIPAA, India's DPDP Act, CCPA) restrict moving data | Raw data never leaves the owner |
| Data is huge (video, logs) and expensive to upload | Only model updates move |
| Organisations will not share data (competition, trust) | They share learned knowledge, not records |
| Single central store is a breach target | No single honeypot of raw data |
| Data is naturally distributed (phones, sensors, branches) | Training occurs where data is generated |
| Latency / edge requirements | Models learn and run at the edge |

**Important caveat:** FL gives *data minimization*, not automatic privacy. Updates can leak information (see file 04).

## 3. What problem does it solve in cybersecurity?

- Security data (netflow, endpoint telemetry, email, auth logs) is sensitive and regulated.
- Attacks are often **shared across organisations**: the same malware, phishing kit or botnet hits many.
- A single organisation sees only a slice of attack types, so local models have blind spots (limited coverage of zero-days and rare classes).
- FL lets many organisations build a model that has "seen" all the attack types without exposing customer traffic.

## 4. When should you use FL?

Use FL when:
- Data cannot or should not be centralized (legal, contractual, competitive, bandwidth reasons).
- Many data holders have data from similar tasks but different distributions.
- Combined data would improve the model significantly over local-only.

Avoid FL when:
- Data can be legally and cheaply centralized (centralized training is simpler, stronger, and easier to debug).
- Each party has a tiny dataset and heterogeneity is extreme (may not beat local training).
- You need strong guarantees and cannot afford DP/SecAgg overhead.
- Clients are too few and unreliable.

**Decision checklist**
1. Is data movement restricted? -> FL candidate.
2. Is a pooled dataset much better than local? -> FL worthwhile.
3. Are updates alone acceptable to share? -> proceed, add privacy tech.

## 5. Where is FL used? (real-world domains)

- **Mobile/consumer:** next-word prediction, emoji suggestion, voice wake-word detection (Google Gboard, Apple on-device learning).
- **Healthcare:** hospitals training diagnostic models on imaging/EHR (e.g. brain tumor segmentation, COVID prediction).
- **Finance:** fraud detection and anti-money-laundering across banks.
- **IoT / smart cities / industry 4.0:** predictive maintenance, anomaly detection.
- **Automotive:** driving models from fleets.
- **Telecom:** network optimization, 5G/6G.
- **Cybersecurity:** intrusion detection, malware classification, phishing detection, DDoS detection, threat intelligence sharing (details in file 05).

## 6. How does FL work? (high-level loop)

```
Server initializes global model w0
for round t = 1..T:
    1. Server selects a subset of clients
    2. Server sends current global model w_t to them
    3. Each client trains on local data for E epochs -> w_k
    4. Each client sends update (w_k or w_k - w_t) to server
    5. Server aggregates updates -> w_{t+1}
Return final global model
```

Aggregation in plain FedAvg is a **weighted average** by local dataset size.

## 7. Comparison with other paradigms

| Aspect | Centralized | Distributed (data-center) | Federated | Split learning | Local only |
|---|---|---|---|---|---|
| Data location | One place | Sharded, but controlled | Stays at clients | Stays; model is split | Stays |
| Data distribution | Pooled | IID shards (you control) | Non-IID, unbalanced | Varies | Local |
| Clients | 1 | Reliable nodes | Unreliable, many | Few | 1 |
| Communication | Low | Very high-bandwidth | Expensive, limited | Per-batch activations | None |
| Privacy | Low | Low | Better (not perfect) | Better (not perfect) | Best |
| Accuracy | Upper bound usually | High | Close to centralized if tuned | Close | Often lowest |

Related ideas: **Edge AI**, **Transfer learning**, **Multi-party computation (MPC)**, **Homomorphic encryption (HE)**, **Differential privacy (DP)**, **Trusted execution environments (TEE)**, **Swarm learning** (blockchain-based decentralized FL).

## 8. Key challenges (the "four heterogeneities" + more)

1. **Statistical heterogeneity (non-IID data):** each client has a different label/feature distribution. Biggest accuracy killer.
2. **System heterogeneity:** different CPU, memory, network, battery. Stragglers and dropouts.
3. **Communication cost:** models have millions of parameters; many rounds. Compression and fewer rounds are needed.
4. **Privacy leakage:** gradients leak data (inversion, membership inference).
5. **Security:** poisoning, backdoors, Sybil attacks, malicious server.
6. **Fairness:** the model may favor clients with more data.
7. **Evaluation & debugging:** cannot inspect client data.
8. **Incentives and governance:** why would participants contribute honestly?

## 9. Core terminology cheat sheet

| Term | Meaning |
|---|---|
| Client / node / party | Data holder doing local training |
| Server / aggregator | Coordinates rounds, aggregates updates |
| Round | One cycle of broadcast -> local train -> aggregate |
| Local epoch (E) | Passes over local data per round |
| Participation fraction (C) | Fraction of clients sampled each round |
| Global model | Aggregated shared model |
| Local model | Client's current copy/personalized model |
| Update / delta | `w_local - w_global` (or gradients) |
| IID / non-IID | Same / different data distribution across clients |
| Straggler | Slow client delaying the round |
| Cross-device / cross-silo | Many small devices / few big organisations |
| Personalization | Adapting global model per client |
| Secure aggregation | Server sees only the sum of updates, not individuals |
| DP | Formal privacy guarantee via noise (epsilon, delta) |
| Byzantine client | Arbitrary/malicious behaviour |
| Poisoning | Corrupting training to degrade or implant behaviour |
| Backdoor | Targeted hidden behaviour triggered by a pattern |

## 10. Brief history

- 2015-2016: concept emerges at Google; "Federated Optimization".
- 2017: FedAvg paper (McMahan et al.).
- 2017: Secure aggregation protocol (Bonawitz et al.).
- 2018: FedProx, LEAF benchmark, TensorFlow Federated.
- 2019: "Advances and Open Problems in FL" survey; Flower, FATE, PySyft ecosystem grows; large-scale systems paper (Bonawitz).
- 2019-2021: attacks (DLG, backdoors "How to backdoor FL"), robust aggregation (Krum, Median, Trimmed mean).
- 2021 onwards: FL for LLMs / foundation models, federated analytics, FL with TEEs, split + federated hybrids, personalization.

## 11. Quick self-check questions

1. Why is FL not automatically private?
2. What is the difference between cross-silo and cross-device FL?
3. Why does non-IID data hurt FedAvg?
4. Name three cybersecurity tasks suited to FL and why.
