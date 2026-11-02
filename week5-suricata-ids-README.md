# Week 5: Suricata IDS on pfSense, and the "Filtered" Mystery

**Short version:** I installed Suricata, an intrusion detection system (IDS), on my pfSense firewall. The install looked broken, which led me into upgrading the firewall's whole operating system. Once Suricata ran, my Nmap scans showed every port as "filtered", and it took me weeks and a proper step-by-step investigation to find out why. The cause was not the firewall. It was a wrong route on my Kali machine.

This writeup tells the story in the order it happened, explains every idea in plain words, and keeps the evidence (screenshots) next to each claim.

---

## Contents

1. [What I was trying to build](#1-what-i-was-trying-to-build)
2. [Part 1: The install that looked broken](#part-1-the-install-that-looked-broken)
3. [Part 2: Upgrading pfSense itself](#part-2-upgrading-pfsense-itself)
4. [Part 3: A clean Suricata install](#part-3-a-clean-suricata-install)
5. [Part 4: The scans that said "filtered"](#part-4-the-scans-that-said-filtered)
6. [Part 5: Finding the real cause](#part-5-finding-the-real-cause)
7. [What I learned](#what-i-learned)
8. [Cheat sheet](#cheat-sheet)
9. [Still open](#still-open)

---

## 1. What I was trying to build

**The goal:** put a "security camera" on my network that watches traffic and raises an alert when something looks like an attack.

**The lab (all virtual machines, fully isolated from the internet):**

```
 Kali (attacker)                    pfSense (firewall + Suricata)                  Metasploitable2 (target)
 192.168.58.10  ----eth1----  LAN em0 192.168.58.1 | WAN em1 192.168.59.1  ----  192.168.59.20
                                                    ^
                                      Suricata watches the WAN side
                                      (OPT1 em2 = NAT, internet for package downloads only)
```

**Plain words:**
- **Kali** is the "attacker" machine I run tools from.
- **pfSense** is the firewall in the middle. Every packet between the two sides passes through it.
- **Metasploitable2** is a server that is deliberately full of security holes, so I can practise on something legal and safe.
- **Suricata** is the camera. It compares traffic against a library of known attack patterns (called *rules*) and logs an alert on a match.

**IDS vs IPS:** an IDS only watches and reports. An IPS also blocks. My Suricata is set to **Blocking Mode: DISABLED**, so it is an IDS.

---

## Part 1: The install that looked broken

I installed the Suricata package from pfSense's Package Manager. Then the *Installed Packages* page showed:

> Package is configured but not (fully) installed or deprecated

![Installed packages page with the warning](images/01-installed-packages-banner.png)

My first thought was "Suricata is broken, reinstall it". The reinstall failed with a much more helpful message:

> WARNING: Current pkg repository has a new PHP major version. pfSense should be upgraded before installing any new package.

![Reinstall failed with the PHP version warning](images/02-reinstall-failed-php-warning.png)

### Explain it like I'm five

Imagine pfSense is a phone and Suricata is an app. The app store has moved on and only sells apps built for the **newest** version of the phone's system. My phone was on an **old** version (pfSense 2.7.2), and the store expected 2.8.1. The app was not broken. My phone was too old for it.

### Why this matters beyond one lab

This is a common pattern. `apt install` fails because a library is too old, `pip install` fails because Python is too old, a phone app refuses to install on an old Android. A package manager does not just copy files. It first checks "does this match what you are running?" When an install fails, the problem is often underneath the thing you are installing.

---

## Part 2: Upgrading pfSense itself

I had to move the base system from **2.7.2 to 2.8.1**. That turned into four separate problems.

### Problem 1: "Up to date" was wrong

The System Update page said I was up to date. It was comparing against the wrong track: the **Branch** dropdown was set to 2.7.2, so of course 2.7.2 was the latest.

![Branch set to 2.7.2, shows up to date](images/03-system-update-branch-272-up-to-date.png)

*Simple version:* it is like asking "is there a new season?" while the app is pointed at Season 1's page.

I switched the branch to 2.8.1. The page then said **Unable to check for updates**.

![Branch 2.8.1, unable to check for updates](images/04-branch-281-unable-to-check.png)

### Problem 2: The download died halfway

When the update finally started, it listed a huge job: **425 MiB** to download and over 190 packages to change. Every package showed `ABI changed: 'freebsd:14' -> 'freebsd:15'`. The **ABI** (Application Binary Interface) is the low-level contract between programs and the operating system. A change from `freebsd:14` to `freebsd:15` means the whole foundation was being swapped.

![Update plan with ABI change and 425 MiB download](images/05-update-abi-change-425mib.png)

The very first file failed to download. My internet connection was not stable enough for one big transfer.

![First download failed](images/06-update-download-failed.png)

### Problem 3: The failure left a broken catalog

After the failure, pfSense could no longer read its list of available packages. The update page showed confusing messages:

![pfSense-repoc: no package message](images/07-repoc-no-package-message.png)

I used pfSense's built-in **Diagnostics > Command Prompt** to force a fresh catalog download:

```
pkg-static update -f
```

`-f` means *force*: throw away the old catalog and fetch a new one. pkg then asked a yes/no question about the OS mismatch, which a web command box cannot answer:

![pkg asks to ignore the OS version mismatch](images/08-pkg-update-ignore-osversion-prompt.png)

The fix is to answer "yes" in advance with an environment variable:

```
IGNORE_OSVERSION=yes pkg-static update -f
```

My first try had a typo (`update-f` with no space), so pkg said `unknown command`:

![Typo in the command](images/09-typo-update-f.png)

*Lesson: one missing space can look like a deep technical failure. Copy and paste commands.*

I also tried `pfSense-repoc`, which is a separate tool from `pkg-static`, and learned it does not accept the same options. The update page still reported `failed to fetch the repo data`:

![repoc failed to fetch the repo data](images/10-repoc-failed-to-fetch.png)

### Problem 4: The update, at last

With the catalog fixed and a steadier connection, I retried the update. pfSense rebooted. For a few minutes the web interface was unreachable, which is expected during a reboot:

![Unable to connect while pfSense reboots](images/11-pfsense-rebooting-unable-to-connect.png)

When it came back, the page confirmed the real result:

![Base system 2.8.1, up to date](images/12-base-281-up-to-date.png)

---

## Part 3: A clean Suricata install

The old Suricata was tied to the old operating system, so I removed it completely and started fresh.

The delete command stopped at a yes/no prompt that the web command box cannot answer:

![Delete asks y/N](images/13-delete-suricata-confirm-prompt.png)

The fix is the `-y` flag, which means "assume yes":

```
pkg-static delete -y suricata pfSense-pkg-suricata
```

![Both packages removed](images/14-delete-suricata-done.png)

*Remember: web-based command boxes cannot answer interactive prompts. Look for a `-y` or similar flag.*

I reinstalled both packages from the Package Manager. The "configured but not fully installed" banner **stayed on the page**, but I stopped trusting it. The real test is whether the feature works, so I opened **Services > Suricata**. The menu was complete and working, just with no interfaces yet:

![Suricata interfaces tab, empty](images/15-services-suricata-empty.png)

I added the **WAN** interface, the side facing Metasploitable, which is the side I want to protect:

![Choosing the WAN interface](images/16-add-wan-interface.png)

![WAN added, stopped](images/17-wan-interface-stopped.png)

I also confirmed the **Emerging Threats Open** rule set had downloaded. The MD5 hash and timestamp prove it:

![ET Open rules downloaded](images/18-et-open-rules-downloaded.png)

Then I started Suricata on WAN. The green tick means it is running:

![Suricata running on WAN](images/19-suricata-running-green.png)

**Where things stood:** Suricata installed, running, watching the WAN, with real rules loaded.

---

## Part 4: The scans that said "filtered"

To get Suricata to alert, I needed to attack Metasploitable from Kali with Nmap. This is where the second long problem began.

### First, mistakes of my own

- The first scan hung on "Ping Scan: 0.00% done" for minutes:

  ![Nmap stuck on the ping scan](images/20-nmap-stuck-ping-scan.png)

- I typed `-pn` (lowercase) instead of `-Pn`, and wrong IP addresses (`192.16.59.20`, `192.168.168.59.20`):

  ![Typos in the Nmap command](images/21-nmap-typos.png)

  *Capital `-P` is about the "ping" check. Lowercase `-p` selects ports. Case matters.*

### The scan that finished, with no good news

A SYN scan (`-sS`, the default for root) took **211 seconds** and reported:

> All 1000 scanned ports on 192.168.59.20 are in ignored states. Not shown: 1000 filtered tcp ports (no-response)

![SYN scan progress](images/22-nmap-syn-scan-progress.png)

![All 1000 ports filtered](images/23-nmap-all-ports-filtered.png)

**What "filtered" means:** Nmap sent packets and got **no answer at all**. It does not say who dropped them or why. It is not the same as *closed*, where the target answers "no" immediately.

### How I checked the obvious things

| Check | Result | What it showed |
|---|---|---|
| Metasploitable's own firewall (`sudo iptables -L -n`) | All chains policy ACCEPT, no rules | The target is not blocking |
| Is the FTP service running? (`sudo netstat -antp \| grep :21`) | `0.0.0.0:21 LISTEN` handled by `xinetd` | The service is up |
| pfSense WAN rules | My ICMP allow rule showed `0/0 B` states | That rule never matched anything |
| pfSense firewall log | `Default deny` entries for traffic from 192.168.59.20 | Normal side traffic (see below) |

![Metasploitable firewall is wide open](images/27-metasploitable-iptables-open.png)

![vsftpd listening on port 21](images/28-vsftpd-listening-port21.png)

![WAN ICMP rule with zero states](images/24-wan-icmp-rule-zero-states.png)

![pfSense firewall log with default deny entries](images/25-firewall-log-default-deny.png)

![Single-port scan on 21: filtered](images/26-nmap-port21-filtered.png)

About the firewall log: the entries show Metasploitable itself trying to open connections back to Kali (port 113, "ident") and to a DNS server. pfSense blocks new connections from the WAN side by default, which is correct behaviour. Those blocks are **not** the reason the scan failed.

### My theory, and why it was wrong to trust it

I guessed that pfSense's built-in "Block private networks" and "Block bogon networks" options on the WAN were dropping the traffic. I toggled the checkboxes, but I **never proved** it. I tried to confirm it with a packet capture on pfSense, but I gave up after repeated mistakes in the form (a wrong IP, for example).

![The packet capture form I abandoned](images/29-packet-capture-form-abandoned.png)

I moved on to Week 6 without solving it. That open question stayed with me until the next time I picked it up.

---

## Part 5: Finding the real cause

On Nov 2 I came back and did it properly. Instead of guessing, I followed one packet along its whole trip and tested the start, the middle and the end.

### The packet's journey

1. **Kali** picks a network card from its routing table.
2. **pfSense LAN** checks its rules.
3. **pfSense** passes the packet toward the WAN.
4. **Metasploitable** receives it and replies.
5. **pfSense** matches the reply to the original request and forwards it.
6. **Kali** receives the reply and Nmap reports `open`.

"Filtered" means the packet died somewhere on that trip. The question is where.

### Evidence 1: Nmap says why

```
sudo nmap -Pn -n -p 21 --reason 192.168.59.20
```

The `--reason` flag prints the exact reply. The result:

```
21/tcp filtered ftp   no-response
```

`no-response` means silent. No reset, no "prohibited" message.

![Nmap: no-response](images/30-nov2-nmap-no-response.png)

### Evidence 2: Did the packet leave Kali toward pfSense?

I listened on Kali's `eth1` (the card that faces pfSense) while scanning:

```
sudo tcpdump -n -i eth1 host 192.168.59.20
```

**Nothing appeared.** The scan packets never went out of `eth1`.

![tcpdump on eth1 shows nothing](images/31-nov2-tcpdump-eth1-silent.png)

That ruled out pfSense and Metasploitable as the first suspects. The packet had not even reached them.

### Evidence 3: Which way does Kali send it?

```
ip route get 192.168.59.20
```

Result:

```
192.168.59.20 via 10.0.2.2 dev eth0 src 10.0.2.15
```

Kali was sending the packet out **`eth0`** (its internet card, via the VirtualBox NAT gateway at `10.0.2.2`), and not out `eth1` toward pfSense. Kali had no saved rule saying "to reach 192.168.59.0/24, go through pfSense", so the packet took the default route and vanished.

### The fix

```
sudo ip route add 192.168.59.0/24 via 192.168.58.1 dev eth1
ip route get 192.168.59.20      # now: via 192.168.58.1 dev eth1
sudo nmap -Pn -n -p 21 --reason 192.168.59.20
```

The scan came back in 0.06 seconds, instead of 2 seconds of waiting:

```
21/tcp open ftp   syn-ack   ttl 63
```

![Port 21 open, and the list of connections](images/32-nov2-nmap-open-and-nmcli.png)

**Reading that line:**
- `open` and `syn-ack`: Metasploitable answered the TCP handshake, so the service is listening.
- `ttl 63`: **TTL** (Time To Live) starts at 64 on Linux and loses 1 at every router. Seeing 63 proves **exactly one router, pfSense, sat in the path**.

### Making it permanent

A route added with `ip route add` lives only in memory and disappears on reboot. I stored it in the `labnet` connection's saved settings (found with `nmcli con show`):

```
sudo nmcli con mod labnet +ipv4.routes "192.168.59.0/24 192.168.58.1"
sudo nmcli con up labnet
```

![Permanent route saved and verified](images/33-nov2-permanent-route.png)

---

## What I learned

1. **Do not blame the obvious suspect without evidence.** I blamed the firewall for weeks. The packet had never reached it.
2. **"Filtered" is a symptom, not a diagnosis.** It means "no answer". Finding out why needs evidence along the path.
3. **Bisect.** Test the start, middle and end of the path to find which half the packet dies in.
4. **Measure with the right tool.** `tcpdump` on the sending side answered in one minute what weeks of theories could not.
5. **`ip route get` is a truth-teller.** It shows what Linux will actually do with a packet.
6. **"Host is up" under `-Pn` means nothing.** With `-Pn`, Nmap prints it without checking.
7. **A failing install is often about what is underneath** (old OS, old library), not the thing being installed.
8. **A stale banner is not proof.** Test the real feature.

### Honest caveat

The Oct 1 failure and the Nov 2 failure had the same symptom, and the Nov 2 cause is fully proven by the three pieces of evidence above. I did not check Kali's route on Oct 1, so I cannot prove it was the same cause on that day. What I can say is that today the path works end to end and the fix survives a reboot (to be re-checked after my next restart).

---

## Cheat sheet

| Command | What it does |
|---|---|
| `ip route get <IP>` | Shows the card and gateway Linux will use for that IP |
| `ip route` | Prints the whole routing table |
| `sudo tcpdump -n -i eth1 host <IP>` | Prints packets on a card for one IP |
| `sudo nmap -Pn -n -p 21 --reason <IP>` | Scans one port and says why it got its result |
| `sudo ip route add <net> via <gw> dev <if>` | Adds a temporary route |
| `nmcli con show` | Lists saved network connections |
| `sudo nmcli con mod <name> +ipv4.routes "<net> <gw>"` | Saves a route permanently |
| `IGNORE_OSVERSION=yes pkg-static update -f` | Forces a fresh pfSense package catalog |
| `pkg-static delete -y <pkg>` | Removes a package without asking |

| Nmap flag | Meaning |
|---|---|
| `-Pn` | Skip the "is anyone home?" check |
| `-n` | No DNS lookups |
| `-p 21` | Scan port 21 only |
| `--reason` | Show why each port got its verdict |
| `-sV` | Detect service versions |

| Nmap result | Meaning |
|---|---|
| `open` | Something answered the handshake |
| `closed` | The host answered "no" |
| `filtered` | No answer at all, cause unknown |

---

## Still open

- Generate a real alert in Suricata's **Alerts** tab from a Kali scan of Metasploitable.
- Write a custom Suricata rule for the vsftpd 2.3.4 backdoor and document the alert-to-analysis chain.
- The Week 4 ICMP allow rule (WAN) still shows `0/0 B` states. Re-test it now that the route works.
- Re-check the saved route after the next Kali reboot.
- Change the default pfSense admin password.
