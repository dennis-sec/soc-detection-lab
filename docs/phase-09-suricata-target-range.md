# Phase 9 — Suricata on the Target Range (Second Sensor)

## Objective

A sealed, routed network still is not a *watched* one. This phase places a second Suricata sensor on the target segment so that traffic crossing it is inspected to the same standard as the operator network.

## The core concept — Suricata is per-interface

The single most important idea here: in pfSense, Suricata does **not** watch "the network" as a whole. Each instance is bound to exactly one interface and sees only the traffic crossing that specific interface. The existing LAN instance inspects LAN traffic and nothing else — it does not, and cannot, automatically extend to the new target interface. A monitored interface must have its **own** instance, created deliberately.

Three consequences follow, and each shaped the build:
- **Home Net is per-instance.** It resolves automatically to the interface's own subnet (`10.10.20.0/24` here), so the custom rules — written to fire on traffic heading *into* Home Net — fire on attacks aimed at the target. It was left at default so this applies automatically.
- **Rules are per-instance.** A new instance starts with an empty custom-rules box; rules do not propagate. The six custom rules had to be copied across by hand — the step that is easy to forget and would silently leave the target undetected.
- **Each instance writes its own `eve.json`.** The Phase 7 `syslog-ng` file source uses a wildcard pattern, so it picks up the new instance's log automatically with no pipeline change.

## What was built

- **Resource adjustment first.** Each instance loads the full ruleset into memory independently, roughly doubling Suricata's footprint, so the pfSense VM was raised from 2 GB to **4 GB** before enabling the second instance — avoiding an out-of-memory failure mid-startup.
- **New instance** bound to the target interface, in **alert-only** mode (detect and record, do not drop) to match the LAN sensor and preserve a clean observation of the attack in Phase 11.
- **Logging configured around `eve.json`**, the primary feed `syslog-ng` reads. EVE JSON logging was enabled to file and trimmed to **alerts only** (the high-volume traffic logs left off), mirroring the LAN sensor — this does not weaken detection, as every packet is still inspected.
- **Rules loaded in two places:** the **ET Open** attack categories (scan, exploit, shellcode, web-server, web-specific-apps, malware and related) enabled on the Categories tab, then the **six custom rules** pasted into the Custom rules box.

## Verification — why it looked like a failure but was not

Scanning the target's gateway from Kali fired the **LAN** sensor, and the new target sensor stayed silent. On the surface this looked broken; understanding why is the point of the phase. The sensor that inspects a packet is the one bound to the interface the packet **enters** through. Kali lives on LAN, so its scan enters on LAN and the LAN sensor sees it. The target sensor saw nothing because **the target segment was still empty** — no machine, no traffic to inspect. A sensor watching an empty network correctly reports nothing.

This is the same ingress principle from Phase 7, seen from the other side, and it sets the correct expectation for Phase 11: **the attacker's inbound traffic (scans, exploits) is caught on the LAN sensor; the target's outbound traffic (responses, reverse shell) is caught on the target sensor.** The two sensors catch opposite halves of the engagement. The real firing test had to wait until a live target existed.

## Real-world relevance

Understanding exactly where a sensor sees traffic — and where it is blind — is fundamental to sensor placement in real network monitoring. The per-interface model, and the ingress-direction logic that decides which sensor fires, are precisely the considerations behind TAP/SPAN placement and traffic-mirroring design in production detection.

*Evidence: see `/screenshots` for both Suricata instances running and the custom rules loaded on the target sensor.*
