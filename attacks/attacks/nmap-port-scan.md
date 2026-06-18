# IR-2026-002 — Nmap Port Scan / Reconnaissance

| Field | Details |
|-------|---------|
| **Date & Time** | June 17, 2026 - 19:55 EDT |
| **Target** | 192.168.64.11 (Ubuntu VM) - All TCP ports |
| **Attack Type** | Network Reconnaissance / Port Scan |
| **Tool Used** | Nmap 7.94SVN (run from Ubuntu VM terminal) |
| **Severity** | 🟡 MEDIUM |
| **Status** | Detected (Lab Environment) |

---

## Background

Since I was working on a MacBook Pro without Kali Linux, I ran this scan from inside the Ubuntu VM itself rather than from a separate attack machine. Nmap requires root privileges for SYN scanning, so the first time I ran it I got a `QUITTING!` error and fixed it by adding `sudo`. Running the attack from the same machine as the target works fine for learning purposes in a homelab.

---

## What Happened

I ran an aggressive Nmap scan against the Ubuntu VM to simulate what a real attacker would do before attempting to break in. The goal was to map out open ports, fingerprint the OS, and identify the exact software version running on each service. This kind of scan is almost always the first step in a real attack chain.

**First attempt (failed):**

nmap -sS -A -T4 192.168.64.11
# Error: You requested a scan type which requires root privileges. QUITTING!


**Fixed with sudo:**
sudo nmap -sS -A -T4 192.168.64.11


**Flags breakdown:**
| Flag | Meaning |
|------|---------|
| `-sS` | SYN stealth scan - probes ports without completing a full connection |
| `-A` | Aggressive mode: OS detection, version detection, script scanning, traceroute |
| `-T4` | Fast timing for local network scanning |

**Full Nmap output:**
Starting Nmap 7.94SVN at 2026-06-17 19:55 EDT

Nmap scan report for nathan-QEMU-Virtual-Machine (192.168.64.11)

Host is up (0.000059s latency).

Not shown: 999 closed tcp ports (reset)
PORT   STATE  SERVICE  VERSION

22/tcp open   ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey:

|   256 56:92:6e:40:d9:68:2d:d7:4e:db:f6:b6:a3:ca:63:79 (ECDSA)

|_  256 e3:0b:65:fa:75:4c:20:07:0d:9b:18:c3:cd:78:e8:6a (ED25519)
Device type: general purpose

OS details: Linux 2.6.32

Network Distance: 0 hops
Nmap done: 1 IP address (1 host up) scanned in 1.73 seconds

In under 2 seconds the scan revealed the only open port (22/SSH), the exact OpenSSH version, both SSH host key fingerprints, and the OS type. That is everything a real attacker needs to plan a targeted follow-up, which in this lab I did next with the brute force attack documented in IR-2026-001.

---

## Logs Found in Splunk

Nmap's SYN scan never completes a full TCP handshake so it does not generate normal connection logs. However I could still see evidence of it in Splunk through `/var/log/auth.log`. The SSH daemon logged the probe attempts Nmap made when running its scripts against port 22.

I accessed Splunk from my MacBook browser and ran these searches:

**Primary search (282 events returned):**
index=main source="/var/log/auth.log"

## Incident Response

### Immediate Containment
The first step is to block the scanning IP address at the firewall using UFW so it cannot probe the system further. I would then verify which services are currently listening on the network to confirm nothing unexpected was exposed that the attacker could now target based on what the scan revealed.

### Investigation
After containing the threat I would review the Splunk logs around the time of the scan to look for any follow-up activity from the same source IP, such as login attempts or connections to other ports. A port scan on its own is reconnaissance, but if it was followed immediately by authentication attempts it would suggest the attacker moved straight into an exploitation phase, which is exactly what happened in IR-2026-001.

### Hardening Steps
To reduce the information a scan like this can gather, I would disable the SSH version banner in the SSH configuration file so the service no longer advertises the exact software version to anyone who probes it. I would also install `psad`, a port scan attack detector that monitors incoming traffic and automatically blocks IPs that send an unusually high number of connection attempts in a short period. Longer term I would look into restricting SSH access to specific trusted IP addresses only, and set a Splunk alert to flag any burst of failed pre-authentication connection attempts from a single source.

---

## Lessons Learned

This scan directly set up the brute force attack in IR-2026-001. I already knew SSH was on port 22 and exactly which version was running before I started trying passwords. That is how real attack chains work: recon first, then exploit.

Not having Kali available pushed me to understand the tools at a deeper level. I had to look up each Nmap flag individually and understand why `sudo` was required, rather than just running a pre-built script. For better detection in future labs I would install `psad` from the start so Splunk gets full port scan visibility rather than just the SSH-level side effects.
