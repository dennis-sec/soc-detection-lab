# Detection Coverage Assessment — DC-1 Attack Chain

> Controlled detection exercise. The attack was carried out by me against my own deliberately vulnerable target on an isolated lab network, with the goal of measuring what the monitoring stack detects and misses. This report is written from the defender's point of view, using only the data the defensive sensors produced.

## Attack Path Summary

| Stage | Technique | ATT&CK ID | Detected? |
|---|---|---|---|
| Reconnaissance — port/service scan | Active Scanning / Network Service Discovery | T1595 / T1046 | Yes (Suricata) |
| Web enumeration — CMS fingerprint, directory brute-force | Active Scanning: Wordlist Scanning | T1595.003 | Partial |
| Initial access — Drupalgeddon2 RCE | Exploit Public-Facing Application | T1190 | Yes (CVE named) |
| Execution — commands via web shell | Command and Scripting Interpreter | T1059 | No (host-resident) |
| Credential access — DB creds in `settings.php` | Unsecured Credentials | T1552 | No (host-resident) |
| Privilege escalation — SUID `find` to root | Abuse Elevation Control: Setuid | T1548.001 | No (host-resident) |
| Attempted egress — outbound DNS from owned host | Application Layer Protocol | T1071 | Blocked + logged |

## Executive Summary

On 29 July 2026, network monitoring detected a multi-stage attack against a lab web server at `10.10.20.100` (DC-1), originating from `10.10.10.100`. Using only network telemetry — Suricata IDS alerts and pfSense firewall logs, aggregated in Wazuh — the defence detected the attacker's reconnaissance and the exploitation of the web application, and contained the compromised host's outbound communication attempts. However, monitoring was blind to all host-level activity after compromise: command execution, credential access, and privilege escalation produced no network telemetry and were not observed. The attack resulted in full root compromise of the target — detected at the perimeter, invisible on the host.

The central finding is a **visibility gap**: the architecture sees attacks crossing the network but not attacker actions inside a host.

## Environment and Detection Sources

The target was DC-1, an Apache/Drupal 7 web server at `10.10.20.100` on an isolated segment. The attacker operated from `10.10.10.100`.

Detection sources available to the defender were Suricata IDS in alert-only mode on two interfaces (LAN and VULNHUB), pfSense firewall logs, and the Wazuh SIEM aggregating both via syslog. Critically, no host-based telemetry was available — there was no endpoint agent on the target. This limitation shapes the entire report.

## Timeline of Detected Activity

Reconstructed solely from defensive telemetry, in the order a defender would have seen it:

- **14:49:01 — Reconnaissance (port scan).** A TCP SYN scan from `10.10.10.100` fired the custom rule SID `1000001` ("LOCAL Nmap SYN Scan Detected"), seen on the LAN sensor.
- **~14:49:15 — Reconnaissance (web probing).** Multiple SID `2024364` alerts ("ET SCAN Possible Nmap User-Agent Observed") against port 80 indicated the web service was being enumerated.
- **15:27:06–15:27:12 — Exploitation.** SID `2025534` ("ET WEB_SPECIFIC_APPS Drupalgeddon2 … RCE, CVE-2018-7600"), classified as *Attempted Administrator Privilege Gain*, indicated an attempted remote code execution against Drupal.
- **Silent period — no network alerts.** The blind spot (see below).
- **After ~15:27 — Contained outbound attempts.** The host at `10.10.20.100` repeatedly attempted outbound DNS to `1.1.1.1:53` and `9.9.9.9:53`; every attempt was blocked by the firewall on the VULNHUB interface and logged.

## What the Defence Detected

**Reconnaissance** was caught cleanly. The initial port scan tripped a custom rule immediately, and follow-on web probing generated scanner-signature alerts. A defender would correctly conclude the target was under active enumeration. *(MITRE ATT&CK: T1595 Active Scanning; T1046 Network Service Discovery.)*

