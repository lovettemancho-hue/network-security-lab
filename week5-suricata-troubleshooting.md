# Week 5 Troubleshooting Log: Getting Suricata Working on pfSense

## The short version

I installed Suricata (an intrusion detection system — think of it as a security camera that watches network traffic and flags anything suspicious) on my pfSense firewall. It looked "installed" but wasn't actually working right. Chasing that down led me into a much bigger fix: my firewall's entire operating system was out of date, and I had to upgrade it before the security tool could work properly. Below is the full story, explained simply, plus what I actually did to fix it.

---

## Part 1: The confusing error that started it all

After installing Suricata, the package list showed this message:

> "Package is configured but not (fully) installed or deprecated"

**What I thought at first:** Suricata itself was broken, so I tried reinstalling it.

**What was actually going on:** Reinstalling failed and gave me a much more specific warning:

> "Current pkg repository has a new PHP major version. pfSense should be upgraded before installing any new package."

### Explain it like I'm five

Imagine your firewall (pfSense) is a phone, and Suricata is an app you want to install.

- Your phone is running an **old version of its operating system**.
- The App Store has moved on and now only sells apps built for the **newer operating system**.
- When you try to install the app, the store says: "Sorry, your phone needs to update first."

That's exactly what was happening. My pfSense was on version **2.7.2**, but the package store had moved on to expect version **2.8.1**. Suricata wasn't broken — my firewall's OS was just out of date for it.

---

## Part 2: Why this is a big deal beyond this one lab

This isn't a pfSense-only quirk. It's a pattern that shows up everywhere in real computing:

- `apt install` (Linux) failing because a program needs a newer system library than you have
- `pip install` (Python) failing because a package dropped support for your Python version
- A phone app refusing to install because your phone's OS is too old

**The lesson:** package managers aren't just copying files onto your system — they're checking "does this thing actually match what you're running?" first. When an install fails, the real problem is often *underneath* the thing you're trying to install, not the thing itself.

---

## Part 3: Trying to fix the "old phone" problem

To fix this, I needed to update pfSense's base system (the "phone's operating system") from 2.7.2 to 2.8.1. This turned out to be its own adventure.

### Attempt 1: The update page said "up to date" — but it lied

When I first checked System Update, it said I was already "up to date" — but that was wrong. The real reason: the page has a dropdown for which "track" (branch) of updates to follow, and it was stuck pointed at the *old* track (2.7.2), so of course it thought I was current. I had to manually change that dropdown to point at the newer track (2.8.1) before it would even acknowledge a real update existed.

**Simple version:** it's like checking "is there a new season of my show?" but your streaming app is still set to only check Season 1's page. You have to point it at the right season first.

### Attempt 2: The actual download failed partway through

Once pointed at the right version, I started the real update. It needed to download **425 MB** of files and swap out **over 200 packages** — because this wasn't a small patch, it was a full operating-system-generation jump (the update log showed my whole system's low-level "ABI" changing from `freebsd:14` to `freebsd:15` — basically, the entire foundation the OS is built on was being swapped out).

Partway through downloading the very first (and largest) file, it failed. My connection just wasn't stable enough to get through the whole 425 MB in one go.

**Simple version:** imagine trying to download a giant movie file on shaky wifi — it can get partway through and just give up.

### Attempt 3: The failed download left things in a broken, half-updated state

After the failed download, pfSense's internal "shopping catalog" of available packages got corrupted — it couldn't tell what was available anymore, so every check just failed with vague errors.

I fixed this with a command that forces pfSense to throw out its broken catalog and grab a completely fresh one:

```
IGNORE_OSVERSION=yes pkg-static update -f
```

*(The `IGNORE_OSVERSION=yes` part was needed because, mid-upgrade, my system was straddling two OS generations at once — pkg wanted to double-check with me before proceeding, but the tool I was using couldn't accept a live "yes" answer, so this told it "yes" in advance.)*

### Attempt 4: Retried the real update — and this time it worked

With the catalog fixed and a more stable moment, I retried the actual base system upgrade. This time it completed successfully, rebooted, and came back up running the new operating system version.

**Confirmed:** System Update now shows **Current Base System: 2.8.1 / Latest Base System: 2.8.1 — Up to date.** Real this time.

---

## Part 4: Finishing the Suricata install

With the "phone" (base OS) finally updated, I could now properly reinstall Suricata:

1. Removed the old Suricata install completely (it was tied to the outdated system):
   ```
   pkg-static delete -y suricata pfSense-pkg-suricata
   ```
2. Installed Suricata fresh from the Available Packages page.
3. The Package Manager page *still* showed the same old "configured but not fully installed" message — but I learned not to trust that specific banner at face value anymore. Instead, I checked the **real** test: does Services > Suricata actually load and work? It did — full menu, all tabs present, fully functional. That old banner turned out to just be a stale/cosmetic message that doesn't always update correctly on this version of pfSense.
4. Added a monitoring interface: **WAN** (the side of my network facing Metasploitable, my intentionally vulnerable practice target) — this is the interface Suricata should "watch."
5. Clicked the start button to actually turn Suricata on for that interface.
6. Confirmed the **Emerging Threats Open** ruleset (a big library of known attack signatures) downloaded successfully with a real signature hash and timestamp.

**Final confirmed state:** Suricata is installed, running (green checkmark), monitoring the WAN interface, with a real rule set loaded.

---

## Key takeaways for future me (and anyone reading this)

- **A failed package install is often not about the package — check the base system version first.**
- **"Up to date" isn't always true** — if an update checker looks suspiciously wrong, check what branch/track it's actually comparing against.
- **A failed download can leave a system in a broken in-between state** — sometimes you need to force-refresh the underlying catalog/cache before retrying, not just try the same thing again.
- **Don't blindly trust every status banner** — cross-check with a real functional test (does the actual feature work?) when a message seems inconsistent with reality.
- **Interactive y/n prompts don't work in web-based command boxes** — look for a `-y` or similar auto-confirm flag instead.

---

*[Screenshots to be inserted here in order: initial error, failed reinstall/PHP warning, branch dropdown fix, download failure, catalog error, IGNORE_OSVERSION fix, successful upgrade confirmation, clean Suricata reinstall, WAN interface added, green running status, ET Open ruleset confirmed]*
