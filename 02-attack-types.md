# 02 — Attack Taxonomy: Every Major DoS/DDoS Category

DoS/DDoS attacks are best organized by **which resource they exhaust** and **which OSI layer they operate at**. This file walks through all three major categories, with packet/protocol-level mechanics for each named attack. This is the core reference file — expect to come back to it repeatedly while reading `03` through `06`.

```
┌─────────────────────────────────────────────────────────┐
│                     DoS / DDoS ATTACKS                    │
├───────────────────┬───────────────────┬───────────────────┤
│  VOLUMETRIC        │  PROTOCOL /        │  APPLICATION-LAYER │
│  (Layer 3/4)        │  STATE-EXHAUSTION  │  (Layer 7)         │
│                     │  (Layer 3/4)        │                     │
│  Goal: saturate      │  Goal: exhaust     │  Goal: exhaust      │
│  bandwidth/PPS       │  connection state  │  app logic/compute  │
│                     │  tables on servers │  per request         │
│                     │  & middleboxes      │                     │
└───────────────────┴───────────────────┴───────────────────┘
```

A single real-world DDoS campaign very often **combines attacks from two or three of these categories simultaneously** ("multi-vector attacks") specifically to defeat defenses tuned for only one category.

---

## 2.1 Category A: Volumetric Attacks

**Goal:** saturate the available bandwidth (or packets-per-second capacity) between the internet and the target, so legitimate traffic physically cannot get through — this is a "clogged pipe" problem, not a logic problem.

### 2.1.1 UDP Flood
The attacker (or botnet) sends a high volume of UDP packets to random or specific ports on the target. Because UDP is connectionless, the target's OS checks each packet for a listening application on that port; if none exists, it must generate an ICMP "Destination Unreachable" reply — this processing, multiplied across millions of packets per second, consumes CPU and bandwidth even before considering the inbound flood itself.

### 2.1.2 ICMP Flood ("Ping Flood")
Large volumes of ICMP Echo Request ("ping") packets are sent to the target, which (if configured to reply) must generate an Echo Reply for each one, consuming bandwidth and processing in both directions.

### 2.1.3 Amplification/Reflection Attacks (the dominant volumetric technique today)
This is the single most important mechanic to understand in this entire guide, because it explains how attackers achieve multi-terabit attacks without owning a multi-terabit botnet.

**Mechanics:**
1. The attacker finds publicly accessible servers running a UDP-based protocol that returns a **much larger response** than the request (DNS, NTP, memcached, CLDAP, SSDP, etc. — see `03-tools-and-techniques.md` §3.2 for the full protocol table).
2. The attacker sends a small request to each of these servers, but **spoofs the source IP address** to be the victim's IP instead of their own.
3. Each server, believing the request genuinely came from the victim, sends its (much larger) reply **to the victim**.
4. Multiplied across thousands of reflector servers, the victim is flooded with reply traffic it never requested — and because the traffic comes from *legitimate, real* DNS/NTP/etc. servers, it's much harder to filter by "is this source IP malicious" than a direct flood would be.

**Why it works (two independent flaws stacking):**
- **UDP has no handshake**, so there is no built-in way for the responding server to verify the request's source address actually sent it (unlike TCP, where a handshake would fail against a spoofed address because the attacker can't see the SYN-ACK to complete it).
- **IP source-address spoofing is possible** because many networks don't filter outbound packets for forged source addresses (this is exactly what BCP 38 / ingress filtering is meant to prevent, and its incomplete global adoption is why this attack class still works in 2026).

**Amplification factor** = (response size ÷ request size). This varies wildly by protocol and misconfiguration — see the protocol table in `03-tools-and-techniques.md`; some memcached-based attacks have recorded amplification factors over 50,000x.

### 2.1.4 NTP Amplification (a canonical, well-documented example worth knowing in detail)
The NTP `monlist` command (available in older `ntpd` versions) returns a list of the last ~600 machines that queried the server — a tiny request can produce a response hundreds of times larger. This was one of the most widely abused amplification vectors from roughly 2013–2015 before most servers patched or disabled `monlist`.

