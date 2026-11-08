# Honeypot Threat Reports

A T-Pot honeypot running on Oracle Cloud that collects real-world attacks from the internet. I analyze the traffic, enrich attacker IPs with threat intelligence, map behavior to MITRE ATT&CK, and publish what I find here.

**Latest:** [Week 1 Threat Report (Nov 1-7)](reports/week-01.md): about 844K attack events, 16 ATT&CK techniques, a crypto miner and a Mirai dropper caught.

## What I built
- **Sensor:** T-Pot 24.04.1 (Hive install) with 20+ honeypots, including Cowrie (SSH/Telnet), Dionaea (SMB, databases, malware capture), Conpot (industrial control systems), and Suricata (IDS)
- **Cloud:** Oracle Cloud Ampere A1 VM (2 OCPU, 12 GB RAM, 200 GB disk), Ubuntu 26.04 ARM64
- **Analysis stack:** Elasticsearch + Kibana dashboards, AbuseIPDB lookups, VirusTotal hash lookups
- **Access control:** honeypot ports (1-64000) open to the internet; admin SSH and dashboard ports (64294-64297) restricted to my IP only

## Repo layout
| File | What's in it |
|---|---|
| `setup.md` | How I built it, the problems I hit, and how I fixed them |
| `reports/week-01.md` | Week 1 threat report (summary) |
| `reports/findings-log.md` | Day-by-day findings with evidence |

## Status
Live since November 1, 2026. Week 1 report (Nov 1-7) complete. Next: sending the honeypot logs to Microsoft Sentinel to build detection rules 

> Server IP and dashboard URL are intentionally left out of this repo.
