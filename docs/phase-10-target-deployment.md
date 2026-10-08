# Phase 10 — Deploying the Target (DC-1)

## Objective

Place a real, deliberately vulnerable machine on the range, confirm it is reachable and quarantined, and validate the entire detection chain end to end against live traffic for the first time.

## Target selection

Chose **DC-1**, a VulnHub boot2root machine running **Drupal 7 on Debian 7**. It was selected for a realistic web-application entry point tied to a **named CVE**, a clean reconnaissance-to-root path that maps neatly onto MITRE ATT&CK, and DHCP support so it drops onto the segment cleanly.

A note on telemetry: a modern endpoint agent will not install on a 2015 Debian 7 system, so for this run the target is monitored at the **network layer**, which requires nothing installed on it. Host telemetry from such a legacy box could be obtained agentlessly via syslog forwarding — recorded as a future enhancement.

## Concepts applied

- **OVA / OVF.** VulnHub ships DC-1 as a VirtualBox OVA — a tar archive containing an OVF (the VM's hardware description) and a VMDK disk image. Proxmox does not run VMDK natively, so the disk is converted on import.
- **Compatibility hardware for a legacy guest.** A 2015 kernel has drivers for old-style virtual hardware only. Presenting modern hardware (q35, VirtIO disk) kernel-panics it, so the box was given the legacy-safe combination: **i440fx machine type, SeaBIOS, SATA disk, e1000 NIC**.

## What was built

- **Integrity check.** DC-1.zip was downloaded and its **MD5 checksum verified** against the value published by VulnHub — a basic but important habit before running any deliberately vulnerable image.
- **Import.** The disk was imported with `qm importovf` into local storage.
- **Hardware configured** from the command line: an e1000 NIC bridged to the **target segment (`vmbr2`)**, the disk attached on SATA, boot order set, 1 GB RAM, and auto-start on host boot disabled (the target is started deliberately).
- **First boot** succeeded with no kernel panic, confirming the compatibility hardware. The console showed the expected services starting: rpcbind (111), the Apache web server hosting Drupal (80), the MySQL database behind it, and SSH (22). The box settled at a login prompt — no login needed or attempted, as the box is meant to be entered by attacking it.

## Verification — the payoff of the whole build

- **DHCP lease.** pfSense showed DC-1 at **`10.10.20.100`** on the target interface, with a MAC matching the assigned NIC — confirming in one result that the box booted, sits on the correct isolated segment, and that the Phase 8 DHCP server works.
- **First contact and live detection.** The first scan from Kali (`nmap -sS -sV -p- 10.10.20.100`) enumerated the expected open ports (22, 80, 111) and fired detections across the stack:
  - The custom **SID 1000001** ("LOCAL Nmap SYN Scan Detected").
  - An **ET Open** scan signature catching the version-detection probes against Drupal.
  - Both alerts appeared on the **LAN sensor** — exactly as the Phase 9 ingress behaviour predicted, since Kali's scan enters pfSense on LAN even though the target lives on the other segment.
  - The alerts reached the **Wazuh dashboard** through the Phase 7 pipeline with no further configuration — confirming the pipeline generalises automatically to new activity.

## Known limitation — ATT&CK enrichment not automated

Wazuh's ATT&CK dashboard does not auto-classify these alerts. Wazuh only tags an alert with a technique when the firing rule carries MITRE metadata; these Suricata alerts arrive through the custom decoder path (built in Phase 7 to unwrap the nested JSON), which bypasses Wazuh's built-in tagged rules. Automating it would require a MITRE-tagged rule per signature — a substantial effort for a cosmetic dashboard gain. The ATT&CK mapping is therefore done **by hand in the detection report**, which is the more useful skill to demonstrate.

## End state

The lab is complete and proven on a live target: DC-1 is imported, booted, isolated at `10.10.20.100`, reachable from Kali, and watched. A first scan enumerates it, fires both custom and ET Open signatures, and delivers them to the SIEM — validating the detection chain end to end and confirming the separate-segment design in a single test. What remains is to use the range: the full attack and its analysis in Phase 11.

## Real-world relevance

Importing and hardening a legacy VM, verifying image integrity before execution, and standing up an instrumented target against which detections can be validated are all routine in detection engineering and malware/attack-range work. Confirming the full telemetry chain fires on a real target — not just in theory — is the difference between a lab that is *built* and one that is *proven*.

*Evidence: see `/screenshots` for the DC-1 boot, the DHCP lease, and the first scan firing SID 1000001 and the ET Open signature in both Suricata and Wazuh.*
