# 09. Research Roadmap, Project Ideas, Reading List, Glossary, and Q&A

## 1. 8-week project plan (adjustable)

| Week | Goal | Deliverable |
|---|---|---|
| 1 | Learn basics (files 01-03); set up environment (06) | Environment works; FedAvg on MNIST |
| 2 | Choose task + dataset; preprocess; centralized + local baselines | Clean data pipeline, baseline numbers |
| 3 | Implement FedAvg on security data with IID and non-IID splits | Convergence curves, per-class results |
| 4 | Add one improvement (FedProx/FedAdam/personalization) | Ablation on alpha, E, K |
| 5 | Implement attacks (label flip, backdoor, scaling) | ASR and MTA results without defense |
| 6 | Implement defenses (clip, median, Krum, DP, ...) | Defense vs attack matrix |
| 7 | Novel contribution + adaptive attacks + overhead | Final experiments, multiple seeds |
| 8 | Write paper/report, polish code, plots | Report, repo, slides |

## 2. Choosing a research contribution

Pick one of these contribution types (a project does not need all):

1. **Application study:** apply FL to a new security task or dataset and benchmark carefully.
2. **Attack:** a new poisoning/backdoor/inference attack for security-domain features (respecting feature validity constraints).
3. **Defense:** new robust aggregation or detection of malicious clients under non-IID.
4. **Privacy-utility:** DP/SecAgg tuned for rare attack classes.
5. **Systems:** efficient/communication-light FL for IoT/edge IDS; real testbed.
6. **Personalization:** per-network adapted detectors.
7. **Evaluation/benchmark:** systematic comparison fixing flaws in prior evaluations (often valuable, e.g. showing leakage in a dataset).
8. **Survey/SoK:** systematization of knowledge of FL security in a sub-domain.

## 3. Concrete project ideas with expected outcomes

| # | Idea | Core experiments | Difficulty |
|---|---|---|---|
| 1 | Federated NIDS benchmark on CIC-IDS2017 / UNSW-NB15 with attack-type non-IID | FedAvg vs FedProx vs FedAdam vs local vs centralized | Easy |
| 2 | Backdoor attacks on federated NIDS + defenses | Trigger via feature pattern; ASR vs defense matrix | Medium |
| 3 | DP for rare attack classes | Epsilon vs per-class recall; class-aware clipping/noise | Medium |
| 4 | Federated autoencoder zero-day detection (leave-one-attack-out) | AUC for unseen attacks; personalized thresholds | Medium |
| 5 | Poison-resilient FL with reputation using update similarity + client history | Compare with Krum, FLTrust, FoolsGold; adaptive attacker | Hard |
| 6 | FL on Raspberry Pi testbed for IoT botnet detection | Real latency/energy, quantized model | Medium |
| 7 | Federated phishing URL detector with MIA evaluation | AUC of membership attack with/without DP | Medium |
| 8 | Federated threat intel via DP histograms | Heavy-hitter recovery accuracy vs epsilon | Medium |
| 9 | SecAgg-compatible robust aggregation (norm proofs/bounds) | Overhead and detection under SecAgg | Hard |
| 10 | Federated continual learning for concept drift in malware | Temporal splits across rounds; forgetting metrics | Hard |
| 11 | Federated LLM fine-tuning (LoRA) for log anomaly triage | Communication vs quality; leakage tests | Hard |
| 12 | Vertical FL fraud detection between two parties (PSI + split model) | AUC vs local; privacy analysis | Hard |
| 13 | Explainable federated IDS (federated SHAP summary) | Consistency of explanations across clients | Medium |
| 14 | Blockchain-audited FL for SOC consortium | Audit overhead; tamper detection | Medium |

## 4. Paper/report structure

1. **Abstract** (problem, method, key numbers)
2. **Introduction** (motivation, gap, contributions list, threat model summary)
3. **Background** (FL, relevant security task)
4. **Related work** (FL-IDS, FL attacks/defenses, DP/SecAgg; compare to your work in a table)
5. **Threat model and problem statement**
6. **Method / system design** (diagram, algorithm box)
7. **Experimental setup** (datasets, preprocessing, partitions, baselines, metrics, hyperparameters, hardware)
8. **Results** (main table, ablations, attack/defense, overhead)
9. **Discussion** (limitations, adaptive attackers, ethics, deployment)
10. **Conclusion and future work**
11. **Appendix** (extra plots, hyperparameters, proofs)

Venues: IEEE TIFS, IEEE TDSC, IEEE IoT Journal, Computers & Security, ACM CCS / USENIX Security / NDSS / IEEE S&P (top-tier security), NeurIPS/ICML/ICLR workshops on FL, MLSys, AsiaCCS, ACSAC, RAID, ESORICS, IEEE Access (broad). Check each venue's current policies.

