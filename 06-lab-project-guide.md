# 06 — Lab & Project Guide: Legal Boundaries, Safe Setup, and Project Ideas

This is the practical "how do I actually build something" file. Read **§6.1 (legal/ethical boundaries) in full before building anything** — it's not boilerplate, it genuinely determines what you're allowed to do.

## 6.1 Legal and ethical boundaries — read this first

### 6.1.1 The core rule
**Only ever send attack-style traffic to systems you own or have explicit, documented, written authorization to test.** This is true even if the target "looks fine afterward," even if you only ran it for 10 seconds, and even if your intent was purely educational. In most jurisdictions (e.g., under the U.S. Computer Fraud and Abuse Act, the UK's Computer Misuse Act, and equivalent laws in most other countries), **unauthorized denial-of-service activity against any system you don't own is a criminal offense**, independent of actual damage caused, intent, or the attacker's age (minors have been prosecuted in multiple jurisdictions, including some of the historical cases cited in `01-fundamentals.md` and `03-tools-and-techniques.md`).

### 6.1.2 What this means concretely for your project
- ✅ **Fine**: running attack-simulation traffic between two (or more) virtual machines that **you** created, on a **host-only / internal network** with **no route to the internet or to anyone else's network**.
- ✅ **Fine**: using legitimate load-testing tools (JMeter, k6, Locust) against a web application **you deployed yourself**, even if deployed on a cloud VM you provisioned and own — but check your cloud provider's acceptable use policy first; some providers require advance notice even for load-testing your own instances, because traffic patterns can be indistinguishable from an attack to their own abuse-detection systems and may trigger an account suspension.
- ✅ **Fine**: analyzing **publicly available, pre-captured attack traffic datasets** (CICDDoS2019, etc. — see `04-detection-and-analysis.md` §4.3.5) for a detection/ML project — you're not sending any traffic to anyone, just analyzing recorded data.
- ✅ **Fine**: writing and discussing detection/mitigation code, architecture diagrams, and analysis — none of this requires ever actually launching an attack.
- ❌ **Not fine, under any framing** ("just testing," "just curious," "it's a small server," "I have permission from a friend who doesn't actually own the infrastructure," "it's for a class"): sending attack traffic to any IP, domain, or service you do not own and do not have explicit written authorization to test. This includes school/university network infrastructure unless your institution's security team has explicitly authorized it in writing for your specific project.
- ❌ **Not fine**: using, renting, or even creating an account on a booter/stresser platform, even "just to see how it works" — simply purchasing access to such a service is itself illegal in many jurisdictions, independent of whether you ever use it.
- ❌ **Not fine**: testing against third-party cloud-hosted infrastructure (even "free tier" test servers, demo sites, or practice targets you find online) unless that specific service explicitly documents that it welcomes this kind of testing (a small number of deliberately-built "legal hacking practice" platforms exist for *web vulnerability* practice, but essentially none permit DoS-style load testing, since it would degrade the platform for everyone else using it).

### 6.1.3 If your coursework asks you to attack a real, shared target
If an assignment genuinely asks you to run attack traffic against shared infrastructure (a shared classroom server, a cloud instance the whole class uses, etc.), **get this in writing from your instructor** before proceeding, confirm the exact scope (which IPs, which time window, which attack types are permitted), and keep that written authorization — this is standard practice in real-world penetration testing too (a signed "Rules of Engagement" / authorization letter is a prerequisite for any legitimate pentest), so documenting it is itself good practice to demonstrate in your project.

### 6.1.4 Why this guide won't help you skip this step
Even with good intentions, this guide will not provide ready-to-run attack scripts targeting anything other than your own fully isolated lab, won't help identify real-world exposed amplification reflectors, and won't provide booter/stresser usage instructions. This isn't about distrust of you specifically — it's a consistent line this guide holds regardless of who's asking, because "I'm doing a class project" is indistinguishable, from a text standpoint, from any other stated intent, and the instructions themselves would be equally usable either way. Everything you need for a genuinely strong, hands-on project is achievable within the isolated-lab and dataset-analysis boundaries below.

## 6.2 Building a fully isolated lab (the recommended hands-on core of your project)

The goal: a self-contained environment where you control both the "attacker" and "victim" machines, with **no network path to the internet or to anyone else**, so you can generate, observe, and defend against real attack traffic patterns safely.

### 6.2.1 Recommended setup

- **Virtualization**: VirtualBox, VMware Workstation/Fusion, or (if comfortable with containers) Docker Compose with an isolated bridge network. VirtualBox's **"Host-only Adapter"** or **"Internal Network"** mode is ideal — it creates a network segment your VMs can use to talk to each other with **zero route out to the internet or your host's LAN**.
- **Minimum VM set**:
  - 1x "target" VM — a lightweight Linux distro (Ubuntu Server or Debian are fine) running a simple web server (nginx or Apache) and, optionally, a small test application (even a basic Flask/Node app with one deliberately "expensive" endpoint, like one that does a slow database query or heavy computation, to demonstrate the cost-asymmetry concept from `02-attack-types.md` §2.3.1).
  - 1–3x "attacker" VM(s) — same lightweight distro, used to generate load/attack-style traffic toward the target.
  - 1x "monitor" VM (optional but recommended) — runs your packet capture and detection tooling (tcpdump, Suricata, Grafana) with a network interface that can see the traffic between attacker and target (e.g., configure the hypervisor's internal network to also attach to this monitoring VM, or run it on the target VM itself for simplicity).
- **Confirm isolation before doing anything else**: from an attacker VM, try to ping `8.8.8.8` or any public IP — it should **fail**. If it succeeds, your network isn't isolated yet; fix this before generating any attack-style traffic, as a basic safety check.

### 6.2.2 What to actually generate in the lab

Using **only legitimate load-testing tools**, against **only your own target VM**:

- **Volumetric-style simulation**: `hping3` (preinstalled on many security-focused Linux distros, or installable via apt) can craft high-rate TCP/UDP/ICMP packet floods — use it against your own isolated target VM to observe bandwidth/PPS saturation and SYN-to-ACK ratio shifts.
- **SYN flood simulation**: `hping3 -S --flood -p 80 <target-internal-ip>` (run only inside your isolated network) demonstrates half-open connection buildup; watch it with `netstat -n | grep SYN_RECV` or `ss -tan state syn-recv` on the target.
- **Slowloris-style simulation**: there are open-source Slowloris proof-of-concept scripts intended specifically for authorized testing of your own servers; running one against your own isolated nginx/Apache instance lets you directly observe worker-thread/connection-slot exhaustion with minimal bandwidth.
- **Application-layer load**: use **Apache JMeter**, **Locust**, or **k6** (all legitimate, widely-used load-testing tools) to ramp up request rate against your target app's endpoints, including your deliberately "expensive" endpoint — this demonstrates the cost-asymmetry concept concretely and produces clean, explainable request-rate graphs for your report.

### 6.2.3 What to measure and show in your report
Tie this directly back to `04-detection-and-analysis.md` §4.2 — for each simulated attack, capture and graph:
- Target CPU/memory utilization over time (simple: `top`/`htop` logging, or a Prometheus node-exporter + Grafana panel).
- SYN-to-ACK ratio over time, for the SYN flood scenario.
- Concurrent connection count and average connection duration, for the Slowloris scenario.
- Requests-per-second and average response time together on one chart, for the application-layer scenario — showing the point where response time starts climbing is a strong visual.
- A "before mitigation" and "after mitigation" comparison — e.g., re-run the SYN flood scenario after enabling SYN cookies (`sysctl -w net.ipv4.tcp_syncookies=1` on Linux) and show the backlog queue no longer fills up; re-run the application-layer scenario after adding a basic rate-limiting rule (e.g., nginx's `limit_req` module) and show RPS capped while response time stays stable. **This before/after comparison is usually the single most impressive, gradable artifact you can produce** — it demonstrates you understand not just the attack but why the specific defense works.

## 6.3 Dataset-based project track (no live traffic generation needed)

If you'd rather focus on analysis/ML over infrastructure building, or your coursework timeline doesn't allow for a full lab build:

1. Download **CICDDoS2019** or **CICIDS2017** (Canadian Institute for Cybersecurity, publicly available for research use).
2. Pick 2–4 attack categories represented in the dataset (e.g., a volumetric one like UDP flood, a protocol one like SYN flood, and an application-layer one if present).
3. Do exploratory data analysis first — plot the distributions of flow duration, packet size, and inter-arrival time for benign vs. each attack class, and *explain what you see in terms of the mechanics from `02-attack-types.md`* (e.g., "the SYN flood flows show near-zero duration because the handshake never completes, consistent with the half-open connection mechanic described in §2.2.1").
4. Train at least two different classifiers (e.g., Random Forest and a simple neural network, or Random Forest vs. logistic regression as a baseline) and report **per-class precision/recall/F1**, not just accuracy (see the false-positive note in `04-detection-and-analysis.md` §4.3.5 — this is a common grading criterion precisely because it's a common student mistake to skip).
5. Discuss which features mattered most (feature importance from a tree-based model is easy to extract and makes for a strong report figure) and connect that back to the mechanics — e.g., if SYN/ACK flag counts dominate importance for detecting SYN floods, explain *why* that lines up with the attack's mechanics.

## 6.4 Suggested project structures by scope/time budget

| Time budget | Suggested scope |
|---|---|
| Short (1–2 weeks, overview-level) | Literature review + taxonomy write-up (using `01`–`03`) + one simple isolated-lab demo (one attack, one defense, before/after) |
| Medium (3–5 weeks, standard course project) | Full isolated lab covering one attack from each category (volumetric/protocol/app-layer) + detection dashboard + before/after mitigation for each |
| Long (full semester / capstone) | Medium scope + the dataset/ML detection track in §6.3, combined into one report — hands-on lab demonstrates mechanics and mitigation, ML track demonstrates detection-at-scale thinking |

## 6.5 Suggested report/deliverable structure

1. **Introduction** — what DoS/DDoS is, why it matters (draw from `01-fundamentals.md` §1.1–1.5).
2. **Taxonomy** — your own synthesis of the attack categories (draw from `02-attack-types.md`), ideally with your own diagram, not copy-pasted.
3. **Methodology** — your lab architecture (diagram it — isolated network topology is an easy, clean diagram to produce) and/or dataset and ML pipeline.
4. **Results** — the before/after graphs from §6.2.3, or the classifier metrics from §6.3.
5. **Discussion** — what surprised you, what the limitations of your approach are (be honest — e.g., "my lab network is far smaller/less noisy than real internet traffic, so detection thresholds that work here might not transfer directly"), and how this connects to real-world mitigation (draw from `05-mitigation-and-defense.md`).
6. **Ethics/legal section** — a short, explicit statement of your lab's isolation and authorization boundaries (draw from §6.1) — including this explicitly is good practice and, in many courses, directly rewarded in grading rubrics for security projects.
7. **References** — cite primary sources where possible: vendor incident write-ups (Cloudflare, AWS, GitHub, Akamai all publish detailed technical post-mortems of major attacks they've mitigated — these are excellent, citable, primary-source material), CERT/NIST advisories, and the academic papers behind any dataset you use.

## 6.6 Common pitfalls to avoid (things that commonly cost students marks)

- Treating DDoS as "just send lots of traffic" without engaging with the taxonomy — markers specifically look for whether you understand *which resource* is exhausted and *why* a given defense works, not just that an attack "worked."
- Reporting only overall accuracy on an imbalanced ML dataset (see §6.3 point 4) — a trivial "always predict benign" model can score 90%+ accuracy on some of these datasets while being useless.
- Skipping the before/after mitigation comparison — showing an attack "succeeding" is half a project; showing you understand and can demonstrate the fix is usually weighted more heavily.
- No isolation/authorization statement — even in an internal lab project, explicitly stating your scope and isolation demonstrates the professional practice real security work requires, and its absence is a common, easily-avoided deduction.
- Overclaiming — a project that says "my detection system stops all DDoS attacks" undermines its own credibility; the field's experts are explicit that no defense is complete (see `05-mitigation-and-defense.md` §5.9), and a project that shows that same honesty about limitations generally reads as more rigorous, not weaker.

---

This completes the guide set. Suggested next step: skim back through `00-INDEX.md`, pick your project track from §6.4 above, and start with the isolated-lab network setup in §6.2.1 or the dataset download in §6.3 step 1.
