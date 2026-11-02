# Findings Log

Quick notes as I find things. These feed the weekly reports.

## 2026-11-01: Day 1

**1. Attacked within minutes.** The Attack Map showed **243 attacks in roughly the first 20 minutes** after the honeypot went live. The server had no domain name, no website, and had never been advertised anywhere. Bots found it by scanning the whole internet.
- ATT&CK: T1595 Active Scanning

**2. Repeated hits on a fake API.** One IP geolocated to Germany sent repeated requests to **Honeyaml on port 3000** (a fake web API; port 3000 is common for dev servers and dashboards like Grafana), many within the same second. The speed and repetition point to an automated scanner looking for exposed admin panels and APIs, not a person.
- IP: 217.154.210.250
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

## 2026-11-02: Day 2

**8. Volume exploded overnight.** About **152,000 attack attempts in the first 24 hours**. By honeypot: Sentrypeer (VoIP) 70K, Cowrie (SSH/Telnet) 48K, Honeytrap 21K, Dionaea 10K.

**9. One host made ~40% of all traffic, and it was hunting phone systems.** Sentrypeer (fake SIP/VoIP server, port 5060) was the #1 target. Port 5060 traffic formed a flat block of about 8,000 hits/hour overnight, which looked like a single campaign. Filtering to Sentrypeer confirmed it:
- **172.110.223.9** sent **63,830** hits, about 91% of all VoIP traffic and roughly 40% of everything the honeypot saw that day
- Its neighbor **172.110.223.49** sent another 1,560 (same operator, same block)
- Both are in **AS47154** (Dedires LLC, data center hosting)
- AbuseIPDB: **100% abuse confidence (Critical)**, 480 reports from 105 reporters, still active
- Likely goal: find exposed phone systems to hijack for **toll fraud** (placing expensive calls on someone else's bill)
- ATT&CK: T1595 Active Scanning, T1110 Brute Force

**10. Geolocation disagreed.** The Kibana country chart spiked for the Philippines in the same window, so I guessed it was the same campaign. AbuseIPDB (IPinfo data) places 172.110.223.9 in **New York, US**. IP geolocation databases often disagree, especially for hosting IPs, so country data is a weak signal and shouldn't be trusted on its own.

**11. Old exploits never die.** Port 445 (Windows SMB) drew about 10K hits on Dionaea. That's the port WannaCry/EternalBlue used in 2017, and bots are still scanning for unpatched machines years later.
- ATT&CK: T1190 Exploit Public-Facing Application

**12. One bot spread across three IPs.** 160.119.71.250, .251 and .252 sent nearly identical counts (129/128/122).

**13. SSH brute force (Cowrie, 48K attempts).** Almost all of it hit port 22, with a little on Telnet (23).
- **Coordinated trio:** 109.160.32.108, .114 and .147 each sent about 6,000 attempts (6,004 / 5,998 / 5,987). Near-identical counts from neighboring IPs point to one operator splitting the work. All are in AS198364 (Banatsync Srl), the #1 network with 23K attempts.
- **Another pair:** 77.239.124.70 and .76 sent 2,513 and 2,512.
- ATT&CK: T1110.001 Password Guessing, T1110.003 Password Spraying

**14. Usernames show what bots are hunting for.** Besides `root` and `admin`, the list is full of default and service accounts:
- Cloud defaults: `ubuntu`, `ec2-user` (AWS), `oracle`
- Databases and apps: `postgres`, `mysql`, `hadoop`, `nginx`, `git`, `odoo`, `frappe`
- Devices and game servers: `pi` (Raspberry Pi), `minecraft`, `steam`, `ts3server`
- DevOps: `deploy`, `deployer`, `devops`, `backup`
- New-ish: `claude` and `openclaw`, which look like bots guessing accounts people create when they host AI agents on a VPS
- Takeaway: attackers target accounts that software creates automatically or that admins create in a hurry.

**15. Passwords are mostly the classics,** like `123456`, `password`, `12345678`, `qwerty`, `1qaz@WSX` and `P@ssw0rd`. One odd value, **`345gs5662d34`**, was also tried as a username. It shows up constantly in honeypot data reported by other researchers.

**16. What bots do after they "get in" (Cowrie Input).** About 195 sessions ran the exact same command sequence, so this is one automated script:

| Command | What it does | ATT&CK |
|---|---|---|
| `uname -m` | CPU architecture (x86 or ARM?) | T1082 System Information Discovery |
| `cat /proc/cpuinfo \| grep name \| wc -l` | Counts CPU cores | T1082 |
| `free -m`, `df -h` | RAM and disk size | T1082 |
| `crontab -l` | Lists scheduled tasks (theirs or a rival's) | T1053.003 Cron |
| `cd ~; chattr -ia .ssh; lockr -ia .ssh` | Removes the "immutable" lock on the `.ssh` folder so they can add their own SSH key | T1098.004 SSH Authorized Keys |
| `ls -lh $(which ls)` | Checks whether `ls` is real or a honeypot fake | T1497 Virtualization/Sandbox Evasion |

- Interpretation: the script sizes up the machine's CPU and RAM (is it worth mining crypto on?), then tries to plant a permanent SSH backdoor. The same command pattern is widely reported by other honeypot operators.

**17. Same tool, different IPs (HASSH).** Cowrie saw 277 unique source IPs but only 27 unique HASSH fingerprints. HASSH fingerprints the attacker's SSH *client software*. The trio 109.160.32.108/.114/.147 and 161.132.48.211 share one HASSH, which supports the idea that it's the same tool or operator. Client banners include `SSH-2.0-Go`, `libssh`, `PuTTY`, `Nmap` and `ZGrab`.

**18. Malware downloads.** Attackers fetched `payload.sh` from 165.245.199.76 (4 times) and dropped files named `.16`, `.b0s` and `sshd`. Hidden dot-names and a fake `sshd` (pretending to be the real SSH server) are classic tricks to blend in.
- ATT&CK: T1105 Ingress Tool Transfer, T1036.005 Masquerading
- Cowrie saved 8 files, named by SHA-256 hash. I looked them up on VirusTotal (hash search only, never downloaded or run). One hash (01ba4719...) is just a file containing a blank line, so that download failed.
- **`sshd` = crypto miner.** Hash 94f2e4d8d4436874785cd14e6e6d403507b8750852f7f2040352069a75da4c00: **47/64 vendors** flag it, label `miner.genericrxss/multiverze`, a 28.9 MB 64-bit Linux ELF. A miner disguised as the SSH server. This **confirms finding 16**: the recon script checked CPU and RAM because the goal was crypto mining. ATT&CK: T1496 Resource Hijacking, T1036.005 Masquerading.
- **`payload.sh` = Mirai dropper, barely detected.** Hash c363b0179cbd542865af005d2cc861525e5a5fdb2cd955eeb45741e99690be45: only **2/40 vendors** flag it (Kingsoft: Linux.Backdoor.Mirai.a; Microsoft: Trojan:Script/Wacatac). Sandbox tags: self-delete, persistence, sets-process-name, detect-debug-environment. Mirai turns infected devices into a DDoS botnet. **38 of 40 antivirus engines missed it**, so a low score does not mean safe. ATT&CK: T1105, T1070.004 File Deletion, T1497.

**Defender takeaway (Day 2):** Blocking or rate-limiting the top 1-2 source IPs would remove a huge share of the noise. Phone systems should never expose SIP (5060) to the whole internet, and SMB (445) should never be internet-facing. For SSH: disable password logins (keys only), disable root login, and remove default accounts.