## 5. Reading list (foundational and security)

### Foundations
1. McMahan et al. "Communication-Efficient Learning of Deep Networks from Decentralized Data" (FedAvg).
2. Kairouz et al. "Advances and Open Problems in Federated Learning."
3. Li et al. "Federated Optimization in Heterogeneous Networks" (FedProx).
4. Karimireddy et al. "SCAFFOLD."
5. Reddi et al. "Adaptive Federated Optimization" (FedOpt).
6. Li et al. "Federated Learning: Challenges, Methods, and Future Directions."
7. Hsieh et al. "The Non-IID Data Quagmire of Decentralized ML."
8. Bonawitz et al. "Towards Federated Learning at Scale: System Design."
9. Yang et al. "Federated Machine Learning: Concept and Applications" (HFL/VFL/FTL).

### Privacy
10. Bonawitz et al. "Practical Secure Aggregation for Privacy-Preserving ML."
11. McMahan et al. "Learning Differentially Private Recurrent Language Models."
12. Abadi et al. "Deep Learning with Differential Privacy" (DP-SGD).
13. Zhu et al. "Deep Leakage from Gradients."
14. Geiping et al. "Inverting Gradients - How Easy Is It to Break Privacy in FL?"
15. Nasr et al. "Comprehensive Privacy Analysis of Deep Learning" (white-box MIA).
16. Melis et al. "Exploiting Unintended Feature Leakage in Collaborative Learning."

### Attacks and defenses
17. Bagdasaryan et al. "How To Backdoor Federated Learning."
18. Bhagoji et al. "Analyzing Federated Learning through an Adversarial Lens."
19. Fang et al. "Local Model Poisoning Attacks to Byzantine-Robust FL."
20. Baruch et al. "A Little Is Enough."
21. Wang et al. "Attack of the Tails: Yes, You Really Can Backdoor FL."
22. Xie et al. "DBA: Distributed Backdoor Attacks."
23. Blanchard et al. "Machine Learning with Adversaries: Byzantine Tolerant Gradient Descent" (Krum).
24. Yin et al. "Byzantine-Robust Distributed Learning" (median, trimmed mean).
25. Cao et al. "FLTrust."
26. Fung et al. "Mitigating Sybils in FL Poisoning" (FoolsGold).
27. Nguyen et al. "FLAME: Taming Backdoors in FL."
28. Shejwalkar et al. "Back to the Drawing Board: A Critical Evaluation of Poisoning Attacks on FL" (realistic threat models).
29. Lyu et al. "Threats to Federated Learning: A Survey."

### Cybersecurity applications
30. Nguyen et al. "DIoT: A Federated Self-learning Anomaly Detection System for IoT."
31. Surveys on federated learning for intrusion detection and IoT security (search recent surveys in IEEE Communications Surveys & Tutorials, ACM CSUR).
32. Rahman et al. and others on federated IDS using CIC-IDS/UNSW/ToN-IoT; Mothukuri et al. "Federated-Learning-Based Anomaly Detection for IoT Security Attacks."
33. Sarhan et al. on standardized NetFlow-based datasets (NF-UNSW-NB15, etc.) and federated evaluation.
34. Engelen et al. "Troubleshooting an Intrusion Detection Dataset: CICIDS2017 and its Improved Version."

(Verify titles and authors with a search; check for newer works in your sub-topic from 2024-2026.)

## 6. Where to learn more

- Flower docs and tutorials (flower.ai), TFF tutorials, OpenMined courses, Google "Federated Learning" comic and blog.
- Courses: Stanford/Berkeley/CMU talks on FL and privacy; "Privacy in ML" lectures; Coursera/Udacity "Secure and Private AI."
- Communities: Flower Discord, OpenMined Slack, r/MachineLearning, arXiv-sanity, Papers with Code (FL tasks).
- Awesome lists on GitHub: "Awesome-Federated-Learning", "Awesome-FL-Security".

## 7. Glossary (A-Z)

| Term | Definition |
|---|---|
| Aggregation | Combining client updates into a global model |
| ASR | Attack Success Rate of a backdoor/targeted attack |
| Backdoor | Hidden behaviour triggered by a specific input pattern |
| Byzantine | Arbitrary/malicious behaviour of a participant |
| Client drift | Local models diverging from global optimum due to non-IID local training |
| Clipping | Bounding the norm of updates/gradients |
| Cross-device / cross-silo | Many unreliable devices / few reliable organisations |
| Dirichlet partition | Method to simulate label-skewed clients |
| DP | Differential Privacy |
| epsilon, delta | DP parameters (privacy loss bound, failure probability) |
| FedAvg | Federated Averaging algorithm |
| FedProx | FedAvg with proximal regularization |
| HE | Homomorphic Encryption |
| HFL / VFL | Horizontal / Vertical FL |
| IID | Independent and identically distributed data |
| Krum | Byzantine-robust selection rule |
| MIA | Membership Inference Attack |
| MPC | Multi-Party Computation |
| MTA | Main Task Accuracy |
| Non-IID | Data distributions differ among clients |
| PSI | Private Set Intersection (entity alignment in VFL) |
| Personalization | Tailoring models to each client |
| RDP | Renyi Differential Privacy (accounting) |
| Round | One communication cycle |
| SecAgg | Secure Aggregation |
| Sybil | One adversary posing as many clients |
| TEE | Trusted Execution Environment |
| Straggler | Slow client delaying a round |
| Zero-day | Previously unseen attack |

