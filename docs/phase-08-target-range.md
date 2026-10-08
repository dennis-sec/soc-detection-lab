# Phase 8 — Isolated Target Range (Attack-Range Segmentation)

## Objective

Build the dedicated network the vulnerable target will live on — walled off from the internet and the home network, reachable by the attacker so it can be tested, able to report to the SIEM, and routed through pfSense so the firewall and IDS can act on its traffic.

## Why a separate segment — driven by a Phase 7 finding

This design comes directly from a Phase 7 result: two hosts on the same virtual bridge are switched directly to each other, so their traffic never crosses pfSense and the IDS cannot inspect it. Placing the attacker and the target on the *same* segment would therefore leave Suricata blind to the entire exercise. The fix is to give the target its own separate, **routed** segment so every attack crosses pfSense and becomes inspectable.

## Addressing

- Operator network: `vmbr1`, `10.10.10.0/24`, gateway `10.10.10.1` — Kali, Pi-hole, Wazuh.
- **Target range (new): `vmbr2`, `10.10.20.0/24`, gateway `10.10.20.1`**, DHCP pool `10.10.20.100–200`.

## What was built

A new bridge, **`vmbr2`**, created with **no host IP at all** — unlike the operator segment, the hypervisor is given zero presence on this "risky" network. pfSense was given a third virtual NIC on it and configured as the segment's gateway: a **static** IPv4 address (`10.10.20.1`), no upstream gateway (only WAN has one), with its own DHCP server serving the pool.

## The firewall rules — the isolation design

Three rules on the target interface govern what the target may **initiate** (rules are evaluated on the interface traffic enters, and the target's outbound traffic enters here):

1. **Allow → Wazuh manager** (`10.10.10.101`, ports 1514–1515) — so host telemetry could report to the SIEM.
2. **Allow → Kali** (`10.10.10.100`, any protocol) — so an exploit's **reverse shell can call back**. Without this, an exploit lands but the shell silently never connects — one of the hardest failures to diagnose.
3. **Block → everything else, logged** — placed last, this is the isolation wall: no internet, no home network, no other lab hosts, not even pfSense's own web UI.

A note on NAT that is easy to worry about: outbound NAT is set to Automatic, so pfSense does create a NAT entry for the segment — but this grants **no** internet access, because **pfSense evaluates firewall rules before NAT**. Rule 3 drops an outbound packet before it ever reaches the NAT stage. The firewall block is what enforces "no internet," and every escape attempt is logged into the Phase 7 pipeline, surfacing in Wazuh.

Nothing needed changing on the operator (LAN) side — its existing allow rule already permits Kali to reach the target, so only the target's outbound side had to be constrained.

## Design principle — isolated, but not from monitoring

The target is isolated from the internet and the home network, but deliberately **not** from the attacker or the SIEM. A fully sealed, dead-end network would defeat the exercise — attacks against an unwatched, unreachable box teach nothing. The correct posture is: no route out, a path in from the attacker, a path to the SIEM, and all of it routed through pfSense where the firewall and IDS can act.

## Real-world relevance

This is micro-segmentation and egress filtering applied to a deliberately high-risk asset — the same containment thinking used to isolate DMZ hosts, malware-analysis sandboxes, or any system that must be reachable for a purpose but prevented from reaching anything else. Reasoning about rule order, ingress evaluation, and the firewall-before-NAT path is core firewall-administration knowledge.

*Evidence: see `/screenshots` for the VULNHUB interface, the DHCP configuration, and the three firewall rules with the block rule last.*
