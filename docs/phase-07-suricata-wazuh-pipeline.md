# Phase 7 — Feeding Suricata & pfSense into Wazuh (Detection Pipeline)

## Objective

Give the SIEM sight. At the end of Phase 5 Wazuh was running but blind — no data sources. This phase builds the log pipeline so that firewall events and IDS detections both surface as alerts on a single dashboard, turning three isolated tools into one correlated detection platform.

## Design decision — agentless integration

**No Wazuh agent was installed on pfSense.** pfSense is a hardened FreeBSD appliance where agent installation is unsupported and discouraged, so the integration is **agentless**: pfSense forwards its logs over the network to a syslog receiver on the Wazuh manager. A side benefit is that the same receiver can later ingest logs from other network devices.

## What was built

On the **Wazuh manager**, a syslog receiver was added (a `<remote>` block listening on UDP 514, restricted by `allowed-ips` to accept traffic only from pfSense).

On **pfSense**, the `syslog-ng` package was installed as a relay handling two inputs:
1. A **network source** receiving pfSense's own system and firewall logs.
2. A **file source** reading Suricata's `eve.json` directly from disk, with `program-override("suricata")` to tag it.

Both feed a single destination pointing at the Wazuh manager, with the source address forced to match the manager's allow-list.

On the **Wazuh side**, two custom pieces were written:
- A **custom decoder** that uses the `JSON_Decoder` plugin to unwrap Suricata's JSON from inside the syslog envelope pfSense wraps it in.
- A **custom rule (ID 100101, level 8)** that raises an alert only when that decoder has run and the event type is `alert` — so genuine IDS detections raise alerts while routine traffic stays silent.

Finally, **Service Watchdog** was installed on pfSense to monitor both `syslogd` and `syslog-ng`, and Suricata's EVE output was tuned down to alerts only.

## Troubleshooting — six distinct faults, and how each was diagnosed

This phase was mostly debugging, and the diagnostic method matters more than each fix:

1. **A scan that produced no alert.** An nmap scan from Kali against the Wazuh VM generated nothing. The cause was architectural, not a misconfiguration: two hosts on the same virtual bridge are switched directly to each other, so the traffic never crosses pfSense and Suricata never sees it. Re-testing against pfSense itself fired the rule instantly. **This finding directly drove the Phase 8 decision to place the target on its own routed segment** — co-locating attacker and target would have left the IDS blind to the whole exercise.

2. **Silent syslog failure.** pfSense's remote logging looked correctly configured, yet Wazuh received nothing. The chain was split in two to isolate it: `ss -lnup | grep 514` on Wazuh confirmed the receiver was listening, while `tcpdump` on port 514 captured zero inbound packets — receiver healthy, sender dead. The cause is a documented pfSense issue where the log daemon stops and never retries. Re-saving the settings restarted it; Service Watchdog was added so it self-heals.

3. **The missing hostname.** Packets now arrived, but pfSense events never became dashboard alerts. `wazuh-logtest` on a raw event showed the hostname field had been consumed into the wrong slot — pfSense's native syslog omits the hostname from its header, so the decoder could never match. Installing `syslog-ng` fixed it by applying strict RFC3164 formatting with the hostname inserted; the event then decoded correctly and fired a rule.

4. **A case-sensitivity failure that took down all logging.** After adding the Suricata source, `syslog-ng` refused to start entirely — killing the firewall logging that had been working. Reading the generated config revealed the pfSense package lowercases object names, so a source was written as `suricata` while a log object still referenced `Suricata`. `syslog-ng` is case-sensitive and refuses to start on any invalid reference — one bad reference takes down the whole service. The lesson: check the generated config file, not just the web form.

5. **A decoder pattern mismatch.** Wazuh's built-in Suricata rules expect raw `eve.json`, but pfSense nests that JSON inside a syslog header. The first custom decoder expected a process ID (`suricata[\d+]:`) that the arriving format didn't contain. Matching on `program_name` instead worked immediately, and `wazuh-logtest` confirmed every field extracted: signature, signature ID, source and destination IPs, ports, protocol.

6. **A duplicate rule ID.** Decoding succeeded but no rule fired. Inspecting the rules file showed two rules sharing one ID — both config commands had used append mode rather than replace. Wazuh keeps the first definition and discards the second, and the survivor depended on a parent rule that never fires for custom-decoded logs. Rewriting it as a single self-contained rule using `decoded_as` and a field match resolved it; rule 100101 then fired correctly.

## An unexpected result worth keeping

During the session, Wazuh fired an alert when pfSense's own log daemon died — the SIEM detected the failure of its own telemetry pipeline. This is genuinely valuable, because **a silently dead log source looks identical to a quiet network**. Combined with Service Watchdog auto-restarting the daemon, the pipeline now both self-heals and reports when it fails.

## Known limitations

- Timestamps run in UTC while local time is BST; pfSense and Wazuh agree with each other, so correlation is accurate — only wall-clock comparison needs the one-hour offset applied. Running infrastructure in UTC is standard practice, so this is documented rather than changed.
- Syslog runs over UDP — fire-and-forget with no delivery confirmation — which is exactly why the silent-sender fault (number 2) was hard to see without packet capture.
- Suricata's EVE output was reduced to alerts only. Full traffic logging (DNS, TLS, HTTP, flows) produced enormous volume that buried real detections. This does not weaken detection — every packet is still inspected and every rule still evaluates — it only gives up bulk traffic forensics, partly covered by Pi-hole's DNS logs.

## End state

The pipeline runs end to end: an attack crosses the network, pfSense logs it and Suricata inspects it, `syslog-ng` normalises and forwards both streams, and Wazuh decodes them onto one dashboard — all persistent across reboots. Every future detection flows through automatically, not just the nmap test.

## Real-world relevance

Log aggregation and normalisation into a SIEM is core SOC engineering, and the realistic part is that it rarely works first try — most of this phase was methodical fault isolation across a multi-stage pipeline (`ss`, `tcpdump`, `wazuh-logtest`, reading generated configs). The self-monitoring outcome — alerting on the death of a log source — reflects a mature detection mindset: the absence of data is itself a signal.

*Evidence: see `/screenshots` for the decoded Suricata alert in Wazuh and the log-source-failure alert.*
