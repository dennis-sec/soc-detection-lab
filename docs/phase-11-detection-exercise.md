# Phase 11 — The Detection Exercise (Full Attack, Analysed From the Defender's Side)

## Objective

The capstone of the lab: carry out a complete attack against DC-1, start to finish, while the defensive stack watches, then reconstruct that attack **using only the sensors' output** to measure exactly what the monitoring catches and what it misses. The attack is a means to an end — the deliverable is the detection analysis.

Suricata was kept in **IDS (alert-only) mode** on purpose. The goal is to observe what the defences *see*; a sensor that silently dropped traffic mid-attack would prevent that observation. Both instances were confirmed in alert-only mode before starting.

## Two points of view, kept separate

The attack produces evidence for two different perspectives, and keeping them apart is what makes the analysis honest:

- The **attacker's** record is what was done — the commands, the exploit, the path to root.
- The **defender's** record is the reconstruction built only from Suricata, the firewall, and the SIEM. A real defender has no access to the attacker's notes; they see only what their sensors captured and must infer the rest.

This phase is the **defender's** analysis. It was built strictly from defensive telemetry — the attacker's command log was deliberately not used as a source — because the value of a detection report depends on it being limited to what a defender would genuinely have.

## How the attack was carried out

The target was attacked over the network from Kali (`10.10.10.100`) with **no console access and no credentials** — access to the machine came entirely from exploitation. The kill chain:

1. **Reconnaissance** — a full port and service scan identified SSH (22), Apache/Drupal 7 (80), and rpcbind (111).
2. **Web enumeration** — fingerprinted the CMS as Drupal 7 and mapped its structure.
3. **Initial access** — exploited **Drupalgeddon2 (CVE-2018-7600)** for a shell as the web-server user, `www-data`.
4. **Post-exploitation** — recovered database credentials stored in Drupal's `settings.php` and located flag files.
5. **Privilege escalation** — abused a **SUID root `find` binary** (which runs with its owner's privileges and can execute commands) to spawn a shell that retained root, confirmed by an effective UID of 0 — full control of the host.
6. **Pivot test** — from the root shell, attempted outbound connectivity to confirm the isolation held under worst-case conditions.

## What the defence detected

Reconstructed from the sensors, in order:

**Reconnaissance — detected.** The initial SYN scan fired the custom rule **SID 1000001** on the LAN sensor, and follow-on web probing generated repeated scanner-signature alerts (**SID 2024364**). A defender would correctly conclude the target was under active enumeration. *(ATT&CK: T1595 Active Scanning; T1046 Network Service Discovery.)*

**Exploitation — detected, and precisely identified.** The highest-value alert of the run: the Drupalgeddon2 attempt fired **SID 2025534** ("ET WEB_SPECIFIC_APPS Drupalgeddon2 … CVE-2018-7600"), classified as *Attempted Administrator Privilege Gain*. The alert named the **exact CVE against the exact application** — a defender would know not merely that an attack occurred, but precisely which exploit was in use. *(ATT&CK: T1190 Exploit Public-Facing Application.)*

**Containment — held, and doubled as a detection signal.** After compromise, the owned host repeatedly attempted outbound DNS to external resolvers (`1.1.1.1`, `9.9.9.9`); every attempt was **blocked by the firewall and logged**. This prevented any internet reach — no exfiltration, no command-and-control — **even with the attacker holding root**. The isolation designed in Phase 8 held under the worst case. The blocked entries are themselves an indicator of compromise: a healthy internal host does not normally have its outbound traffic dropped. *(ATT&CK: consistent with T1071, denied by policy.)*

## The critical gap — what the defence could not see

Between the exploitation alert and the blocked-egress logs there is a **silent window** — and that is precisely when the most damaging activity happened. During it, the attacker executed commands on the host, read stored credentials, and escalated to root. A defender relying on network telemetry would have observed **none** of it.

The reason is fundamental, not a tuning failure. Reading files, running local binaries, and abusing a SUID program to gain root are **entirely host-resident** actions — they never cross the network, so a network IDS is architecturally incapable of seeing them. The consequence, stated plainly: the defence saw the attacker **arrive** and saw the compromised host **try to leave**, but was blind to the attacker **taking full control in between**. In a real incident, the defender would know a break-in was attempted and the host was behaving abnormally, but would hold no record of the privilege escalation or the credential theft. *(ATT&CK techniques that occurred but were NOT detected: T1059 Command and Scripting Interpreter; T1552 Unsecured Credentials; T1548.001 Abuse Elevation Control: Setuid.)*

## Detection coverage, stage by stage

- **Reconnaissance** (T1595 / T1046) — Detected, Suricata LAN.
- **Web enumeration** (T1595.003) — Partial; scanner user-agent flagged, but directory brute-forcing largely silent.
- **Exploitation / RCE** (T1190) — Detected, with the CVE identified. Suricata LAN.
- **Command execution** (T1059) — Not detected (host-resident).
- **Credential access** (T1552) — Not detected (host-resident).
- **Privilege escalation** (T1548.001) — Not detected (host-resident).
- **Attempted egress / C2** (T1071) — Blocked and logged. pfSense.

The pattern is consistent: **strong at the perimeter, blind on the host.**

## Sensor placement, confirmed live

This run confirmed the ingress principle from Phase 9 against real attack data. The attacker's traffic — the scan and the exploit — entered pfSense on the **LAN** interface (where Kali sits), so those detections appeared on the LAN sensor rather than the target sensor, even though the target lives on the other segment. The target sensor's role is the **return** direction — the host's own outbound traffic. Reasoning, now validated by observation.

## Key recommendations

- **Deploy host-based telemetry on protected assets** — the single highest-value improvement. A host agent, or agentless syslog forwarding for legacy systems that cannot run one, would have surfaced the command execution, credential access, and privilege escalation the network missed entirely. This directly closes the gap that defined the run.
- **Improve web reconnaissance detection** — directory brute-forcing was nearly silent; rate-based HTTP alerting would raise the cost of quiet enumeration.
- **Alert on blocked egress as an indicator of compromise** — repeated denied outbound from an internal host is a strong signal and should notify an analyst, not merely sit in a log.
- **Evaluate inline prevention (IPS)** for high-confidence signatures like the Drupalgeddon2 rule, which was detected but not blocked in this alert-only run.

## End state and conclusion

A full attack was carried out and analysed purely from the defender's point of view. The stack detected the reconnaissance, identified the exploitation by CVE, and contained the host's outbound communication — but was blind to all on-host activity between break-in and containment. The headline conclusion is the kind of finding the entire lab was built to produce: **the architecture observes attacks crossing the network but not attacker actions inside a host**, and host-based monitoring is the primary improvement that would turn a partially-observed intrusion into a fully-observed one.

## Full report

The complete detection analysis — the attack-path summary, the full timeline, the stage-by-stage ATT&CK mapping, and the detailed recommendations — is in the dedicated report:

**➡️ [Blue Team Detection Report](../reports/blue-team-detection-report.md)**

## Real-world relevance

This is the core work of a SOC analyst: reconstructing an intrusion from telemetry alone, mapping it to MITRE ATT&CK, and articulating the coverage gaps and the improvements that would close them. The central finding — network monitoring is blind to host-resident post-exploitation — is one of the most important and commonly-cited limitations in real-world detection, and demonstrating it with evidence from a live attack is exactly what the lab set out to prove.

*Evidence: see `/screenshots` for SID 1000001 (recon), SID 2025534 naming the CVE (exploitation), and the blocked outbound DNS from the compromised host (containment).*
