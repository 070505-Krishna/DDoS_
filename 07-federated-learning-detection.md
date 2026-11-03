# 07 — Federated Learning for DDoS Detection

This file extends the detection discussion in `04-detection-and-analysis.md` §4.3.5 into a specific, active research direction: using **Federated Learning (FL)** to train DDoS detection models collaboratively across multiple organizations/clients without centralizing raw traffic data. This is a legitimate, well-published defensive research area — every technique below trains a model to *recognize* attack traffic, not to generate or coordinate one.

## 7.1 Why federated learning fits this problem

Centralized ML-based DDoS detection (as described in `04-detection-and-analysis.md`) assumes you can pool labeled traffic data from many sources into one place to train a strong model. In practice this assumption usually breaks down:

- **Privacy/regulatory barriers** — raw traffic logs can contain sensitive information (user IPs, access patterns, sometimes payload fragments); many organizations are legally or contractually barred from sharing this externally.
- **Competitive sensitivity** — ISPs, CDNs, and enterprises often don't want to reveal traffic patterns, customer behavior, or security posture to competitors or even to a shared third-party platform.
- **No single vantage point sees the whole attack** — a distributed attack, by definition, spreads load across many networks; each individual organization only observes its own slice, which may look unremarkable in isolation even when the aggregate pattern across organizations is clearly an attack.
- **Data volume and sovereignty** — raw packet/flow data at ISP scale is enormous and often can't legally cross jurisdictional borders.

**Federated Learning's core idea**: instead of moving data to a central model, you move the **model to the data**. Each participating client (an ISP, a data center, an edge node, an enterprise network) trains a local copy of a shared model on its own local traffic data. Only the **model updates** (weights or gradients) — not the raw data — are sent to a central aggregating server, which combines them into an improved global model and redistributes it. This directly addresses the privacy and data-sovereignty barriers above while still letting every participant benefit from patterns learned across the whole federation.

## 7.2 Baseline mechanics: FedAvg

The foundational FL algorithm, **Federated Averaging (FedAvg)**, works roughly like this:

```
 Central Server                     Clients (1..K)
 ───────────────                    ───────────────
 1. Initialize global model W₀
 2. For each round t = 1..T:
      │
      ├─► Send current global model Wₜ to a
      │   selected subset of clients
      │
      │                              3. Each selected client trains
      │                                 locally on its own data for
      │                                 a few epochs, producing an
      │                                 updated local model Wₜ₊₁ᵏ
      │
      │   ◄───────────────────────── 4. Client sends only the updated
      │        (weights/gradients,       weights (or gradient deltas)
      │         not raw data)             back to the server
      │
 5. Server aggregates all received
    local updates — typically a
    weighted average by each
    client's local dataset size —
    to form the new global model Wₜ₊₁
      │
 6. Repeat until convergence
```

For DDoS detection specifically, each client's local dataset is its own labeled (or semi-labeled) traffic flows — benign traffic plus whatever attack types that client has observed historically. The resulting global model, after enough rounds, has effectively learned from the attack profiles of *every* client in the federation, without any client ever seeing another's raw traffic.

## 7.3 Why plain FedAvg struggles specifically for DDoS detection

Several published works (see §7.5) identify the same recurring problems when applying vanilla FedAvg to this domain, which is exactly why DDoS-specific FL variants exist:

- **Non-IID data across clients** — this is the single biggest issue in the literature. Client A (say, a gaming platform) might see mostly SYN floods and application-layer floods; Client B (a DNS provider) might see mostly amplification/reflection traffic. Each client's local data distribution is wildly different from the global distribution, which standard FedAvg assumes away — in practice this causes slow convergence and models that perform unevenly across attack types.
- **Unbalanced client dataset sizes** — some clients (large ISPs) may have far more traffic/labeled examples than others (small enterprises), which skews a naive size-weighted average toward whichever client has the most data rather than whichever client has the most *useful* signal for a given attack type.
- **Some attack types are "harder to learn" than others** — rare or subtle attack profiles (e.g., low-rate application-layer attacks vs. obvious volumetric floods) can get drowned out during aggregation if the FL process doesn't allocate extra training attention to clients whose local data represents those harder cases.
- **Requiring shared test/validation data breaks the privacy premise** — the original FedAvg formulation assumes the central server has some test data to monitor global model performance and decide convergence; in a security context this is often unrealistic, since recent attack traffic is exactly the sensitive data clients don't want to share, even for validation purposes.

