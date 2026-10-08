# Phase 2 — Network Bridges (Segmentation Foundation)

## Objective

Create the virtual switching layer the lab networks sit on, and establish a clean separation of responsibility between the hypervisor and the firewall.

## Build

Created **`vmbr1`** as the lab's Layer 2 switch. A core design decision underpins this: **Proxmox does not route for the lab — pfSense does.** All routing, NAT, DHCP and DNS are owned by a single authority (the firewall), rather than split between the hypervisor and pfSense. Putting an IP and NAT directly on the bridge would introduce a second, competing router — a well-known source of asymmetric routing and hard-to-diagnose connectivity faults. Keeping the bridge as a plain switch keeps the network model simple and the traffic path predictable, which matters when the whole point of the lab is knowing exactly where traffic flows and where it is inspected.

## Host management addressing

The Proxmox host is given a single management address on `vmbr1` (`10.10.10.2`) so it can reach the Wazuh SIEM on the lab network — with the **gateway field deliberately left blank**. This distinction is important: an address makes the host *reachable* on the segment, while leaving the gateway empty ensures it does **not** become a router or acquire a second default route. The host's own route to the internet remains via `vmbr0`, and pfSense continues to provide all routing, NAT, DHCP and firewalling for the lab.

The resulting exposure is bounded and understood: hosts on the lab segment can reach the Proxmox management address, but every machine on that segment is trusted infrastructure (attacker, DNS filter, SIEM). The deliberately vulnerable target is deployed on a **separate, isolated segment** (Phase 8) with firewall rules that give it no path to the host at all. Tightening host-management access is tracked as a hardening item.

## Real-world relevance

Single-authority routing and deliberate network segmentation are foundational to defensible network design. Reasoning explicitly about which segments can reach management interfaces — and bounding that exposure — is exactly the network-security thinking that separates a flat, fragile lab from a segmented one.

*Evidence: see `/screenshots` for the Proxmox bridge configuration.*
