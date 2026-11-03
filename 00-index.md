# Federated Learning for Cybersecurity Research: Complete Guide

A learning path from zero to a working research project.

## How to use this guide

| # | File | What you get | Level |
|---|------|--------------|-------|
| 01 | `01-fundamentals.md` | What, why, when, where, history, core terms | Beginner |
| 02 | `02-architecture-and-types.md` | Topologies, FL types, system components, lifecycle | Beginner-Intermediate |
| 03 | `03-algorithms.md` | FedSGD, FedAvg, FedProx, SCAFFOLD, personalization, math | Intermediate |
| 04 | `04-privacy-and-security.md` | Threat model, attacks, defenses (DP, SecAgg, robust aggregation, HE) | Intermediate-Advanced |
| 05 | `05-cybersecurity-applications.md` | IDS, malware, phishing, IoT, fraud, threat intel, design patterns | Intermediate |
| 06 | `06-datasets-and-tools.md` | Datasets, frameworks, hardware, environment setup | Practical |
| 07 | `07-implementation-guide.md` | Runnable code: FedAvg, Flower, DP, attacks, defenses | Practical-Advanced |
| 08 | `08-evaluation-and-experiments.md` | Metrics, non-IID simulation, baselines, ablations, reproducibility | Advanced |
| 09 | `09-research-roadmap.md` | Project ideas, timeline, paper structure, reading list, glossary, exam Q&A | All |

## Suggested reading order

1. Fast track (1 day): 01 -> 02 -> 03 (FedAvg only) -> 07 (FedAvg code).
2. Security-research track (1 week): all files in order, then pick a project in 09.
3. Exam revision: 01, 03, 04 summaries and the Q&A in 09.

## One-paragraph summary

Federated learning (FL) trains a shared model across many clients (phones, hospitals, firewalls, SOC sensors) **without moving raw data** to a central server. Clients train locally and send only model updates; a server aggregates them into a better global model. In cybersecurity, FL is attractive because security logs and traffic are sensitive and legally restricted, yet attackers share techniques across organisations, so defenders benefit from collaborating. FL is also itself an attack surface: poisoning, backdoors, inference and gradient-leakage attacks. A strong research project usually combines **FL + a security task + an attack or defense evaluation**.

## Project outline you can follow

1. Pick a task (e.g. network intrusion detection).
2. Pick a dataset and split it non-IID across clients.
3. Train centralized and local-only baselines.
4. Implement FedAvg and compare.
5. Add an attack (label flipping, backdoor).
6. Add a defense (median, Krum, clipping, DP).
7. Report accuracy, F1, attack success rate, communication cost, privacy budget.

## Conventions

- Notation: `K` clients, `C` fraction selected per round, `E` local epochs, `B` batch size, `w` model weights, `n_k` samples at client k, `T` rounds.
- Code uses short, simple variable names and PyTorch.
- Library APIs change; always check the version in your environment against the official docs.