## 8. Exam-style Q&A (revision)

**Q1. Define federated learning. How does it differ from distributed training in a datacenter?**
FL trains a shared model across decentralized, private data held by clients, exchanging only updates. Datacenter distributed training controls data placement (IID shards, fast network, reliable nodes); FL faces non-IID data, unreliable clients, limited bandwidth and privacy constraints.

**Q2. Explain the FedAvg algorithm.**
Server broadcasts global weights; sampled clients train E epochs locally with SGD; server takes the average of returned weights weighted by sample counts `n_k/n`; repeat for T rounds.

**Q3. Why does non-IID data degrade FedAvg and what are remedies?**
Local objectives differ so local updates drift toward client-specific optima; their average may be a poor global direction. Remedies: FedProx (proximal term), SCAFFOLD (control variates), server momentum/Adam, personalization, fewer local epochs, data sharing/augmentation.

**Q4. List three privacy attacks on FL and a defense for each.**
Gradient inversion (DP, SecAgg, larger batches); membership inference (DP, regularization); property inference (DP, limiting update access, secure aggregation).

**Q5. What is a backdoor attack? Give a cybersecurity example and defenses.**
Targeted poisoning causing a trigger-conditioned misclassification while preserving normal accuracy; e.g. flows with a specific flag/size pattern classified as benign. Defenses: norm clipping + noise, robust aggregation (FLAME, median), RLR, anomaly detection, pruning.

**Q6. Explain secure aggregation briefly.**
Clients add pairwise random masks that cancel when summed; the server learns only the aggregate; secret sharing handles dropouts.

**Q7. What is DP in FL and its trade-off?**
Clip updates and add calibrated noise to bound the influence of any single client/record, giving (epsilon, delta)-DP. Smaller epsilon = better privacy but lower accuracy, especially on rare classes.

**Q8. Compare Krum, median, and trimmed mean.**
Krum selects the single update closest to its neighbours; median takes the per-coordinate median; trimmed mean drops extremes per coordinate then averages. All tolerate a minority of Byzantine clients but struggle with high non-IID and adaptive attackers.

**Q9. Why use federated learning for intrusion detection?**
Organisations cannot share raw traffic, yet attacks are shared across them; FL pools knowledge to improve coverage, including rare and zero-day attacks, while keeping logs local.

**Q10. Horizontal vs vertical FL?**
Horizontal: same features, different samples (multiple orgs, same log schema). Vertical: same samples, different features (bank + telecom customers), requiring PSI and split models.

**Q11. Which metrics matter for a federated IDS?**
Macro-F1, per-class recall, FPR, PR-AUC, TPR@low FPR, plus FL metrics (rounds, communication), ASR under attack, and epsilon if DP.

**Q12. What is a Sybil attack and how can it be mitigated?**
An adversary registers many fake clients to gain influence. Mitigations: authentication/attestation, rate limits, FoolsGold-like similarity penalties, reputation.

## 9. Advanced topics for future exploration

- Certified robustness in FL (randomized smoothing, CRFL).
- Federated unlearning for malicious-client removal.
- FL with foundation models/LLMs: parameter-efficient tuning, prompt leakage, memorization.
- Fairness in FL (q-FFL, AFL) and incentive mechanisms (Shapley value, auctions, contract theory).
- Asynchronous and semi-asynchronous FL; staleness-aware aggregation.
- Federated reinforcement learning for adaptive defense/honeypots.
- Federated graph learning (network graphs, provenance graphs with GNNs).
- Federated learning over wireless/6G (over-the-air computation).
- Verifiable FL (zero-knowledge proofs of training/aggregation).
- Quantum-safe cryptography for FL.
- Regulatory compliance-by-design (DPDP, GDPR, EU AI Act).

## 10. Final practical advice

1. Get a correct FedAvg baseline running before anything fancy.
2. Pick a precise, defensible threat model; most weak papers fail here.
3. Always compare against centralized and local-only.
4. Use realistic non-IID data and report heterogeneity.
5. Evaluate with security-relevant metrics (per-class recall, FPR).
6. Test adaptive attackers and report overhead.
7. Keep experiments reproducible and code clean from day one.
