# Segmented Lab Network with pfSense Firewall

**Author:** Mancho Lovette Feseng
**Program context:** Block 1 (Network Security), Week 4 — Cybersecurity Program v2
**Builds on:** [Week 3 — Segmented Home Lab](./README.md)
**Status:** In progress — network layer complete and verified; firewall rule configuration via the web UI is the next step

## Overview

This project extends the Week 3 lab by introducing **pfSense**, an open-source firewall, as a router/firewall sitting between the attacker (Kali) and target (Metasploitable2) machines. The goal was to move from a flat, unrestricted network to a properly segmented one, where a firewall — not just VirtualBox's network isolation — controls what traffic is allowed to cross between the two sides.

## Architecture

| VM | Role | Interface | Network | IP |
|---|---|---|---|---|
| Kali Linux Purple | Attacker | eth1 | `labnet1` (LAN) | `192.168.58.10` |
| pfSense | Firewall/router | em0 (LAN) | `labnet1` | `192.168.58.1` |
| pfSense | Firewall/router | em1 (WAN) | `labnet2` | `192.168.59.1` |
| Metasploitable2 | Target | eth0 | `labnet2` (WAN) | `192.168.59.20` |

Kali and Metasploitable2 no longer share a network directly — each sits on its own isolated segment (`labnet1` / `labnet2`), with pfSense as the only path between them. Kali plays the trusted "LAN" role; Metasploitable2 plays the exposed "WAN"/DMZ-style role, reflecting its status as the deliberately vulnerable, most-likely-to-be-compromised machine.

## What was tested and confirmed

**One-hop connectivity (Kali → pfSense LAN):**
```
ping -c 4 192.168.58.1
```
Result: 4/4 received, 0% loss — confirms Kali correctly reaches its gateway.

**LAN → WAN routing (Kali → Metasploitable2, through pfSense):**
```
ping -c 4 192.168.59.20
```
Result: 4/4 received, 0% loss, TTL=63 (down from the usual 64) — the TTL drop is direct evidence the packet was actually forwarded through one router (pfSense), not sent over some unintended direct path.

**WAN → LAN blocking (Metasploitable2 → Kali, through pfSense):**
```
ping -c 4 192.168.58.10
```
Result: 4/4 sent, 0 received, 100% loss — with **no error message at all**, unlike earlier misconfiguration failures which produced explicit "unreachable" errors. This silence is the signature of a firewall *dropping* traffic by policy, rather than a routing failure.

**Conclusion:** pfSense's default rule set allows LAN-initiated traffic out to WAN (and its replies back), while silently blocking WAN-initiated traffic into LAN — the same asymmetric-trust model used at the edge of real corporate networks (e.g., a DMZ web server can be reached from outside, but can't freely reach back into the internal network).

## Build notes and troubleshooting

- **pfSense's current official download requires an internet connection mid-install** (the "Netgate Installer" fetches packages live), which conflicts with an isolated lab VM that has no internet leg. Worked around by using the last **fully self-contained release, pfSense CE 2.7.2**, mirrored by Dakota State University's IA lab — functionally identical for lab purposes.
- **VirtualBox VM boot-media loops:** both the Metasploitable and pfSense installs looped back into their installers after finishing, because the installation ISO was still attached and taking boot priority. Fixed each time by powering off the VM and detaching the ISO from Storage settings before the next boot.
- **Internal Network name mismatches broke connectivity silently.** After wiring Kali and Metasploitable to new segment names (`labnet1`/`labnet2`), pfSense's own adapters were still attached to the old `labnet` name from Week 3 — an oversight that wasn't visible in the GUI at a glance. Diagnosed using `VBoxManage showvminfo <vm> | findstr "NIC"` on the host to inspect the exact configured network name per adapter, and fixed with `VBoxManage modifyvm <vm> --intnet1 <name>` / `--intnet2 <name>`.
- **A VM name with a trailing space** (`"pfsense "` instead of `"pfsense"`) required matching the exact string, including the space, in every subsequent `VBoxManage` command.
- **Non-persistent network config on Metasploitable.** IP address and gateway set via `ifconfig`/`route add` do not survive a reboot on this older Ubuntu build — after a restart, the interface came back up with no configuration at all, reproducing a "Network is unreachable" error until the commands were re-run.
- **TTL as a verification tool:** rather than trusting a successful ping alone, comparing TTL values (64 for a direct one-hop reply vs. 63 for a reply that passed through one router) confirmed the packet actually traversed pfSense as expected — a technique also used by tools like `traceroute`.

## Key concepts reinforced

- **Segmentation:** a firewall can only enforce policy on traffic that is forced to physically pass through it; two hosts on the same network segment can always bypass any firewall sitting outside that path.
- **DMZ pattern:** exposing a vulnerable/public-facing host on its own segment (here, `labnet2`/WAN) limits the blast radius if that host is compromised, since the firewall still separates it from the trusted internal segment.
- **Stateful vs. asymmetric default behavior:** a stateful firewall permits return traffic for connections the trusted side initiated, while blocking new connections initiated from the untrusted side by default.

## Tools used

- VirtualBox
- pfSense CE 2.7.2
- Kali Linux Purple, Metasploitable2 (from Week 3)

## Next steps

- Log into pfSense's web interface (`https://192.168.58.1`) and write explicit firewall rules (e.g., allow only SSH from Kali to Metasploitable, block everything else)
- Document the difference between default behavior (this writeup) and custom rule behavior (next writeup)
- Carry this segmented topology forward into Week 5 (IDS/IPS with Suricata) and Week 9 (VLAN-based segmentation)
