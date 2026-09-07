# Segmented Home Lab — Isolated Attacker/Target Network

**Author:** Mancho Lovette Feseng
**Program context:** Block 1 (Network Security), Week 3 — Cybersecurity Program v2
**Status:** In progress — this is the base network, extended in later weeks with a firewall (pfSense), IDS (Suricata), and segmentation (VLANs/Zero Trust)

## Overview

This project is the foundation lab for my self-directed cybersecurity program. The goal was to build a genuinely isolated attacker/target environment — one where a deliberately vulnerable machine can be attacked freely without any risk of exploit traffic, malware, or scanning reaching my real home network or the internet.

The lab consists of two VMs on a private, host-only virtual network with no route out:

| VM | Role | IP | Internet access |
|---|---|---|---|
| Kali Linux Purple | Attacker box | `192.168.58.10` | Yes (separate NAT adapter, for tool updates) |
| Metasploitable2 | Vulnerable target | `192.168.58.20` | No — isolated on purpose |

## Architecture

Kali is **dual-homed** — it has two virtual network adapters:
- **Adapter 1 (NAT):** gives Kali internet access for downloading tools and updates
- **Adapter 2 (Internal Network, `labnet`):** connects Kali to the isolated lab segment

Metasploitable2 has **only one adapter**, attached to the same Internal Network (`labnet`), with no NAT or bridged adapter at all. VirtualBox's Internal Network mode creates a virtual switch that exists purely inside the hypervisor — no host involvement, no external routing — so there is no possible path from that segment to the outside world unless one is deliberately configured.

## Why Internal Network instead of NAT or Bridged

- **NAT** routes through the host machine — convenient for internet access, but not appropriate for a machine you intend to attack, since traffic still transits host-adjacent infrastructure.
- **Bridged** puts the VM directly on the home LAN with its own IP — the opposite of what a lab needs; it would expose the vulnerable machine to every other device on the network.
- **Internal Network** has no host or internet path at all. Even if every vulnerability on Metasploitable2 were triggered simultaneously, there's no route for anything to travel outward on — the isolation is structural, not just a firewall rule that could be misconfigured.

## Verification

Two tests confirmed the lab works as intended:

1. **Connectivity test** — from Kali: `ping -c 4 192.168.58.20` → 4/4 packets received, 0% loss. Confirms both VMs are correctly reachable on the same internal segment.
2. **Isolation test** — from Metasploitable2: `ping -c 4 8.8.8.8` → 100% loss / network unreachable. Confirms the target VM has no path to the internet whatsoever.

## Build notes and troubleshooting

Real debugging encountered during the build, kept here because it's often more useful to a reader than a clean happy-path tutorial:

- **VirtualBox "Saved State" locks hardware changes.** A VM that's merely closed (not explicitly powered off) enters a saved state, which silently blocks network adapter changes in both the GUI and `VBoxManage`. Fixed with `VBoxManage discardstate <vm>` before making adapter changes.
- **`VBoxManage` not recognized in Windows cmd.** Not on PATH by default. Worked around by calling the full path (`"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"`) or adding the install folder to the system PATH permanently.
- **`dhclient` missing on newer Kali builds.** Kali has moved to NetworkManager; used `nmcli device connect <iface>` instead.
- **Static IP via nmcli requires exact syntax.** `ipv4.method manual ipv4.addresses <ip>/<prefix>` — a typo'd `ifname` (as `iframe`) or `ipv4.addresses` (as `ipv4.adesses`) fails silently with a generic "invalid setting" error, so exact spelling matters.
- **VirtualBox's "New VM" wizard optical-disk step is not the hard-disk step.** When attaching a pre-built `.vmdk` (like Metasploitable2's), skip the optical disk prompt entirely (it filters for `.iso` only) and attach the existing disk afterward via **Settings → Storage → Controller: SATA → Add Existing Disk**.
- **Windows hides file extensions by default**, which made a valid `.vmdk` file look like it had "no extension" — worth turning on "File name extensions" in File Explorer's View options to avoid this confusion.

## Tools used

- VirtualBox
- Kali Linux Purple
- Metasploitable2 (intentionally vulnerable Linux target — [official download](https://sourceforge.net/projects/metasploitable/files/Metasploitable2/metasploitable-linux-2.0.0.zip/download))

## Next steps

- Deploy pfSense as the lab's edge firewall (Week 4)
- Deploy Suricata IDS and write a custom detection rule (Week 5)
- Add VLAN-based segmentation (Week 9)
- Roll this into a full "build and defend a segmented network" capstone writeup (Week 11)
