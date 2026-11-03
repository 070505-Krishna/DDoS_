# 05 — Mitigation & Defense

No single control stops every DoS/DDoS category (see `02-attack-types.md` §2.5 on multi-vector attacks), so real-world defense is built as **layered, defense-in-depth architecture**, with different controls targeting different categories. This file walks through that architecture from the network edge inward, then covers category-specific mitigations, then two real-world case studies.

## 5.1 The defense-in-depth model

```
 Internet
    │
    ▼
┌─────────────────────────┐
│  1. Upstream/ISP         │  ← BCP 38 filtering, blackhole routing
│     filtering             │
└───────────┬───────────────┘
            ▼
┌─────────────────────────┐
│  2. Scrubbing/CDN layer   │  ← Anycast absorption, traffic scrubbing,
│     (Cloudflare, Akamai, │     rate limiting at the edge
│      AWS Shield, etc.)    │
└───────────┬───────────────┘
            ▼
┌─────────────────────────┐
│  3. Perimeter network     │  ← Firewall/IPS, SYN cookies, connection
│     defenses               │     rate limiting
└───────────┬───────────────┘
            ▼
┌─────────────────────────┐
│  4. Application layer      │  ← WAF, CAPTCHA/bot-management,
│     defenses                │     application-level rate limiting,
│                              │     caching, graceful degradation
└───────────┬───────────────┘
            ▼
       Origin server
```

The guiding principle: **stop an attack as far upstream, and as cheaply, as possible** — it's far better to drop forged DNS amplification traffic at an ISP's edge router than to let it reach your origin server and let your application-layer WAF try to cope with it.

## 5.2 Layer 1: Upstream / ISP-level defenses

