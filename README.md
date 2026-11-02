# Honeypot Threat Reports

A T-Pot honeypot running on Oracle Cloud that collects real-world attacks from the internet. I analyze the traffic, enrich attacker IPs with threat intelligence, map behavior to MITRE ATT&CK, and publish what I find here.

## What I built
- **Sensor:** T-Pot 24.04.1 (Hive install) with 20+ honeypots, including Cowrie (SSH/Telnet), Dionaea (SMB, databases, malware capture), Conpot (industrial control systems), and Suricata (IDS)
- **Cloud:** Oracle Cloud Ampere A1 VM (2 OCPU, 12 GB RAM, 200 GB disk), Ubuntu 26.04 ARM64
- **Analysis stack:** Elasticsearch + Kibana dashboards, AbuseIPDB lookups, CyberChef
- **Access control:** honeypot ports (1-64000) open to the internet; admin SSH and dashboard ports (64294-64297) restricted to my IP only

## Repo layout
| File | What's in it |
|---|---|
| `setup.md` | How I built it, the problems I hit, and how I fixed them |
| `reports/findings-log.md` | Running notes on interesting things I find |
| `reports/week-XX.md` | Weekly threat reports |
| `reports/final.md` | Summary after the full collection period |

## Status
Live since November 1, 2026.

> Server IP and dashboard URL are intentionally left out of this repo.
