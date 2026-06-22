# SSH Brute Force Attack

## Background

I built this lab on a MacBook Pro. My original plan was to use Kali Linux as the attack machine, but I ran into compatibility issues getting it running in my environment. Rather than give up, I decided to simulate the attacks manually using tools available directly in the Ubuntu VM terminal. This meant running the attack commands from inside the same machine I was targeting.

---

## What Happened

I simulated a brute force attack by writing a bash loop that fired 20 consecutive SSH login attempts at the Ubuntu VM (192.168.64.11). I used `sshpass` to pass a hardcoded wrong password non-interactively, targeting a fake username that did not exist on the system.

**Attack command I ran in the Ubuntu VM terminal:**
<img width="817" height="130" alt="image" src="https://github.com/user-attachments/assets/da5eb4ba-cfc4-4e93-986f-dfa442ab0a23" />


When I sent this, the terminal went into a loading phase for about 10 to 15 seconds while it ran through all 20 attempts. Every attempt was rejected by the SSH server since the user did not exist, and each failure got written to `/var/log/auth.log`.

---

## Logs Found in Splunk

<img width="1512" height="859" alt="image" src="https://github.com/user-attachments/assets/5f0065d5-1106-49de-8f92-05a7218063fc" />


**What showed up in the logs** (repeated 20 times):
Failed password for invalid user fakeuser from 192.168.64.11 port XXXXX ssh2

Invalid user fakeuser from 192.168.64.11

Connection closed by invalid user fakeuser 192.168.64.11

**Total events in Splunk at time of detection:** 192 events  
**Source:** /var/log/auth.log | **Sourcetype:** linux_secure

<img width="1244" height="421" alt="image" src="https://github.com/user-attachments/assets/74fee96c-8f3d-4bd8-b0a1-c4cddb919102" />


Seeing these flood into Splunk in real time was the moment the lab clicked for me. I could watch exactly what the attack looked like from a defender's perspective.

### Key Indicators of Compromise
- 20 failed SSH logins within approximately 10 seconds
- All attempts targeting the same non-existent username (fakeuser)
- Consistent 0.5 second spacing, clearly scripted and not a human typing
- Source and destination IP both 192.168.64.11 (loopback in this lab setup)

---

## Incident Response

### Immediate Containment
The first priority is to stop the attack from continuing. I would block the offending IP address at the firewall level using UFW to prevent it from reaching port 22 entirely. At the same time I would check who is currently logged into the system to confirm no active sessions were established by the attacker during the attack window.

### Investigation
Once contained, I would look at the auth.log entries in Splunk to get a full count of how many failed attempts came from each IP address, and check whether any login attempts actually succeeded.

### Hardening Steps
The most important fix is to disable password-based SSH authentication entirely and require SSH key pairs instead, since no password means a brute force attack has nothing to guess. On top of that I would restrict SSH access to trusted IP addresses only using UFW, and set up a Splunk alert to notify me any time more than 5 failed logins come from the same IP within 60 seconds.

---

## Lessons Learned

Even without Kali Linux, I was able to generate realistic attack traffic and observe exactly how it looks inside a SIEM. The scripted, repetitive pattern of same username, same password, and fixed intervals is something a basic Splunk alert would catch immediately.
