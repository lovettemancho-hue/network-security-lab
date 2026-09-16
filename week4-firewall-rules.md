# pfSense Firewall Rules — Default Behavior vs. Custom Rules

**Author:** Mancho Lovette Feseng
**Program context:** Block 1 (Network Security), Week 4 — Cybersecurity Program v2
**Builds on:** [Segmented Lab Network with pfSense Firewall](./week4-pfsense-firewall.md)
**Status:** In progress — default-deny behavior confirmed; custom allow rule written but not yet taking effect due to rule ordering, open issue tracked below

## Overview

This continues the Week 4 firewall build by moving from observing pfSense's *default* behavior to writing an explicit custom rule, and by digging into two core firewall concepts in practice: **stateful behavior** and **rule evaluation order (ACL design)**.

## Concept: stateful vs. stateless firewalls

- A **stateless** firewall evaluates every packet in isolation, with no memory of prior traffic. To allow a request and its reply, you'd need two separate rules — one for each direction.
- A **stateful** firewall (what pfSense is) tracks active connections. Allowing an outbound request automatically permits its return traffic, without a separate rule for the reply. This is why the default "allow LAN to any" rule was enough to let Kali's outbound pings get their replies back, with nothing needed on the WAN side for the *return* leg of LAN-initiated traffic.

## Concept: rule evaluation order (ACL design)

pfSense evaluates firewall rules **top to bottom on each interface, first match wins** — once a packet matches a rule, evaluation stops and later rules are never checked, even if they would also match. This is the central design principle behind Access Control Lists (ACLs) generally, not just in pfSense.

## What was done

Two default rules exist on the WAN interface out of the box, both aimed at anti-spoofing protection:
- **Block private networks** — rejects traffic whose source is an RFC 1918 (private) address
- **Block bogon networks** — rejects traffic from reserved/unassigned address ranges

In a real deployment, a WAN interface faces the public internet, where a private-looking source address is a strong sign of spoofed/forged traffic — so blocking it by default is a sound security default. In this lab, however, the WAN side is simulated using a private internal network (`192.168.59.0/24`), so these anti-spoofing rules end up incidentally blocking legitimate lab traffic as a side effect of their intended purpose.

A custom rule was created to explicitly allow ICMP (ping) from Metasploitable2 (`192.168.59.20`) to Kali (`192.168.58.10`):
- Protocol: ICMP, Subtype: Echo Request
- Source: `192.168.59.20/32`, Destination: `192.168.58.10/32`
- Narrowly scoped to a single protocol, single source, and single destination, rather than a broad allow — reflecting the principle of least privilege in ACL design.

## Verification and the open issue

Using pfSense's per-rule **State** counters (which show live traffic/byte counts matched by each rule) as a diagnostic tool, it was confirmed that the "Block private networks" and "Block bogon networks" rules were matching and incrementing on each test ping, while the custom ICMP rule's counter stayed at zero — direct evidence that the block rules were intercepting the traffic *before* the custom rule was ever evaluated, due to rule order (the block rules sit above the custom rule in the list).

**Current status:** the custom rule has been recreated multiple times at the top of the list (via "insert above selected rule"), but state evidence and live testing still show the block rules firing first. This is being treated as an open troubleshooting item rather than a blocker — the underlying concept (order determines outcome, first match wins) has already been clearly demonstrated by the evidence gathered so far, even though the specific fix hasn't landed yet.

## Key evidence gathered

| Test | Result | Interpretation |
|---|---|---|
| Kali → Metasploitable (LAN→WAN) | 4/4 received, TTL 63 | Default LAN rule + stateful return traffic work as expected |
| Metasploitable → Kali (WAN→LAN), no custom rule | 100% loss, no error | Default-deny on WAN, silent drop |
| Metasploitable → Kali, custom rule added but below block rules | 100% loss, block-rule state counters increment | Rule order confirmed as the cause — first match wins |

## Tools and techniques used

- pfSense CE 2.7.2 web interface (Firewall → Rules)
- Per-rule state counters as a live diagnostic tool for confirming which rule actually matches traffic
- `curl -k` and `ping` from Kali as independent connectivity checks

## Next steps

- Resolve the rule-ordering issue so the custom ICMP rule actually takes effect (either by successfully reordering, or by adding an explicit exception ahead of the anti-spoofing rules)
- Extend to a second rule allowing only SSH (port 22) as a more realistic real-world example than ICMP
- Carry the "first match wins" and least-privilege principles forward into Suricata rule-writing (Week 5)
