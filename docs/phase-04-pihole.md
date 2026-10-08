# Phase 4 — Pi-hole (DNS Filtering & Query Visibility)

## Objective

Add network-wide DNS filtering, and — more valuable for a detection lab — a single, central record of every domain the lab resolves.

## Build

Deployed **Pi-hole** on the lab network at `10.10.10.10`, with **Quad9** as the upstream resolver and **DNSSEC** enabled for response integrity. pfSense advertises `10.10.10.10` as the DNS server via DHCP, so all lab name resolution funnels through Pi-hole.

## Why it matters beyond ad-blocking

Ad and tracker blocking is the obvious benefit. The security value is **DNS-layer visibility and detection**: Pi-hole logs every domain every host looks up, giving a natural vantage point for spotting **command-and-control (C2) callbacks** — a compromised host beaconing to a suspicious or newly-registered domain surfaces as a DNS query long before anything else. This complements Suricata's packet-level view with an always-on, low-overhead "what is talking to what" record. The Quad9 upstream adds a second layer, blocking known-malicious domains at the resolver before they ever resolve — a form of protective DNS.

**Verification:**
- `doubleclick.net` → `0.0.0.0` — a known tracker domain, correctly sinkholed.
- `google.com` → resolves normally — legitimate traffic passes untouched.

## Real-world relevance

DNS is one of the richest and most under-used telemetry sources in real detection work — malware routinely reveals itself through DNS before any payload moves. A central, filtered, logged resolver is a lightweight but genuinely useful detection and threat-hunting layer, and pairing it with an upstream that enforces protective DNS reflects a defense-in-depth approach.

*Evidence: see `/screenshots` for the Pi-hole dashboard and the sinkhole verification.*
