# Cybersecurity Lab Notes

Hands-on writeups from my self-directed cybersecurity study, built in a home lab on VirtualBox. Every week I build something, break it, find out why, and write down the evidence.

## The lab

| Machine | Role | Address |
|---|---|---|
| Kali Linux | Attacker / client | 192.168.58.10 (tunnel 10.10.10.2) |
| pfSense CE 2.8.1 | Firewall, router, IDS, VPN server | LAN 192.168.58.1, WAN 192.168.59.1 |
| Metasploitable 2 | Deliberately vulnerable target | 192.168.59.20 |

Kali (LAN) → pfSense → Metasploitable (WAN side). Everything is isolated inside VirtualBox.

## Writeups

| Week | Topic | What I did |
|---|---|---|
| 4 | [pfSense firewall](week4-pfsense-firewall.md) and [rules](week4-firewall-rules.md) | Set up the firewall, NAT and allow/deny rules, and tested them. |
| 5 | [Suricata IDS](week5-suricata-ids.pdf) | Installed an intrusion detection system, chased an Nmap "filtered" mystery to a wrong route on Kali, got real alerts, and wrote my own detection rule for the vsftpd 2.3.4 backdoor. |
| 6 | [WireGuard VPN](week6-wireguard-vpn.pdf) | Built a VPN between Kali and pfSense and used Wireshark to show the wire carries only encrypted data. |

## Skills shown

- Firewall rules, NAT and the state table (pfSense)
- Network scanning and reading results (Nmap: open, closed, filtered, `--reason`)
- Intrusion detection and writing custom rules (Suricata)
- Troubleshooting by evidence: `tcpdump`, `ip route get`, Wireshark
- VPN setup and verification (WireGuard)

## How I work

I don't trust a success message by itself. When something fails, I test the start, middle and end of the path and keep the screenshots. Several of these writeups include a "what I got wrong" section, because that's where I learned the most.

## What's next

- Network security wrap-up: Wireshark packet analysis, VLAN segmentation
- Then six months on **cloud security** with an ethical-hacking mindset: identity and access management (IAM), AWS security, and DevSecOps

## Safety note

All attacks and scans run only against my own lab machines. No real systems are touched. No private keys or passwords are published in this repo.
