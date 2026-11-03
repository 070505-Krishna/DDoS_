# 01 — Fundamentals of Denial of Service Attacks

## 1.1 What is a Denial of Service attack?

A **Denial of Service (DoS)** attack is any deliberate action that makes a computer system, network, service, or resource unavailable to its intended, legitimate users. The goal is almost never to steal data — it's to **deny availability**, one of the three pillars of information security alongside Confidentiality and Integrity (the "CIA triad"). A successful DoS attack doesn't need to breach a single password or exploit a single buffer overflow to "win" — it just needs to exhaust something the service depends on: bandwidth, CPU, memory, connection slots, disk I/O, or application logic capacity.

A **Distributed Denial of Service (DDoS)** attack is the same goal achieved using **many source machines at once**, usually spread across many networks and geographies. Distribution matters for two reasons:

- **Scale**: a single attacking machine's bandwidth and processing power is tiny compared to most modern infrastructure; thousands of machines acting together can generate traffic volumes a single host never could.
- **Resilience to blocking**: if traffic comes from one IP, the defender just blocks that IP. If it comes from 100,000 IPs across 190 countries, IP-based blocking stops working, and the defender has to fall back to pattern-based or behavioral mitigation.

## 1.2 Why "denial of service" instead of "theft of service"

It helps to contrast DoS with the other major attack families early, because this distinction shapes everything else in this guide:

| Attack goal | Example | What's compromised |
|---|---|---|
| Confidentiality | Data breach, eavesdropping | Secrecy of data |
| Integrity | SQL injection to alter records, MITM tampering | Correctness/trustworthiness of data |
| **Availability** | **DoS/DDoS** | **Access to a working system** |

DoS attacks are often the "blunt instrument" of cyberattacks — they don't require finding a subtle logic flaw (though some do); many just require **more** of something than the target can handle. This is also why DoS is historically one of the *easiest* attack categories to execute and one of the *hardest* to fully prevent — you cannot wall off a public-facing service from receiving traffic without also blocking the legitimate users you're trying to serve.

## 1.3 A brief history

| Period | Milestone |
|---|---|
| 1974 | One of the first documented DoS-style disruptions: a 13-year-old student crashed 31 PLATO terminals at the University of Illinois using an "external" command. |
| 1996 | Early SYN flood attacks against ISPs (e.g., Panix) demonstrate TCP handshake abuse at scale. |
| 1999–2000 | **MafiaBoy** (a 15-year-old) takes down Yahoo!, Amazon, CNN, eBay, and Dell using early DDoS tools (Tribe Flood Network, Trinoo) — one of the first DDoS attacks to make mainstream news. |
| 2007 | **Estonia** suffers a nationwide DDoS campaign against government, banking, and media sites following a political dispute — one of the first DDoS attacks widely framed as having geopolitical/state motivation. |
| 2008–2010 | **Low Orbit Ion Cannon (LOIC)** and the "Anonymous" collective popularize crowd-sourced, voluntary-botnet DDoS as a form of **hacktivism** (Project Chanology, Operation Payback). |
| 2013 | **Spamhaus attack** (~300 Gbps) was, at the time, one of the largest DDoS attacks recorded, using DNS amplification. |
| 2016 | **Mirai botnet** — built from hijacked IoT devices (cameras, routers) — takes down **Dyn** (a major DNS provider), disrupting Twitter, Netflix, Reddit, GitHub, and more. Source code was later published, spawning countless variants. This is widely considered the turning point where IoT devices became a dominant DDoS weapon. |
| 2018 | **GitHub** hit by a 1.35 Tbps attack using **memcached amplification** — at the time the largest attack ever recorded, notable because it required no botnet at all, just reflection. |
| 2020–2023 | Attack volumes continue climbing; **AWS** reports mitigating a 2.3 Tbps attack (2020); Cloudflare and others report multi-terabit and 100M+ requests-per-second "HTTP/2 Rapid Reset" attacks (2023) exploiting a protocol-level flaw in HTTP/2 multiplexing. |

The trend line across 50 years is consistent: **attack volume and sophistication both keep growing**, driven first by better-connected attacker machines, then by botnets, then by amplification techniques, then by the explosion of poorly-secured IoT devices, and most recently by flaws in widely-deployed protocols themselves (like HTTP/2).

## 1.4 Core motives behind DoS/DDoS attacks

Understanding *why* attacks happen helps you reason about *who* is likely to target *what*, which matters for both detection heuristics and your project's threat-modeling section.

- **Hacktivism** — ideological or political protest (e.g., Anonymous campaigns, pro-government or anti-government operations during conflicts).
- **Extortion ("Ransom DDoS" / RDoS)** — attacker demands payment (often cryptocurrency) to stop or avoid an attack, sometimes preceded by a small demonstration attack.
- **Competitive sabotage** — disrupting a rival business, especially common against online gaming services, gambling sites, and e-commerce during peak sales periods.
- **Smokescreening** — using a DDoS attack as a distraction while a separate, quieter intrusion (data exfiltration, fraud) happens elsewhere in the network, consuming the security team's attention.
- **Cyberwarfare / state-sponsored disruption** — nation-state actors disrupting critical infrastructure, media, or government services of an adversary (Estonia 2007, Georgia 2008, Ukraine-related activity since 2014 are commonly cited cases).
- **Personal grudges / "booter" culture** — individuals (sometimes minors) renting cheap "booter"/"stresser" services to attack game servers, rivals in online gaming, or people they have personal disputes with. This is, numerically, one of the most common real-world sources of low-end DDoS activity.
- **Testing/benchmarking gone wrong** — not malicious, but worth knowing: misconfigured load tests against production systems can look identical to an attack from the defender's point of view.