- **BCP 38 / Ingress-Egress filtering**: ISPs filter outbound traffic from their customers to drop packets with a source IP that couldn't legitimately originate from that customer's network. If every network on the internet implemented this, **IP spoofing — and therefore essentially all reflection/amplification attacks — would become impossible**. Adoption remains incomplete decades after the recommendation, which is itself a well-documented and citable case study in why "solved in theory" security problems can remain unsolved in practice (fragmented ownership of the internet, no single enforcing authority, and the cost/benefit being asymmetric — the network implementing the filtering gets little direct benefit, while the *victims* elsewhere get the protection).
- **Remote-Triggered Black Hole (RTBH) routing**: when an attack is detected, the ISP (or the target, via BGP signaling, e.g. RFC 7999's "blackhole community") announces a route that silently drops all traffic to the victim IP at the network edge. Effective but blunt — it takes the target **fully offline** rather than just blocking attack traffic, which is why it's typically a last-resort or short-term measure.
- **Flowspec (BGP Flowspec)**: a more surgical alternative to RTBH — lets a network operator push fine-grained filtering rules (match on protocol, port, packet size, etc., not just destination IP) to routers via BGP, enabling targeted mitigation without fully blackholing the victim.

## 5.3 Layer 2: Scrubbing centers, CDN, and Anycast

- **Scrubbing centers**: traffic is rerouted (via DNS or BGP announcement changes) through a specialized facility with massive bandwidth capacity and purpose-built filtering hardware/software; clean traffic is forwarded on to the origin, attack traffic is dropped. This is the core service model of providers like Cloudflare, Akamai, AWS Shield, and Azure DDoS Protection.
- **Anycast routing**: the same IP address is announced from many geographically distributed points of presence (PoPs); incoming traffic (legitimate or attack) is routed to the *nearest* PoP by normal internet routing (BGP), which means a volumetric attack's traffic is automatically spread across dozens or hundreds of global locations instead of concentrating on one data center — each PoP only has to absorb a fraction of the total attack. This is a major reason why large CDNs can credibly claim to absorb multi-terabit attacks that would instantly flatten a single-data-center origin.
- **Overprovisioning**: simply having far more bandwidth/compute capacity than normal peak demand requires, as a buffer against attacks — an expensive but simple mitigation, more often used as a secondary layer than a primary strategy today given how large modern attacks can get.

## 5.4 Layer 3: Perimeter/network defenses

- **SYN cookies** (direct countermeasure to SYN floods, see `02-attack-types.md` §2.2.1): instead of allocating backlog-queue memory for every SYN received, the server encodes the connection's state into the initial sequence number it sends back in the SYN-ACK, cryptographically, and allocates no memory until the client's final ACK arrives and proves it knows that encoded value — meaning spoofed SYNs (which never get ACK'd) cost the server almost nothing to handle.
- **Connection rate limiting** — capping new-connections-per-second per source IP (or per subnet, to counter distributed-but-clustered sources) at the firewall/load-balancer level.
- **Stateful firewall tuning** — larger backlog queues, shorter SYN-RECEIVED timeouts, and aggressive dropping of clearly invalid flag combinations (e.g., a packet with both SYN and FIN set).
- **IPS signature rules** — Suricata/Snort rulesets that match known attack-tool fingerprints or protocol anomalies (see `04-detection-and-analysis.md` §4.3.3).

## 5.5 Layer 4: Application-layer defenses

- **Web Application Firewall (WAF)** — inspects HTTP requests for malicious patterns and can enforce rate limits per endpoint, per session, or per client fingerprint; modern WAFs (Cloudflare, AWS WAF, ModSecurity) increasingly incorporate the bot-management and TLS-fingerprinting techniques from `04-detection-and-analysis.md` §4.2.3.
- **CAPTCHA / proof-of-work challenges** — forces the client to complete a task (solve a visual puzzle, or compute a small cryptographic proof-of-work) before a request is served; proof-of-work is particularly interesting because it imposes a real, scalable CPU cost on the *attacker's* side proportional to request volume, directly countering the usual cost asymmetry that makes Layer 7 attacks effective (see `02-attack-types.md` §2.3.1).
- **Caching and CDN edge-caching** — serving cacheable content from the edge means a flood against a cacheable page never even reaches the origin server; this is also exactly why attackers deliberately target *uncacheable, dynamic, expensive* endpoints (search, login, checkout) instead.
- **Rate limiting at the application level** — per-API-key, per-session, or per-IP limits on specific expensive endpoints, often with a "token bucket" or "leaky bucket" algorithm (worth explaining in a project: a token bucket allows short bursts up to a cap while enforcing a steady average rate over time, which is usually a better fit for real user behavior than a hard per-second cap).
- **Graceful degradation / load shedding** — intentionally designing the application to drop or defer non-critical functionality under load (e.g., disable personalized recommendations, serve a simplified static page, queue non-urgent background jobs) so that core functionality survives even when the system is under stress — a resilience-engineering concept adjacent to pure security that's worth including if your project touches system design.

## 5.6 Mitigation mapped back to the taxonomy (quick reference)

| Attack category | Most effective primary defenses |
|---|---|
| Volumetric / amplification | BCP 38 (root-cause prevention), scrubbing centers, Anycast, RTBH as last resort |
| Protocol / state-exhaustion | SYN cookies, connection rate limiting, shorter timeouts, IPS signatures |
| Application-layer | WAF, rate limiting, CAPTCHA/proof-of-work, caching, behavioral/TLS fingerprinting |

Note how **none of these overlap cleanly** — SYN cookies do nothing against an HTTP flood; a WAF does nothing against a pure volumetric UDP flood big enough to saturate the pipe before packets even reach the WAF. This is the single most important takeaway for a project's "defense" section: **effective DDoS defense is necessarily layered, because the attack categories exploit fundamentally different resources.**

## 5.7 Case study 1: Dyn / Mirai, 2016

- **What happened**: the Mirai botnet (built from compromised IoT devices — cameras, DVRs, home routers, compromised mainly via default Telnet credentials) launched a massive, multi-vector attack against Dyn, a major managed DNS provider.
- **Why it cascaded so badly**: because many major sites (Twitter, Netflix, Reddit, GitHub, Spotify, and more) all relied on Dyn for DNS resolution, taking down *one* provider effectively took down dozens of unrelated, otherwise-healthy services — a textbook illustration of **shared-dependency risk** and why DNS, as "the internet's phone book," is a uniquely high-leverage target.
- **Lessons commonly drawn**: (1) IoT security (default credentials, lack of patching) is a systemic internet-wide risk, not just a risk to the device owner; (2) relying on a single DNS provider with no failover is a single point of failure worth architecting around (multi-provider DNS became a more commonly recommended practice afterward); (3) source code for Mirai was later published publicly, which both helped researchers study it and, controversially, enabled many copycat botnets.

## 5.8 Case study 2: GitHub / Memcached, 2018

- **What happened**: GitHub was hit with a 1.35 Tbps attack — at the time the largest ever recorded — using **memcached amplification** reflected off publicly exposed memcached servers (memcached is meant for internal, trusted-network caching, not public exposure, and its UDP interface had no authentication).
- **Why it's notable**: unlike Mirai/Dyn, this attack required **no botnet at all** — just a list of exposed, misconfigured memcached servers and spoofed UDP packets, demonstrating that amplification-based attacks don't require the attacker to control any significant infrastructure themselves.
- **Mitigation**: GitHub was already using a commercial DDoS mitigation/scrubbing service (Akamai Prolexic) and was rerouted into scrubbing within roughly 10 minutes, limiting impact to about that window of partial unavailability. Following the attack, the security community pushed major cloud/hosting providers to identify and notify operators of exposed memcached instances, and the memcached project changed its default configuration to disable the UDP interface.

## 5.9 A nuance worth including in a project: mitigation cost and tradeoffs

It's tempting to present mitigation as a purely technical problem, but a stronger project acknowledges the real tradeoffs:

- Scrubbing/CDN services cost money, sometimes significant amounts at the volumes needed for large-attack protection — smaller organizations often can't afford enterprise-tier DDoS protection, which is part of why free/low-cost tiers from major CDNs (and nonprofit initiatives like Google's **Project Shield**, aimed at protecting journalism and human-rights sites from DDoS) matter as an equity issue in the field, not just a technical one.
- Aggressive rate limiting and CAPTCHA challenges create real friction for legitimate users — a security control that drives away genuine customers is, from a business standpoint, sometimes indistinguishable from a successful attack. "Security vs. usability" is a legitimate tension worth naming, not hand-waving past.
- No defense is permanent — mitigation is an arms race (literally illustrated by the HTTP/2 Rapid Reset case in `02-attack-types.md` §2.3.2, where a previously-unexploited protocol feature became a major attack vector overnight). A good project conclusion acknowledges that today's taxonomy will keep evolving, rather than presenting any current defense as a final solved state.

Continue to **`06-lab-project-guide.md`** to turn this material into an actual hands-on project, safely and legally.
