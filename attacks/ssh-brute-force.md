# IR-2026-001 — SSH Brute Force Attack

| Field | Details |
|-------|---------|
| **Date & Time** | June 17, 2026 - 19:53 EDT |
| **Target** | 192.168.64.11 (Ubuntu VM) - Port 22 (SSH) |
| **Attack Type** | SSH Brute Force / Credential Stuffing |
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
```bash
for i in {1..20}; do
  sshpass -p "wrongpassword" ssh -o StrictHostKeyChecking=no fakeuser@192.168.64.11 2>/dev/null
  sleep 0.5
done
```

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
```bash
# Block the offending source IP at the firewall
sudo ufw deny from 192.168.64.11 to any port 22
sudo ufw enable

# Check who is currently logged in
who
last | head -20
```

### Investigation
```bash
# Count failed attempts per IP to confirm scope
grep 'Failed password' /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn

# Check if any attempt actually succeeded
grep 'Accepted' /var/log/auth.log
```

### Hardening Steps
Disable password-based SSH authentication and switch to key pairs only:
```bash
# Edit /etc/ssh/sshd_config and set:
PasswordAuthentication no
PermitRootLogin no

sudo systemctl restart ssh
```

Install Fail2Ban to automatically block IPs after repeated failures:
```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban --now
```

Additional steps:
- Restrict SSH access to trusted IPs only via UFW
- Move SSH to a non-standard port to reduce automated scanning
- Create a Splunk alert: trigger when 5 or more failed logins come from the same IP within 60 seconds

---

## Lessons Learned

Even without Kali Linux, I was able to generate realistic attack traffic and observe exactly how it looks inside a SIEM. The scripted, repetitive pattern of same username, same password, and fixed intervals is something a basic Splunk alert would catch immediately. The biggest takeaway is that password-based SSH authentication should never be used in a real environment. Key pairs and Fail2Ban together would have stopped this attack before it reached 5 attempts.