### 2.1.5 Fragmentation attacks (e.g., "Teardrop")
Exploit the IP fragmentation/reassembly process by sending malformed or overlapping packet fragments that confuse the target's reassembly logic, consuming CPU/memory or, in older/vulnerable stacks, crashing the system outright. Largely historical against modern OSes but conceptually important and still appears in multi-vector attacks as a secondary stressor.

---

## 2.2 Category B: Protocol / State-Exhaustion Attacks

**Goal:** exhaust finite state-tracking resources — connection tables, session tables, half-open connection buffers — on servers, load balancers, and stateful firewalls. These attacks often need far less raw bandwidth than volumetric attacks because they target a *structural limit*, not a pipe size.

### 2.2.1 SYN Flood (the classic, must-know-cold example)
**Mechanics, step by step:**
1. Normal TCP handshake: Client sends **SYN** → Server replies **SYN-ACK** and allocates a small amount of memory for this "half-open" connection in its backlog queue → Client replies **ACK**, completing the handshake.
2. In a SYN flood, the attacker sends a huge volume of SYN packets, often with **spoofed, non-existent, or unreachable source IPs**.
3. The server dutifully replies SYN-ACK to each one and holds the half-open connection in its backlog queue, waiting for an ACK that will **never come** (because the source address was spoofed or the attacker simply never sends it).
4. The backlog queue — a finite table — fills up. Once full, the server **cannot accept new legitimate connections**, even though CPU and bandwidth may both still have headroom. This is why SYN floods are so efficient: a relatively modest packet rate can take down a server that would easily survive far more *bandwidth*.

**Defenses specific to this attack** (detailed further in `05-mitigation-and-defense.md`): **SYN cookies** (the server encodes connection state into the SYN-ACK sequence number itself instead of storing it, so no backlog slot is consumed until the ACK proves the handshake is real), shorter SYN-RECEIVED timeouts, and larger backlog queues as a stopgap.

### 2.2.2 ACK Flood / Other TCP flag floods
Sends TCP packets with the ACK (or RST, FIN, PSH) flag set, out of the context of any real connection. Stateful firewalls and servers must check each packet against their connection table to determine if it belongs to an existing session, burning CPU even when the packet is ultimately dropped as invalid.

### 2.2.3 Connection Exhaustion / "Slow" Attacks
Rather than sending huge volume, these attacks open many connections and **keep them open as long as possible using minimal resources**, exhausting the server's maximum-concurrent-connections limit.

- **Slowloris**: opens many HTTP connections to the target and sends **partial HTTP headers**, very slowly (e.g., one header line every ~10–15 seconds), never completing the request. Most web servers hold the connection open waiting for the rest of the request, so a modest number of slow connections (sometimes only a few hundred) can exhaust the server's worker/thread pool — notably, this attack needs almost **no bandwidth at all**, making it attractive from a single machine and hard to catch with volume-based detection.
- **Slow POST (R-U-Dead-Yet / RUDY)**: sends a legitimate HTTP header declaring a large `Content-Length`, then trickles the POST body in at a very slow rate, keeping the connection open for the duration.
- **Slow Read**: completes the request normally but reads the *response* extremely slowly (via manipulating the TCP receive window), forcing the server to hold the connection and buffered response data open.

### 2.2.4 SSL/TLS Exhaustion
Exploits the fact that the TLS handshake (key exchange, certificate processing) is computationally expensive for the **server** but cheap for the client to *initiate*. Repeatedly opening TLS handshakes (or sending malformed handshake data, or endlessly renegotiating) can exhaust server CPU asymmetrically.

---

## 2.3 Category C: Application-Layer (Layer 7) Attacks

**Goal:** exhaust the application's own logic/compute/database capacity using requests that are, individually, indistinguishable from legitimate traffic — this is the hardest category to detect and mitigate, because there's no "obviously malformed packet" to filter on.

### 2.3.1 HTTP Flood
The attacker (usually via botnet) sends a large volume of ordinary-looking HTTP GET/POST requests. Two sub-variants:
- **Volume-based HTTP flood**: brute volume of requests overwhelms web server or app server capacity.
- **Resource-targeted HTTP flood**: requests deliberately target expensive endpoints — search functions, report generation, large file downloads, login forms (which often trigger expensive password-hashing operations), or endpoints that trigger uncached database queries. A few thousand requests per second against an expensive, uncached search endpoint can do more damage than a hundred thousand requests per second against a static cached homepage. This "cost asymmetry" is the central idea of Layer 7 attacks.

