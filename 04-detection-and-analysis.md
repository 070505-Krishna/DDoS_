# 04 — Detection & Traffic Analysis

Detection is the bridge between "an attack is happening" and "mitigation kicks in." This file covers what defenders actually look at — metrics, signatures, statistical/ML approaches, and a reference monitoring stack — so you can both recognize attacks in captured traffic and design detection logic for your own lab project.

## 4.1 What you're actually looking for: baseline vs. anomaly

Nearly all detection, at its core, comes down to **establishing a baseline of "normal" traffic and flagging statistically significant deviations from it**. This sounds simple but is genuinely hard in practice because:

- "Normal" traffic itself is bursty (sales events, viral content, breaking news) and can look a lot like an attack.
- Attackers increasingly try to **blend in** — e.g., Layer 7 floods using realistic-looking request patterns, browser-like headers, and even headless-browser automation to pass basic bot checks.
- Thresholds that are too sensitive cause false positives (blocking real users); thresholds too loose miss real attacks. Tuning this tradeoff is one of the most practically important (and most commonly graded) aspects of a detection-focused project.

## 4.2 Key metrics and signals, by attack category

### 4.2.1 Volumetric attack signals
- **Bits per second (bps) / bytes per second** against historical baseline, broken down by protocol.
- **Packets per second (PPS)** — important separately from bps, because small-packet floods can exhaust router/switch forwarding capacity (measured in PPS) long before bandwidth (measured in bps) is saturated.
- **Protocol distribution shift** — a sudden spike in UDP traffic on ports associated with amplification (53, 123, 1900, 11211, etc.) relative to baseline is a strong signal.
- **Source IP diversity** — a sudden, massive increase in *distinct* source IPs contacting the target is characteristic of both botnet floods and reflection attacks (since every reflector server is a distinct source IP from the victim's point of view).
- **Geographic/ASN distribution anomalies** — traffic suddenly arriving disproportionately from ASNs/countries with no normal relationship to the service (e.g., a regional e-commerce site for one country suddenly getting half its traffic from unrelated hosting-provider ASNs worldwide) — a signature of botnet traffic, since bots are wherever they were compromised, not wherever real customers are.

### 4.2.2 Protocol/state-exhaustion attack signals
- **SYN-to-ACK ratio** — in normal traffic this ratio should be close to 1:1 (every SYN is eventually followed by a completing ACK from a real handshake); a sustained skew toward many more SYNs than completed handshakes is the textbook SYN flood signature.
- **Half-open connection count** — direct measurement of the server/firewall's SYN-RECEIVED state table size over time.
- **Connection duration distribution** — Slowloris-style attacks produce a cluster of connections with unusually long duration and unusually low data-transfer rate (many bytes-per-second near zero, but connection open for minutes).
- **Concurrent connection count per source IP** — a small number of IPs holding a disproportionate number of total open connections is suspicious (though note: NAT'd corporate/campus networks can also look like this legitimately — a classic false-positive trap worth discussing in a project).

### 4.2.3 Application-layer attack signals
- **Requests per second (RPS) per endpoint**, not just site-wide — an attack concentrated on one expensive endpoint (e.g., `/search` or `/login`) might not move site-wide RPS much but can still take the backend down; endpoint-level granularity matters.
- **Request-to-response-time correlation** — if average response time climbs sharply while RPS climbs, that's consistent with a resource-exhaustion attack on expensive endpoints, vs. a simple volume flood against cheap/cached pages (which wouldn't necessarily slow down response time until much higher volume).
- **User-agent / header anomalies** — unusually uniform or missing User-Agent strings, missing standard browser headers (Accept-Language, Accept-Encoding), or headers that don't match claimed browser behavior — classic bot signatures, though increasingly defeated by more sophisticated headless-browser-based attack tooling.
- **Behavioral/navigation pattern anomalies** — real users browse with referrer chains, mouse movement (on JS-instrumented pages), and realistic page-to-page navigation timing; bots often hit single endpoints directly with no referrer, at inhumanly regular intervals, or with session/cookie handling that doesn't match a real browser.
- **TLS fingerprinting (JA3/JA3S)** — the specific set of cipher suites, extensions, and their order in a TLS ClientHello forms a fingerprint that can distinguish automated tooling (curl, Python requests, custom botnet clients) from real browsers, even when IP, User-Agent, and other headers are spoofed to look legitimate — an increasingly important advanced technique.

## 4.3 Detection approaches, from simple to advanced

### 4.3.1 Static thresholds
The simplest approach: "if requests-per-second from any single IP exceeds X, flag it." Easy to implement and explain, but trivially evaded by distributing the attack across enough source IPs, and prone to false positives from legitimate traffic bursts.

### 4.3.2 Statistical/behavioral baselining
Compares current traffic against a rolling historical baseline (e.g., "this hour's traffic vs. the average of the same hour over the past 4 weeks") and flags statistically significant deviations (e.g., using standard-deviation thresholds, or more advanced time-series methods like **CUSUM** (cumulative sum control chart) or exponentially weighted moving averages). This adapts to daily/weekly traffic cycles better than static thresholds.

### 4.3.3 Signature-based detection
Matches traffic against known attack patterns — e.g., a known botnet's characteristic packet structure, a specific amplification protocol's response signature, or a known attack tool's default parameters (many attack tools leave recognizable fingerprints, like particular TTL values, packet sizes, or TCP options, when run with default settings). Effective against known/recurring tools, blind to novel attacks.

### 4.3.4 Entropy-based detection
Measures the randomness (Shannon entropy) of fields like source IP, source port, or packet size across a traffic window. Legitimate traffic tends to have moderate, fairly stable entropy; certain attack patterns (e.g., highly randomized spoofed-source floods, or conversely highly *uniform* botnet traffic where every bot sends near-identical packets) show entropy spikes or drops that deviate sharply from baseline — a technique that can catch some novel attacks that evade simple threshold or signature matching.

### 4.3.5 Machine-learning-based detection (good advanced-level project territory)
Supervised approaches train classifiers (commonly: Random Forest, SVM, gradient-boosted trees, or neural networks) on labeled flow-level features (packet size distributions, inter-arrival times, protocol flags, flow duration, byte/packet counts) to classify traffic as benign vs. a specific attack category. Public datasets commonly used in academic DDoS-detection research include:

- **CICIDS2017 / CICDDoS2019** (Canadian Institute for Cybersecurity) — widely used, labeled flow datasets covering multiple DDoS attack types, good for a classroom ML project.
- **NSL-KDD** — an older, cleaned-up successor to the KDD Cup 1999 dataset, still used for baseline intrusion-detection ML work (broader than just DoS).
- **UNSW-NB15** — another widely cited modern labeled network intrusion dataset.

A genuinely strong project can combine `02-attack-types.md`'s taxonomy with one of these datasets: pick 2–3 attack categories, extract/engineer flow-level features, train and compare a couple of classifiers, and report precision/recall/F1 per attack class (not just overall accuracy — overall accuracy is a misleading metric on these datasets because benign traffic usually dominates the sample, so a model that just guesses "benign" every time scores deceptively well; this exact pitfall is worth naming explicitly in your write-up).

## 4.4 A reference monitoring/detection stack (useful for the lab project)

A realistic, buildable stack for a student lab environment:

| Layer | Example tools |
|---|---|
| Packet capture | `tcpdump`, Wireshark |
| Flow export/collection | `nfcapd`/`nfdump` (NetFlow), softflowd |
| Log aggregation | ELK stack (Elasticsearch, Logstash, Kibana) or Grafana + Loki |
| Metrics/time-series | Prometheus + Grafana dashboards |
| IDS/IPS signature engine | Suricata or Snort (both have built-in DoS-related rule sets) |
| Web server-level | nginx/Apache access logs + `fail2ban` for basic automated IP-based response |

In a lab project, a very achievable and genuinely instructive pipeline is: **generate traffic (mixed benign + attack) → capture with tcpdump → parse into flow records → compute the metrics in §4.2 → visualize the SYN/ACK ratio, RPS, or connection-duration distribution spike in Grafana or even just a matplotlib chart → show the moment detection logic would have fired.** This maps cleanly onto the lab plan in `06-lab-project-guide.md`.

## 4.5 Key challenges that make real-world detection hard (good discussion material for a project)

- **Encrypted traffic (TLS/QUIC)** — defenders increasingly can't inspect payload content, pushing detection toward metadata (timing, size, TLS fingerprinting) rather than content inspection.
- **Legitimate traffic spikes that mimic attacks** — flash crowds (a product launch, a viral news event) can look statistically identical to a volumetric attack; distinguishing "the Slashdot effect" from a botnet is a genuinely unsolved hard problem in some cases, and it's reasonable for a project to acknowledge this limitation rather than claim a perfect detector.
- **Low-and-slow attacks designed to stay under thresholds** — Slowloris-style and some application-layer attacks are explicitly engineered to avoid tripping volume-based alarms.
- **Evasion via botnet diversity** — attackers deliberately spreading load thin across huge numbers of bots specifically to keep any single source's contribution below per-IP thresholds.
- **False positive cost** — in a production environment, blocking real customers during a sales event is often considered *worse* than a brief service degradation, which shapes how aggressively real organizations tune detection — a nuance worth including if your project discusses real-world tradeoffs.

Continue to **`05-mitigation-and-defense.md`** for what happens once an attack is detected.
