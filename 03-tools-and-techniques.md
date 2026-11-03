# 03 — Tools, Botnets & Techniques: How Attacks Are Actually Built

This file covers how real-world DoS/DDoS attacks are assembled and launched — botnet architecture, the amplification protocol landscape, the "DDoS-as-a-service" economy, and a table of notable historical attacks you can cite in a project. It is written for **understanding and defensive research**, not as an operational how-to — no step-by-step attack instructions against real infrastructure are included anywhere in this guide.

## 3.1 Botnet architecture

A botnet is the "distributed" engine behind most large DDoS attacks. Understanding its architecture matters because **detection and takedown strategies target different layers of this architecture**.

```
         Attacker
             │
             ▼
      ┌─────────────┐
      │  C2 Server   │  ← Command & Control infrastructure
      │ (or P2P mesh)│     issues attack commands
      └──────┬──────┘
             │
   ┌─────────┼─────────┬─────────────┐
   ▼         ▼         ▼             ▼
 Bot 1     Bot 2     Bot 3   ...   Bot N      ← compromised devices
   │         │         │             │          (routers, cameras, PCs,
   │         │         │             │           cloud instances)
   └─────────┴─────────┴─────────────┘
             │  (attack traffic, often spoofed)
             ▼
          TARGET
```

### 3.1.1 How devices become bots
- **Credential attacks against default/weak passwords** — especially IoT devices shipped with default admin credentials never changed by the owner (Mirai's primary method: a hardcoded dictionary of ~60 common default username/password pairs tried over Telnet).
- **Unpatched vulnerabilities** — remote code execution bugs in routers, cameras, NAS devices, and even cloud workloads.
- **Malware delivered via phishing, drive-by downloads, or malicious attachments** — classic route for compromising traditional PCs.
- **Compromised cloud/hosting accounts** — stolen credentials or leaked API keys used to spin up attack infrastructure on legitimate cloud providers, which has the side benefit (for the attacker) of high-bandwidth, high-reputation source IPs.

### 3.1.2 C2 models
- **Centralized C2**: all bots check in with one (or a small cluster of) command servers. Easier to build, but a single point of failure — taking down the C2 server disables the whole botnet. Classic IRC-based botnets (like early Trinoo/TFN variants) used this model.
- **Peer-to-peer (P2P) botnets**: bots communicate with each other, no single C2 to take down. Harder to build and harder to kill (no single point of failure), used by more sophisticated malware families.
- **Domain Generation Algorithms (DGA)**: bots algorithmically generate a large list of possible C2 domain names daily; the attacker only needs to register one of them. Makes blocklisting C2 domains far harder since defenders can't predict which domain will be "live" on a given day without reverse-engineering the algorithm.

### 3.1.3 Notable botnet families (useful as case studies)

| Botnet | Era | Notable trait |
|---|---|---|
| Trinoo / TFN (Tribe Flood Network) | 1999–2000 | Among the first widely-documented DDoS botnets; used in the MafiaBoy attacks. |
| Mirai | 2016–present (many variants) | Targets IoT (cameras, routers) via default-credential Telnet scanning; source code leaked publicly, spawning dozens of derivative botnets (Satori, Okiru, Masuta, etc.); responsible for the Dyn/2016 attack. |
| Mēris | 2021 | Exploited MikroTik router vulnerabilities; used for some of the highest request-per-second attacks recorded at the time. |
| Emotet-derived / general malware botnets | Ongoing | Often multi-purpose (spam, banking fraud, DDoS-for-hire capability bundled in). |

## 3.2 The amplification protocol landscape

This table is one of the most useful quick-reference artifacts you can include in a project — it summarizes the UDP-based protocols historically (and sometimes still) abused for amplification, with approximate amplification factors reported by researchers (these vary by source/study and server configuration, so cite a primary source like a CERT advisory or academic paper if you use exact figures in a report).

| Protocol | Port | Approx. amplification factor | Root cause | Status (as commonly discussed) |
|---|---|---|---|---|
| DNS | 53 | ~28–54x (higher with large record responses like ANY queries) | Small query, large zone/record response | Still abused; mitigated by response-rate limiting (RRL) on modern resolvers |
| NTP (`monlist`) | 123 | Up to ~550x | `monlist` command returns long list of recent peers | Largely mitigated — `monlist` disabled/removed in modern `ntpd` |
| memcached | 11211 | Up to 10,000–50,000x in worst documented cases | UDP interface meant only for trusted internal networks, exposed publicly | Largely mitigated — vendors now disable UDP by default |
| SSDP (UPnP) | 1900 | ~30x | Home routers/IoT devices respond to discovery queries | Still a factor due to huge installed base of consumer IoT |
| CLDAP | 389 | ~56–70x | Connectionless LDAP used in some enterprise/AD environments | Still observed in the wild |
| Chargen (legacy) | 19 | Very high, protocol-dependent | Ancient "character generator" debug protocol | Mostly extinct but occasionally still found exposed |
| RIPv1 | 520 | ~130x | Legacy routing protocol | Rare today, mostly historical interest |

**The common thread across every row**: a **UDP-based service, reachable from the public internet, that was never meant to be internet-facing or that returns far more data than it receives**, combined with the ability to spoof the request's source IP. This is why "don't expose UDP services you don't need to the open internet" and "implement BCP 38 egress filtering" are two of the single highest-leverage defensive recommendations in the entire DDoS field — they attack the *root cause* shared by this entire table, rather than any one protocol.

## 3.3 The DDoS-as-a-Service ("booter"/"stresser") economy

A large share of real-world, low-to-mid-tier DDoS activity today doesn't come from sophisticated actors building custom botnets — it comes from **commercial platforms**, often thinly disguised as "stress testing" services, that let anyone pay a small fee (sometimes just a few dollars) to launch an attack against a target IP with no technical skill required.

- These platforms typically operate their own botnets (often built on compromised IoT devices or cloud/VPS abuse) or lease capacity from others.
- Law enforcement operations (e.g., the FBI/Europol-coordinated **"Operation PowerOFF"**, and earlier **"Operation Power Off"** actions against platforms like **vDOS** and others) have repeatedly seized booter domains and arrested operators — this is a good, citable example of the legal consequences side of a project, and reinforces why this guide does not provide operational attack instructions.
- Legitimate load-testing/stress-testing tools (used lawfully, against infrastructure you own, with authorization) exist and serve a real engineering purpose — the line between "stress testing tool" and "booter service" is almost entirely about **authorization and intent**, not the underlying technology, which is an important nuance for a project's ethics/legal section (see `06-lab-project-guide.md`).

## 3.4 Tool categories worth knowing about (for recognition, not operational use)

It is reasonable for a project to *name and describe* categories of tooling researchers and red teams commonly reference in the literature — understanding what a SOC analyst might see referenced in a threat report is part of being conversant in this field.

- **LOIC (Low Orbit Ion Cannon)** — an open-source network stress testing tool that became infamous when repurposed by "Anonymous" for voluntary, crowd-sourced application-layer/volumetric floods during Operation Payback (2010) and other hacktivist campaigns; notable because it does **not** hide the sender's IP, leading to many participants being identified and prosecuted — a useful cautionary case study.
- **HOIC (High Orbit Ion Cannon)** — a successor to LOIC with scriptable "booster" files to target multiple sites simultaneously.
- **hping / nping** — legitimate, widely-used packet-crafting and network testing utilities (part of Nmap's suite) capable of generating custom TCP/UDP/ICMP packets; used in authorized penetration testing and also, historically, in SYN flood style attacks — another case where the tool itself is dual-use and legality hinges on authorization.
- **Commercial/legitimate load-testing tools** — Apache JMeter, Locust, k6, Gatling. These are **not attack tools** — they're built for legitimate performance/load testing against infrastructure you own or are authorized to test — but they're worth knowing because they're exactly what you'd use for the *authorized, defensive* lab project described in `06-lab-project-guide.md`.

This guide intentionally does not provide usage instructions, attack scripts, or configuration details for offensive booter/stresser tooling. If your coursework requires live attack traffic generation, do it only against infrastructure you own, fully isolated from the internet, using the legitimate load-testing tools above — covered in detail in `06-lab-project-guide.md`.

## 3.5 Notable attacks timeline (citable case studies for a project)

| Year | Target | Approx. scale | Vector(s) | Why it's notable |
|---|---|---|---|---|
| 2000 | Yahoo!, eBay, Amazon, CNN | N/A (early era) | Early botnet floods | One of the first DDoS attacks in mainstream media; perpetrator ("MafiaBoy") was a minor |
| 2007 | Estonian government/banks/media | Sustained weeks-long campaign | Mixed | Early example of DDoS as geopolitical tool |
| 2013 | Spamhaus | ~300 Gbps | DNS amplification | One of the largest attacks recorded at the time |
| 2016 | Dyn (DNS provider) | ~1.2 Tbps (estimated) | Mirai IoT botnet, multi-vector | Took down Twitter, Netflix, GitHub, Reddit, etc. via a single DNS provider; demonstrated the systemic risk of DNS as a single point of failure |
| 2018 | GitHub | 1.35 Tbps | Memcached amplification | Largest attack recorded at the time; notable for requiring no botnet at all |
| 2020 | AWS (unnamed customer) | 2.3 Tbps | CLDAP reflection | Reported by AWS Shield as the largest attack they'd mitigated at the time |
| 2022–2023 | Various (Google, Cloudflare, AWS customers) | 100M+ req/sec | HTTP/2 Rapid Reset (CVE-2023-44487) | Largest *request-rate* attacks recorded; exploited a protocol-level flaw, not a botnet size record |

Continue to **`04-detection-and-analysis.md`** to see how defenders actually spot these patterns in traffic.
