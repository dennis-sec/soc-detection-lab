# Phase 3 — pfSense, Suricata & WireGuard (The Security Core)

## Objective

Build the defensive backbone of the lab: a perimeter firewall and router, a network intrusion detection sensor with custom detections, and encrypted remote access — each verified rather than assumed.

## pfSense — Firewall & Router

Deployed **pfSense CE 2.8.1** as the perimeter. Its WAN interface pulls an address via DHCP from the home BT hub; its LAN interface is the lab gateway at `10.10.10.1`, serving a DHCP pool to the segment. The defining control is a **default-deny rule from the lab to the home network**: the lab reaches the internet through pfSense, but cannot initiate to home devices. This is the primary containment boundary — it stops a compromised lab host from pivoting laterally into the home LAN.

**Verification (bidirectional isolation):**
- Kali to home gateway (`192.168.1.254`): **100% packet loss** — blocked, as intended.
- Kali to `8.8.8.8`: **0% loss** — internet reachable via WAN NAT.
- Kali to `google.com`: resolves normally.

Proving the rule works in the exact direction it should — home unreachable, internet reachable — is what turns "I wrote a firewall rule" into "I confirmed the containment boundary holds."

## Suricata — Network Intrusion Detection

Deployed **Suricata** in IDS (alert-only) mode on the LAN, running the **ET Open** ruleset for broad, professionally-maintained coverage, augmented with **custom detection rules** of my own — the detection-engineering core of the lab. The signature leaned on for validation catches nmap SYN scans:

    alert tcp any any -> $HOME_NET any (msg:"LOCAL Nmap SYN Scan Detected"; flags:S; threshold:type both,track by_src,count 20,seconds 5; classtype:attempted-recon; sid:1000001; rev:1;)

Read plainly: it fires when a single source sends 20 or more bare SYN packets to the protected network inside a 5-second window — the fingerprint of a fast port scan — and classifies it as reconnaissance. The `threshold` keyword is what suppresses noise and makes it fire on a *scan* rather than on individual connections.

**Verification:** running `nmap -sS` from Kali against pfSense fired **SID 1000001** immediately — confirming the sensor inspects live traffic and that the detection logic is sound. This rule set later becomes the recon-detection layer of the full attack in Phase 11.

## WireGuard — Encrypted Remote Access

Configured **WireGuard** for remote administration from a mobile device over cellular, with **DuckDNS** dynamic DNS tracking the changing residential IP so the tunnel endpoint stays reachable without a static address.

Two genuine faults surfaced, and the diagnostic method matters more than the fixes:

- **Peer saved but never synced to the live interface.** The WireGuard peer was written to the configuration but never applied to the running kernel interface — so the tunnel looked fully configured yet refused to connect. It was isolated by correlating three sources: `wg show` (live peer/handshake state), `ifconfig` (interface status), and the pfSense filter log over SSH (whether packets were even arriving). The mismatch between saved config and live state pinpointed a failed sync, resolved by forcing the tunnel to re-apply.
- **Interface left on IPv4 type "None."** An interface with its IPv4 type set to "None" comes up with no address and silently routes nothing — a failure with no obvious error message that would have quietly broken tunnel routing. Caught and corrected before it could cause a confusing downstream fault.

## Real-world relevance

This phase combines the three pillars a defensible perimeter rests on: **firewall segmentation** (containing lateral movement), **network intrusion detection with custom signatures** (detection engineering against reconnaissance and exploitation), and **secure remote access** (encrypted administration). The WireGuard debugging — reconciling declared configuration against live runtime state — is a routine but essential operational skill.

*Evidence: see `/screenshots` for the firewall rules, the SID 1000001 alert firing, and the WireGuard handshake.*