**Exploitation** was the highest-value detection. The Drupalgeddon2 attempt fired a signature that named the exact CVE (2018-7600) and classed it as an attempted administrator-privilege gain. The defender would know not merely that an attack occurred, but precisely which exploit was used against which application — a high-confidence alert. *(MITRE ATT&CK: T1190 Exploit Public-Facing Application.)*

**Containment** worked and doubled as a detection signal. After compromise, the host tried repeatedly to reach external DNS resolvers; every attempt was blocked, preventing any internet reach (no exfiltration, no command-and-control). The blocked entries are themselves an indicator of compromise — a healthy internal host does not normally have its outbound traffic dropped, so repeated denials from one host warrant investigation. *(MITRE ATT&CK: consistent with T1071 Application Layer Protocol, denied by policy.)*

## The Critical Gap — What the Defence Missed

Between the exploitation alert at 15:27 and the blocked-egress logs, the network sensors recorded nothing — yet this was the window of the most severe activity. From the target's nature and the outcome, the defender can infer (but did not observe) that during this silence the attacker executed commands on the host, accessed stored credentials, and escalated privileges to root.

None of this crossed the network, so Suricata could not see it. Reading files, running local binaries, and abusing a SUID program to gain root are entirely host-resident actions, and a network IDS is architecturally incapable of detecting them. This is not a missed signature or a tuning failure — it is a fundamental blind spot of network-only monitoring.

The consequence is stark: the defence saw the attacker arrive and saw the compromised host try to leave, but was blind to the attacker taking full control in between. In a real incident the defender would know a break-in was attempted and the host was behaving abnormally, but would have no record of the privilege escalation or the credential theft. *(MITRE techniques that occurred but were NOT detected: T1059 Command and Scripting Interpreter; T1552 Unsecured Credentials; T1548.001 Abuse Elevation Control Mechanism: Setuid.)*

## Detection Coverage Summary

- **Reconnaissance** (T1595 / T1046) — Detected, Suricata LAN.
- **Web enumeration** (T1595.003) — Partially detected; scanner user-agent seen, but directory brute-forcing was largely silent.
- **Exploitation / RCE** (T1190) — Detected, with the CVE identified. Suricata LAN.
- **Command execution** (T1059) — Not detected (host-resident).
- **Credential access** (T1552) — Not detected (host-resident).
- **Privilege escalation** (T1548.001) — Not detected (host-resident).
- **Attempted egress / C2** (T1071) — Blocked and logged. pfSense VULNHUB.

The pattern is consistent: strong at the perimeter, blind on the host.

## Recommendations

**Deploy host-based telemetry on protected assets.** This is the highest-value improvement. A host agent — or, for legacy systems that cannot run one, agentless forwarding of authentication and system logs to the SIEM — would have surfaced the command execution, credential access, and privilege escalation that the network missed entirely. This directly closes the gap above.

**Improve web reconnaissance detection.** Directory brute-forcing produced almost no alerts; rate-based HTTP alerting or tuned web-scan signatures would raise the cost of quiet enumeration.

**Alert on blocked egress as an indicator of compromise.** The denied DNS attempts were logged but deserve an explicit SIEM rule — repeated blocked outbound from an internal host is a strong compromise signal and should notify an analyst, not merely sit in a log.

**Evaluate inline prevention (IPS).** The exploit was detected but not blocked, because Suricata ran in alert-only mode. Enabling inline blocking for high-confidence signatures such as the Drupalgeddon2 rule would move the stack from detection toward prevention.

**Enrich alerts with ATT&CK mapping** in the SIEM so techniques classify automatically and speed triage.

## Conclusion

The defensive stack performed well at the network perimeter: it detected the reconnaissance, identified the exploitation by CVE, and contained the host's outbound communication. Its decisive weakness is the absence of host visibility — the escalation to full root control occurred with no network-observable telemetry. The perimeter caught the attacker coming and going, but not what they did inside. Adding host-based monitoring is the primary recommendation and would turn this from a partially-observed intrusion into a fully-observed one.
