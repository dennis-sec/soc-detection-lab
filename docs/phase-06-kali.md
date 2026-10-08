# Phase 6 — Kali Linux (Attack Platform)

## Objective

Deploy the offensive workstation used to test the lab's defences — the attacker's side of the purple-team exercise.

## Build

Provisioned **Kali Linux 2026.1** on the operator network (`10.10.10.100`) with host CPU passthrough for performance. Kali is the single platform from which all reconnaissance, exploitation, and post-exploitation is launched — no tooling is installed on targets; everything is directed at them over the network, exactly as a real external attacker would operate.

## Network posture — isolation confirmed

Kali reaches the internet through pfSense but **cannot reach the home network** — the same default-deny containment boundary verified in Phase 3 applies to it. This keeps all offensive activity inside the lab: the attack platform can reach its targets and the internet, but never the home LAN.

## Real-world relevance

A realistic detection lab needs a credible adversary. Kali provides the industry-standard offensive toolset, and confining it to the isolated lab reflects the controlled, authorised scope of real red-team and penetration-testing engagements. This workstation drives the full attack in Phase 11 that the entire detection stack is measured against.

*Evidence: see `/screenshots` for Kali on the lab network and the isolation check.*
