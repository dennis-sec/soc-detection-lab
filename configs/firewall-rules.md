# pfSense Firewall Rules — SOC Detection Lab

Firewall / router: pfSense CE 2.8.1. Rules sanitised from the live configuration — internal RFC1918 addresses only.

| Interface | Device | Address | Role |
|-----------|--------|---------|------|
| WAN | `vtnet0` | DHCP on `192.168.1.0/24` | Uplink to home router |
| LAN | `vtnet1` | `10.10.10.1/24` | Operator network (Kali, Wazuh, Pi-hole) |
| OPT1 / WG_VPN | `tun_wg0` | `10.6.210.1/24` | WireGuard remote access |
| OPT2 / VULNHUB | `vtnet2` | `10.10.20.1/24` | Isolated target range (no internet) |

## WAN
| # | Action | Proto | Source | Destination | Purpose |
|---|--------|-------|--------|-------------|---------|
| 1 | Pass | UDP | any | WAN address : 51820 | WireGuard tunnel inbound |

All other inbound WAN traffic hits the implicit default-deny. The web UI is not exposed on WAN.

## LAN *(evaluated top-down)*
| # | Action | Proto | Source | Destination | Purpose |
|---|--------|-------|--------|-------------|---------|
| 1 | **Block** | any | LAN net | `192.168.1.0/24` | Block lab → home (lateral-movement prevention) |
| 2 | Pass | IPv4 | LAN net | any | Default allow LAN → any |
| 3 | Pass | IPv6 | LAN net | any | Default allow LAN → any (v6) |

Rule 1 sits above the allow-all, so lab hosts cannot reach the home subnet while still reaching the internet via WAN NAT.

## OPT1 — WireGuard (WG_VPN)
| # | Action | Proto | Source | Destination | Purpose |
|---|--------|-------|--------|-------------|---------|
| 1 | Pass | IPv4 | OPT1 net | any | Allow VPN clients to reach the lab |

## OPT2 — VULNHUB (isolated target range) *(the isolation design — order matters)*
| # | Action | Proto | Source | Destination | Purpose |
|---|--------|-------|--------|-------------|---------|
| 1 | Pass | TCP | VULNHUB net | `10.10.10.101` : 1514–1515 | Target → Wazuh manager (telemetry) |
| 2 | Pass | any | VULNHUB net | `10.10.10.100` | Target → Kali (reverse shells / callbacks) |
| 3 | **Block (logged)** | any | VULNHUB net | any | Isolation wall — no internet, home, or other lab hosts |

Rule 3 is last and logged. Because pfSense evaluates firewall rules before NAT, this block enforces "no internet" even though outbound NAT exists, and every dropped escape attempt is logged into the Wazuh pipeline. The result is a segment sealed to exactly two destinations: the SIEM and the attacker.

## Outbound NAT
Mode: Automatic. NAT entries exist for the internal subnets, but egress from the VULNHUB segment is still denied by the Rule 3 block above (firewall is evaluated before NAT).