## 7.4 Notable published approaches (useful as related work / comparison baselines)

These are real, citable papers — useful both to read directly and to frame your own project's contribution ("I implement X, and compare it against standard FedAvg as a baseline, similar to how [paper] did").

| Approach | Core idea | Dataset(s) used | Citation |
|---|---|---|---|
| **FLAD** (Adaptive FL for DDoS Detection) | Server monitors global model accuracy using **clients' own local validation sets** (no shared test data ever leaves a client) and dynamically assigns **more training rounds/computation to clients whose attack profiles are harder to learn**, directly addressing the non-IID problem above | A recent heterogeneous DDoS dataset (feed-forward neural network) | Doriguzzi-Corin & Siracusa, *Computers & Security* 137 (2024); arXiv:2205.06661. Public prototype implementation available on GitHub (search "FLAD DDoS federated learning GitHub"). |
| **FedDB** | Combines **personalized FL** with **DBSCAN clustering** specifically to handle non-IID client data, rather than forcing one global model to fit every client equally well | — | Lee, Chien & Chang, *Applied Sciences* 14(22), 2024 |
| **FL-DAD** | CNN-based FL detection aimed at identifying attacks "at the source" (i.e., close to where traffic originates, useful for IoT/edge deployment) | CICIDS2017 | Alhasawi & Alghamdi, *IEEE Access* 12, 2024 |
| **Anomaly-Flow** | Combines FL with **Generative Adversarial Networks (GANs)** to generate synthetic flow data for multi-domain detection, aimed at organizations with very different traffic profiles collaborating without sharing raw flows at all — even synthetically | Multi-domain simulated setup | arXiv:2503.14618, March 2025 |
| **Edge-FL IDS for heterogeneous IoT** | 1D-CNN deployed via FL across edge nodes specifically for resource-constrained IoT environments (relevant given IoT's role as the dominant botnet-building substrate — see `03-tools-and-techniques.md` §3.1.1) | CICDDoS2019 | Mahadik et al., cited in Chellaperumal & Muthukumaran, *Economic Computation and Economic Cybernetics Studies* 59(3), 2025 |
| **LSTM Autoencoder FL for 5G/CPS** | Time-series anomaly detection (LSTM Autoencoder) over FL for 5G-enabled cyber-physical systems, rather than flow-based classification | 5G core network multivariate time series | Munaweera et al., ACM, 2025 |

A strong related-work section doesn't just list these — it groups them by **what problem they're each solving** (non-IID handling, privacy of validation data, edge/IoT deployment constraints, synthetic data generation) and positions your own project relative to that grouping.

### 7.4.1 Direct links to read these papers

| Paper | Link |
|---|---|
| FLAD (Doriguzzi-Corin & Siracusa, 2022/2024) | [arxiv.org/abs/2205.06661](https://arxiv.org/abs/2205.06661) (free preprint, all versions); published version: [doi.org/10.1016/j.cose.2023.103597](https://doi.org/10.1016/j.cose.2023.103597) (*Computers & Security*, paywalled) |
| FLAD's public code implementation | Linked from the paper's project page: [rising.fbk.eu/tools/FLAD](https://rising.fbk.eu/tools/FLAD) — has a GitHub link and step-by-step install/usage guide |
| Resource-Efficient FL for NIDS (follow-up FLAD resource-usage paper, Doriguzzi-Corin, Cretti & Siracusa, NetSoft 2024) | Open-access author's manuscript via [iris.unitn.it/handle/11572/446351](https://iris.unitn.it/handle/11572/446351) |
| FedDB (Lee, Chien & Chang, *Applied Sciences* 2024) | Free, fully open access: [mdpi.com/2076-3417/14/22/10236](https://www.mdpi.com/2076-3417/14/22/10236) |
| FL-DAD (Alhasawi & Alghamdi, *IEEE Access* 2024) | Free, fully open access (IEEE Access is OA): [doi.org/10.1109/ACCESS.2024.3378727](https://doi.org/10.1109/ACCESS.2024.3378727) |
| Anomaly-Flow (de Melo, Bertoli, Nogueira, dos Santos & Pereira Jr., arXiv 2025) | [arxiv.org/abs/2503.14618](https://arxiv.org/abs/2503.14618) (free preprint; HTML version at [arxiv.org/html/2503.14618v1](https://arxiv.org/html/2503.14618v1)) |
| IoT Networks with a Federated Learning Model (Chellaperumal & Saravanakumar, *Economic Computation and Economic Cybernetics Studies and Research* 2025 — the paper citing the Edge-FL IoT work in the table above) | Free, fully open access PDF: [ecocyb.ase.ro/nr2025_3/11_ChellaperumalMuthukumaran.pdf](https://ecocyb.ase.ro/nr2025_3/11_ChellaperumalMuthukumaran.pdf) |
| Deep Learning + FL on CICDDoS2019 (Alsarhan, *Engineering, Technology & Applied Science Research* 2026) | Free, fully open access: [etasr.com/index.php/ETASR/article/view/14119](https://www.etasr.com/index.php/ETASR/article/view/14119) |
| Deep learning techniques for DDoS detection (survey covering FL integration, Fu et al., Elsevier 2025) | Publisher page (may be paywalled depending on institutional access): [pure.bit.edu.cn/en/publications/deep-learning-techniques-for-ddos-attack-detection-concepts-analy](https://pure.bit.edu.cn/en/publications/deep-learning-techniques-for-ddos-attack-detection-concepts-analy/) |
| Trusted Multi-Domain DDoS Detection Based on Federated Learning (Yin, Li & Bi, *Sensors* 2022 — an earlier multi-domain FL approach using reputation/blockchain to screen malicious participants, good additional related work) | Free, fully open access: [pmc.ncbi.nlm.nih.gov/articles/pmid/36298104](https://pmc.ncbi.nlm.nih.gov/articles/pmid/36298104) |

A few notes on using these:
- Most of the journals above (*Applied Sciences*/MDPI, *IEEE Access*, *Sensors*, ETASR, the Romanian economics journal) are fully **open access** — the links go straight to the complete paper, free, no institutional login needed.
- FLAD's own paper is a free arXiv preprint regardless of the journal version's paywall status — always check arXiv first for a given paper before assuming you need institutional access.
- If a link above ever breaks or redirects, search the paper's exact title plus "arxiv" or "doi" — all of these are indexed and easy to re-find.

## 7.5 A concrete, scoped project built on this

This builds directly on the dataset track already described in `06-lab-project-guide.md` §6.3 — you don't need live traffic or internet-facing infrastructure for any of this.

### 7.5.1 Tooling
- **Flower** (`flwr`) — a popular, well-documented, framework-agnostic FL library (works with PyTorch, TensorFlow, scikit-learn); designed specifically for exactly this kind of academic FL simulation and has extensive tutorials for "simulate N clients on one machine" setups, which is all you need for a project (no real distributed infrastructure required).
- **TensorFlow Federated (TFF)** — Google's FL research library; slightly steeper learning curve than Flower but has strong built-in support for FedAvg variants and simulation.
- Either is a reasonable choice; Flower is generally considered more approachable for a first FL project.

### 7.5.2 Step-by-step scope

1. **Data partitioning (this is where most of the interesting research decisions live)**: take CICDDoS2019 or CICIDS2017 (per `04-detection-and-analysis.md` §4.3.5) and split it into N simulated clients. Don't just split randomly — that would be IID and wouldn't demonstrate anything interesting. Instead, simulate realistic heterogeneity deliberately, e.g.:
   - Partition by attack type, so Client 1 sees mostly SYN floods + benign, Client 2 sees mostly UDP floods + benign, Client 3 sees mostly HTTP floods + benign, etc. — this directly simulates the "different organizations see different attack profiles" scenario motivating §7.1.
   - Vary client dataset sizes unevenly (some clients get 10x more samples than others) to simulate real-world organizational size differences.
2. **Baseline model**: pick something simple first — logistic regression or a small feed-forward neural network (matching FLAD's approach) — before attempting a CNN or LSTM; a simple model trains fast in simulation and makes debugging the FL mechanics themselves much easier.
3. **Implement standard FedAvg first** as your baseline using Flower's built-in strategy.
4. **Implement one improvement** addressing a specific weakness from §7.3 — reasonable choices for a course project, roughly in increasing order of difficulty:
   - *Weighted aggregation by validation performance* rather than just dataset size (a simplified version of FLAD's core idea) — reward clients whose updates actually improve global validation accuracy.
   - *Client selection strategies* — each round, select which clients participate based on how "hard" their local attack profile has been to learn so far, rather than selecting randomly or using all clients every round.
   - *Personalized/clustered FL* (a simplified FedDB-style approach) — instead of one global model, maintain a few model variants and cluster clients toward whichever variant fits their local distribution best.
5. **Evaluation — compare three things, not just one**:
   - **Centralized baseline** — same model architecture trained on all the data pooled together (the "upper bound" you're comparing against; FL should come close to this without ever centralizing data).
   - **Standard FedAvg** — your first implementation.
   - **Your improvement** — show it closes some of the gap between FedAvg and the centralized baseline, specifically for the attack types or client splits where FedAvg struggled most.
6. **Report, per §6.3's point on metrics**: per-attack-class precision/recall/F1 for all three approaches, plus FL-specific metrics — **number of communication rounds to converge**, **total communication overhead** (bytes transferred client↔server, a genuinely important real-world cost metric for FL that's easy to compute and often overlooked), and **performance variance across clients** (does the global model work equally well for every client, or does it favor whichever client had the most/easiest data? — this variance is itself a legitimate research finding worth reporting even if your "improvement" doesn't fully solve it).

### 7.5.3 What a strong discussion section engages with
- The **privacy–utility tradeoff** directly: FL avoids sharing raw data, but gradient/weight updates can still leak information in principle (a well-known open research area called *gradient inversion* / *model inversion* attacks against FL — worth a paragraph acknowledging even if out of scope to implement).
- **Communication cost is a real constraint**, not just an accuracy number — in a real ISP-scale deployment, sending full model weights every round across many clients is expensive; this is part of why FLAD specifically optimizes for fewer required rounds, not just final accuracy.
- **Honesty about simulation vs. reality** — a simulated non-IID split on one machine is a reasonable proxy, but real deployments would face clients with wildly different hardware (full ISPs vs. resource-constrained IoT edge devices, per the Edge-FL row in §7.4), unreliable connectivity, and clients that drop out mid-round — acknowledging these as limitations, rather than ignoring them, reads as more rigorous.

## 7.6 How this connects back to the rest of the guide

- The attack taxonomy in `02-attack-types.md` is exactly what determines your client data partitioning strategy in §7.5.2 — you need to understand *why* a SYN flood's flow features differ from an HTTP flood's to partition and label data meaningfully.
- The detection metrics and pitfalls in `04-detection-and-analysis.md` (especially the imbalanced-accuracy trap in §4.3.5) apply identically here — FL doesn't change the need for per-class precision/recall.
- The mitigation architecture in `05-mitigation-and-defense.md` is where a successful FL-based detector would ultimately plug in — e.g., feeding a positive detection into a scrubbing-center trigger or an automated rate-limiting rule — worth one paragraph in your introduction to motivate *why* better detection matters, even if deploying the full mitigation pipeline is out of scope for your project.
- The legal/ethical framing in `06-lab-project-guide.md` §6.1 still applies in full: this entire project track uses public, pre-labeled datasets and local simulation — no live traffic, no real multi-organization data sharing, no infrastructure beyond your own machine.

---

Suggested addition to `00-INDEX.md`'s table if you're assembling the final set:

| File | Covers | Level |
|---|---|---|
| `07-federated-learning-detection.md` | Federated learning for collaborative, privacy-preserving DDoS detection — FedAvg mechanics, non-IID challenges, published approaches (FLAD, FedDB, Anomaly-Flow), and a scoped project using Flower/TFF on CICDDoS2019/CICIDS2017 | Advanced
