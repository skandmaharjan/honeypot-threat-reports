# Findings Log

Quick notes as I find things. These feed the weekly reports.

## 2026-11-01: Day 1

**1. Attacked within minutes.** The Attack Map showed **243 attacks in roughly the first 20 minutes** after the honeypot went live. The server had no domain name, no website, and had never been advertised anywhere. Bots found it by scanning the whole internet.
- ATT&CK: T1595 Active Scanning

**2. Repeated hits on a fake API.** One IP geolocated to Germany sent repeated requests to **Honeyaml on port 3000** (a fake web API; port 3000 is common for dev servers and dashboards like Grafana), many within the same second. The speed and repetition point to an automated scanner looking for exposed admin panels and APIs, not a person.
- IP: 217.154.210.250 (AbuseIPDB lookup: TODO)
- ATT&CK: T1595 Active Scanning

**3. A known-bad Oracle Cloud IP.** AbuseIPDB lookup on 159.54.168.110:
- Abuse confidence **80% (Critical)**, **2,443 reports from 36 reporters**, first reported March 2026, still active (last report 37 minutes before my lookup)
- ISP: **Oracle Corporation**, data center/hosting, San Jose, CA
- Top report categories: Port Scan (884), SSH (267), Brute-Force, Hacking
- Takeaway: attackers rent the same cheap cloud servers defenders use. Blocking by country doesn't help much when the attacker sits in a U.S. data center.
- ATT&CK: T1595 Active Scanning, T1110 Brute Force

**4. Hunting for exposed Docker.** Port **2376** (the Docker API over TLS) showed up in the top 5 targeted ports on day 1. Attackers look for exposed Docker daemons so they can launch their own containers, usually crypto miners.
- ATT&CK: T1595 Active Scanning (and T1496 Resource Hijacking as the likely goal)

**5. A password list built around one person.** In Cowrie, the top username was **daryn**, and the top passwords were variations on it: `daryn`, `daryn1`, `daryn123`, `daryn1234`, plus date-like strings such as `03021977` and `03011981`. Generic bots try `admin`/`123456`. A list built around one name and birthdates points to a **leaked credential list** being replayed (credential stuffing).
- ATT&CK: T1110.004 Credential Stuffing

**6. Attackers working from neighboring IPs.** The top SSH sources included pairs of neighboring addresses (2.57.121.112 and 2.57.121.25; 45.148.10.151 and 45.148.10.152). Adjacent IPs attacking together usually means one operator renting a block of hosting servers.
- Top attacker ASNs: Unmanaged Ltd (AS47890), Techoff Srv Limited (AS48090), Scloud Pte Ltd (AS142002)

**7. No successful logins yet.** The Cowrie command input table was empty on day 1. Nobody had logged in and run commands yet.

**Defender takeaway:** Any service exposed to the internet gets probed within minutes. Admin ports should never be open to 0.0.0.0/0, and IP reputation feeds (like AbuseIPDB) can block a lot of this noise automatically. Finding 5 is a reminder that reused passwords from old breaches are still being tried years later, so unique passwords and MFA matter.