### 2.3.2 HTTP/2 Rapid Reset (CVE-2023-44487 — a critical, recent, and genuinely advanced example worth understanding for a modern project)
Discovered/disclosed in 2023, this exploits a feature of the HTTP/2 protocol itself (multiplexed streams over one TCP connection):
1. The client opens an HTTP/2 connection and rapidly sends a request on a new stream, then **immediately cancels it** (via an `RST_STREAM` frame) before the server has finished processing.
2. The client repeats this thousands of times per second over the **same single TCP connection**.
3. Because the stream is cancelled, it never counts against the server's "max concurrent streams" limit for long, but the server still did real work (allocating resources, dispatching to backend logic) for each one before the cancellation landed.
4. This allows request rates far beyond what connection-count-based rate limiting expects, since a single connection can generate an enormous number of effective "requests." Cloudflare, Google, and AWS all reported mitigating attacks exceeding 100 million requests/second using this technique in 2023 — at the time, the largest HTTP attacks ever recorded by request rate.

This is an excellent case study for a project because it shows DoS attacks evolving **with the protocols themselves**, not just with more bots.

### 2.3.3 DNS Query Flood
Floods a DNS server/resolver with a high volume of queries, often for **random, non-existent subdomains** (a variant sometimes called a "Water Torture" or "NXDOMAIN flood" attack), forcing the resolver to perform expensive recursive lookups that can't be served from cache, exhausting its resolution capacity and sometimes cascading to authoritative nameservers upstream.

### 2.3.4 Application Logic / Algorithmic Complexity Attacks
Exploits worst-case algorithmic complexity in application code — e.g., submitting input deliberately crafted to trigger O(n²) or worse behavior in a parsing routine, regex engine (**ReDoS** — Regular Expression Denial of Service, where a crafted string causes catastrophic backtracking in a vulnerable regex), or hashing function with poor collision resistance. A single well-crafted request can consume disproportionate CPU — this blurs into the "vulnerability exploitation" world but is fundamentally a DoS outcome.

### 2.3.5 XML/XXE-based Resource Exhaustion ("Billion Laughs" / XML Entity Expansion)
A small, deeply-nested XML document using entity references that each expand into multiple further entities can cause an XML parser to attempt to expand a tiny file into gigabytes of in-memory data, exhausting memory. Classic example of **small input, huge resource cost**.

---

## 2.4 Comparison table: how to tell these apart quickly

| Category | Typical metric | Primary resource exhausted | Needs huge botnet? | Spoofable source? | Example |
|---|---|---|---|---|---|
| Volumetric | Gbps / Tbps, PPS | Network bandwidth | Often, unless amplified | Yes (amplification depends on it) | DNS amplification |
| Protocol/State | Connections/sec, half-open count | Connection state tables | Sometimes — Slowloris needs very few sources | Sometimes (SYN flood) | SYN flood, Slowloris |
| Application-layer | Requests/sec, expensive-endpoint hit rate | CPU, database, app logic | Not necessarily — single advanced client can hurt | No, usually needs real TCP/TLS session | HTTP/2 Rapid Reset, ReDoS |

## 2.5 Why multi-vector attacks are now the norm

Modern DDoS-for-hire platforms and advanced botnets rarely launch a single attack type. A typical real-world campaign might open with a **volumetric UDP flood** to saturate upstream bandwidth and distract the SOC, follow with a **SYN flood** to exhaust connection tables on anything that survives the volumetric wave, and finish with a targeted **HTTP flood** against a known-expensive API endpoint to finish off anything still standing behind a CDN/WAF. This is precisely why `05-mitigation-and-defense.md` is organized around **defense-in-depth** rather than a single silver-bullet control — no single mitigation stops all three categories at once.

Continue to **`03-tools-and-techniques.md`** for how these attacks are actually built and launched at scale.
