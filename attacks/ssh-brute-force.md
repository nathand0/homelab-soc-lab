# IR-2026-001 — SSH Brute Force Attack

| Field | Details |
|-------|---------|
| **Date & Time** | June 17, 2026 — 19:53 EDT |
| **Target** | 192.168.64.11 (Ubuntu VM) — Port 22 (SSH) |
| **Attack Type** | SSH Brute Force / Credential Stuffing |
| **Tool Used** | sshpass + bash loop |
| **Severity** | 🔴 HIGH |
| **Status** | Contained (Lab Environment) |

---

## What Happened

An attacker ran a scripted bash loop using `sshpass` to fire 20 consecutive SSH login attempts against the target VM. Each attempt used the same fake username (`fakeuser`) and wrong password, spaced 0.5 seconds apart to simulate a low-and-slow credential stuffing pattern.

**Attack command:**
```bash
for i in {1..20}; do
  sshpass -p "wrongpassword" ssh -o StrictHostKeyChecking=no fakeuser@192.168.64.11 2>/dev/null
  sleep 0.5
done
```

All 20 attempts failed — the user account did not exist on the target system. Every failure was logged to `/var/log/auth.log` and ingested by Splunk.

---

## Logs Found in Splunk

**Splunk search used:**