## 1.5 Who gets targeted, and why it matters for your project

- **DNS providers** — taking down DNS for a domain or region cascades into every service relying on it (see Dyn/Mirai, 2016).
- **Web applications and APIs** — e-commerce, gaming, financial services; availability has direct revenue impact.
- **Game servers** — a disproportionate share of real-world DDoS traffic (by attack *count*, not necessarily volume) targets gaming — competitive grudges, cheating disputes, booter-for-hire culture.
- **Critical infrastructure** — power grids, hospitals, emergency services; attacks here carry safety implications, not just financial ones.
- **DNS root/TLD infrastructure and CDNs** — attacking shared infrastructure amplifies blast radius far beyond one organization.

## 1.6 Core networking concepts you need before going further

If you're newer to networking, make sure these are solid before reading `02-attack-types.md`, since attack taxonomy is organized by *which layer/resource* is being exhausted:

- **OSI/TCP-IP layers** — most DoS discussion maps to Layer 3 (Network/IP), Layer 4 (Transport/TCP-UDP), and Layer 7 (Application/HTTP-DNS-etc).
- **TCP three-way handshake** — SYN → SYN-ACK → ACK. Understanding this is essential for SYN flood attacks (§2.2).
- **UDP's connectionless nature** — no handshake, no built-in sender verification, trivially spoofable — essential for amplification attacks (§2.3, §3.2).
- **DNS resolution** — how a small query can trigger a much larger response (the basis of DNS amplification).
- **IP spoofing** — forging the source IP address in a packet header so replies go to a victim instead of the real sender; foundational to reflection/amplification attacks.
- **Bandwidth vs. throughput vs. packets-per-second (PPS)** — volumetric attacks are often measured in Gbps/Tbps, but some attacks (e.g., small-packet floods) are bottlenecked by PPS limits on routers/switches long before bandwidth is saturated — a subtlety worth understanding and mentioning in a project write-up.
- **Connection state tables** — firewalls, load balancers, and servers track open connections in finite memory structures; exhausting this table is a distinct attack class from exhausting bandwidth.
- **Application request cost asymmetry** — the core idea behind Layer 7 attacks: a tiny request (e.g., a search query or login POST) can cost the server far more CPU/DB work to answer than it cost the attacker to send, so even low request-rate attacks can be devastating if they target expensive endpoints.

## 1.7 Key terms glossary (quick reference)

| Term | Meaning |
|---|---|
| **Botnet** | A network of compromised devices ("bots"/"zombies") controlled by an attacker, typically via a Command-and-Control (C2) server. |
| **C2 (Command and Control)** | Infrastructure the attacker uses to issue instructions to bots. |
| **Reflection** | Sending spoofed requests to a third-party server so its *response* goes to the victim instead of the attacker. |
| **Amplification** | A reflection technique where the response is much larger than the request, multiplying the attacker's effective bandwidth (measured as an "amplification factor," e.g., 50x). |
| **Booter / Stresser** | A commercial (often illegal) "DDoS-as-a-service" platform that lets paying customers launch attacks with no technical skill. |
| **Zombie** | An individual compromised machine within a botnet. |
| **Backscatter** | Unsolicited reply traffic received by a spoofed source address as a side effect of an attack — used by researchers to estimate attack activity globally. |
| **Rate limiting** | Restricting how many requests/connections a source (or the system overall) may make in a given time window. |
| **Scrubbing center** | A facility/service that receives suspect traffic, filters out attack traffic, and forwards clean traffic to the origin. |
| **Anycast** | A routing technique where the same IP address is announced from many physical locations, letting traffic (and attack load) be absorbed and spread across a global network. |
| **BCP 38** | Best Current Practice 38 — an IETF recommendation that ISPs filter outbound traffic to block spoofed source IPs at the network edge; if universally adopted it would eliminate most reflection/amplification attacks. |

## 1.8 What "project-ready" understanding looks like

By the end of this guide set you should be able to:

1. Classify any given DoS/DDoS attack into its correct category (volumetric / protocol / application-layer) and explain *which resource* it exhausts and *why*.
2. Explain, at a packet level, how at least one attack from each category works (e.g., SYN flood, DNS amplification, HTTP flood/Slowloris).
3. Describe how such an attack would be detected in traffic (signatures, anomalies, thresholds).
4. Describe at least two mitigation techniques appropriate to each category, and why a mitigation for one category often does nothing for another.
5. Build and demonstrate a small, fully isolated lab showing an attack and its mitigation — see `06-lab-project-guide.md`.

Continue to **`02-attack-types.md`** for the full taxonomy.
