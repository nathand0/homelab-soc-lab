# Home SOC Lab - Ubuntu + Splunk
A personal cybersecurity homelab where I built a SIEM environment to simulate, detect, and respond to real attacks.

## Environment
- **VM:** Ubuntu 24.04 (QEMU/KVM) - IP: 192.168.64.11
- **SIEM:** Splunk (free tier) ingesting `/var/log/auth.log` and syslog
- **Host:** macOS (Splunk web UI accessed from host)

## Attacks Simulated
| # | Attack | Tool | Severity | Status |
|---|--------|------|----------|--------|
| 1 | SSH Brute Force | sshpass + bash loop |  HIGH | Detected |
| 2 | Port Scan / Recon | Nmap -sS -A -T4 |  MEDIUM | Detected |

## Incident Reports
- [SSH Brute Force](attacks/ssh-brute-force.md)
- [Nmap Port Scan](attacks/nmap-port-scan.md)

## Skills Demonstrated
- Linux system administration and VM setup
- SIEM configuration and log ingestion (Splunk)
- Attack simulation and detection
- Incident response documentation
