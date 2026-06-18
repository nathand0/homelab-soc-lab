# IR-2026-001 — SSH Brute Force Attack

| Field | Details |
|-------|---------|
| **Date & Time** | June 17, 2026 - 19:53 EDT |
| **Target** | 192.168.64.11 (Ubuntu VM) - Port 22 (SSH) |
| **Attack Type** | SSH Brute Force |
| **Tool Used** | sshpass + bash loop (macOS Terminal) |
| **Severity** | 🔴 HIGH |
| **Status** | Contained (Lab Environment) |

---

## Background

I built this lab on a MacBook Pro. My original plan was to use Kali Linux as the attack machine, but I ran into compatibility issues getting it running in my environment. Rather than give up, I decided to simulate the attacks manually using tools available directly in the Ubuntu VM terminal. This meant running the attack commands from inside the same machine I was targeting, which works fine for a homelab learning environment.

---

## What Happened

I simulated a brute force attack by writing a bash loop that fired 20 consecutive SSH login attempts at the Ubuntu VM (192.168.64.11). I used `sshpass` to pass a hardcoded wrong password non-interactively, targeting a fake username that did not exist on the system. The 0.5 second delay between attempts mimics the pacing a real attacker might use to avoid triggering basic rate limits.

**Attack command I ran in the Ubuntu VM terminal:**
for i in {1..20}; do
  sshpass -p "wrongpassword" ssh -o StrictHostKeyChecking=no fakeuser@192.168.64.11 2>/dev/null
  sleep 0.5
done


When I sent this, the terminal went into a loading phase for about 10 to 15 seconds while it ran through all 20 attempts. That is completely normal. Every attempt was rejected by the SSH server since the user did not exist, and each failure got written to `/var/log/auth.log`.

---

## Logs Found in Splunk

I accessed Splunk from my MacBook Pro browser while the Ubuntu VM ran in the background. After the attack finished I ran the following search in Splunk:

**Splunk search:** index=main sourcetype=linux_secure "Failed password"

**What showed up in the logs** (repeated 20 times):
Failed password for invalid user fakeuser from 192.168.64.11 port XXXXX ssh2

Invalid user fakeuser from 192.168.64.11

Connection closed by invalid user fakeuser 192.168.64.11

**Total events in Splunk at time of detection:** 192 events  
**Source:** `/var/log/auth.log` | **Sourcetype:** `linux_secure`

Seeing these flood into Splunk in real time was the moment the lab clicked for me. I could watch exactly what the attack looked like from a defender's perspective.

### Key Indicators of Compromise
- 20 failed SSH logins within approximately 10 seconds
- All attempts targeting the same non-existent username (`fakeuser`)
- Consistent 0.5 second spacing, clearly scripted and not a human typing
- Source and destination IP both 192.168.64.11 (loopback in this lab setup)

---

## Incident Response

### Immediate Containment
The first priority is to stop the attack from continuing. I would block the offending IP address at the firewall level using UFW to prevent it from reaching port 22 entirely. At the same time I would check who is currently logged into the system to confirm no active sessions were established by the attacker during the attack window.

### Investigation
Once contained, I would look at the auth.log entries in Splunk to get a full count of how many failed attempts came from each IP address, and check whether any login attempts actually succeeded. A successful login during a brute force window would be a serious escalation and would require a full review of everything that account accessed.

### Hardening Steps
The most important fix is to disable password-based SSH authentication entirely and require SSH key pairs instead, since no password means a brute force attack has nothing to guess. I would also install Fail2Ban, which automatically bans any IP that exceeds a set number of failed login attempts within a short window. On top of that I would restrict SSH access to trusted IP addresses only using UFW, and set up a Splunk alert to notify me any time more than 5 failed logins come from the same IP within 60 seconds.

---

## Lessons Learned

Even without Kali Linux, I was able to generate realistic attack traffic and observe exactly how it looks inside a SIEM. The scripted, repetitive pattern of same username, same password, and fixed intervals is something a basic Splunk alert would catch immediately. The biggest takeaway is that password-based SSH authentication should never be used in a real environment. Key pairs and Fail2Ban together would have stopped this attack before it reached 5 attempts.
