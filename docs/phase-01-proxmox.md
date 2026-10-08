# Phase 1 — Proxmox Host (Virtualisation Foundation)

## Objective

Stand up the type-1 hypervisor that hosts the entire lab on a single physical machine — a Lenovo ThinkCentre M920s (Intel i5-8500, 24 GB RAM) — and prepare it for local administration.

## Build

Installed **Proxmox VE 9.1** (Debian 13 base) as the virtualisation platform. Every other component in this project — firewall, SIEM, attacker, and target — runs as a guest VM on this host, which keeps the whole lab reproducible and snapshot-friendly on commodity hardware. Added a lightweight **XFCE** desktop for local console access alongside the web UI.

## Troubleshooting — the deb822 repository format

Proxmox 9.1 moved APT to the newer **deb822** source format, and disabling a repository behaves differently from the legacy format. Commenting out the components line — the old habit — produces repeated `malformed entry` errors on every `apt` operation. The correct method is setting `Enabled: false` on the repository stanza. Recording this because the symptom (persistent malformed-entry errors) does not obviously point to the cause, and diagnosing it is exactly the kind of "read the actual format, don't assume" discipline this lab is built to practise.

## Design decision and trade-off

The host runs with **root auto-login** for single-operator convenience. This widens the local attack surface — physical access lands directly on a root session — so it is documented as a deliberate trade-off rather than an oversight. In a production environment this would be replaced with a non-privileged admin account and console lockout; that hardening is tracked as future work.

## Real-world relevance

This mirrors the base layer of any virtualised security environment: a hypervisor hosting segmented workloads. The repo-format troubleshooting reflects real sysadmin work, and explicitly logging the auth trade-off reflects the risk-acceptance documentation expected in a real change process.

*Evidence: see `/screenshots` for the Proxmox host and VM inventory.*
