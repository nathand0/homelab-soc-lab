# Nmap Port Scan / Reconnaissance

## Background

Since I was working on a MacBook Pro without Kali Linux, I ran this scan from inside the Ubuntu VM itself rather than from a separate attack machine.

---

## What Happened

I ran a Nmap scan against the Ubuntu VM to simulate what a real attacker would do before attempting to break in. The goal was to map out open ports, fingerprint the OS, and identify the exact software version running on each service.


## Logs Found in Splunk

Nmap's SYN scan never completes a full TCP handshake so it does not generate normal connection logs. However I could still see evidence of it in Splunk through `/var/log/auth.log`. The SSH daemon logged the probe attempts Nmap made when running its scripts against port 22.

I accessed Splunk from my MacBook browser and ran these searches:

**Primary search (282 events returned):**
index=main source="/var/log/auth.log"

## Incident Response

### Immediate Containment
The first step is to block the scanning IP address at the firewall using UFW so it cannot probe the system further. I would then verify which services are currently listening on the network to confirm nothing unexpected was exposed that the attacker could now target based on what the scan revealed.

### Investigation
After containing the threat I would review the Splunk logs around the time of the scan to look for any follow-up activity from the same source IP, such as login attempts or connections to other ports. A port scan on its own is reconnaissance, but if it was followed immediately by authentication attempts it would suggest the attacker moved straight into an exploitation phase.

### Hardening Steps
To reduce the information a scan like this can gather, I would disable the SSH version banner in the SSH configuration file so the service no longer advertises the exact software version to anyone who probes it.

---

## Lessons Learned

This scan directly set up the brute force attack this lab. I already knew SSH was on port 22 and exactly which version was running before I started trying passwords. That is how real attack chains work: recon first, then exploit.

Not having Kali available pushed me to understand the tools at a deeper level. I had to look up each Nmap flag individually and understand why `sudo` was required, rather than just running a pre-built script.
