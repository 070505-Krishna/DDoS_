# Denial of Service (DoS) Attacks — Complete Project Reference

A multi-file reference built for someone doing a **project** on Denial of Service / Distributed Denial of Service attacks — covering theory, taxonomy, tooling, detection, mitigation, and a safe lab/project guide. Difficulty ranges from beginner concepts to advanced, protocol-level detail.

## How this guide is organized

| File | Covers | Level |
|---|---|---|
| `01-fundamentals.md` | What DoS/DDoS is, history, motives, actors, core networking concepts you need | Beginner → Intermediate |
| `02-attack-types.md` | Full taxonomy: volumetric, protocol/state-exhaustion, application-layer, with packet-level mechanics | Intermediate → Advanced |
| `03-tools-and-techniques.md` | Botnets, amplification/reflection, stressers/booters, real attack tooling, notable attacks | Intermediate → Advanced |
| `04-detection-and-analysis.md` | Traffic analysis, signatures, metrics, detection algorithms, monitoring stack | Intermediate → Advanced |
| `05-mitigation-and-defense.md` | Defense-in-depth architecture, rate limiting, scrubbing, Anycast, cloud WAF/CDN, case studies | Intermediate → Advanced |
| `06-lab-project-guide.md` | Legal/ethical boundaries, safe home-lab setup, project ideas with scope, write-up structure, grading-friendly deliverables | All levels |
| `07-federated-learning-detection.md` | Federated learning for collaborative, privacy-preserving DDoS detection — FedAvg mechanics, non-IID challenges, published approaches (FLAD, FedDB, Anomaly-Flow), and a scoped project using Flower/TFF on CICDDoS2019/CICIDS2017 | Advanced |

## Suggested reading order

1. Read `01` and `02` fully first — you need the taxonomy before tools/defense make sense.
2. Skim `03` for context on how attacks are actually launched in the wild.
3. Read `04` and `05` together — detection and mitigation are two halves of the same problem.
4. Use `06` to turn all of the above into an actual project with a lab, a deliverable, and a report structure.
5. If your project direction is a machine-learning/federated-learning detection system specifically, read `07` after `04` and `06` — it builds directly on the detection metrics in `04` and the dataset-based project track in `06`.

## A note on scope and safety

Everything here is written for **defensive understanding and authorized lab work only**. No file in this set gives you a ready-to-fire attack script against a real target, live botnet infrastructure, or working exploit code for a current unpatched vulnerability. `06-lab-project-guide.md` explains exactly how to build a **fully isolated, self-contained lab** (your own VMs attacking your own VMs, with no internet exposure) so you can observe attack traffic and build defenses hands-on without touching anyone else's systems or breaking the law. Read that file's legal section before you run anything.
