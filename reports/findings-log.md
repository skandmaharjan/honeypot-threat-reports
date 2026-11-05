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

## 2026-11-03: Day 3

**19. Traffic dropped by half: 72K attacks vs 152K on Day 2.** Most of the drop is Sentrypeer (SIP), which fell from 70K to 11K. The VoIP toll-fraud campaign from finding 9 slowed down or stopped. Big campaigns come and go, so one day's top attacker isn't a permanent fixture.

**20. Honeytrap became the #1 honeypot (30K).** Honeytrap listens on every port the other honeypots don't use. With the SIP campaign gone, the biggest source of traffic is broad port scanning: bots checking every port to see what's open.

**21. An SMB burst: 2,667 hits on port 445 in one hour.** Around 11 PM on Nov 2, port 445 (Windows file sharing, the port WannaCry spread through) jumped from a few hundred hits an hour to 2,667. Unique source IPs stayed flat at about 400 during the spike, so a small number of IPs caused it, not a wave of new attackers.
- ATT&CK: T1190 Exploit Public-Facing Application (EternalBlue-style), T1110 Brute Force

**22. RDP brute force from two hosting-provider IPs.** RDPHoneypot (port 3389, Windows Remote Desktop) logged about 8K events:

| Source IP | ASN | Hits |
|---|---|---|
| 213.136.90.193 | AS51167 Contabo GmbH | 4,055 |
| 194.180.48.161 | AS201814 MEVSPACE sp. z o.o. | 1,692 |
| 152.53.82.10 | AS214996 netcup GmbH | 611 |

- The top two IPs made about 70% of all RDP traffic. All three are cheap VPS hosts, which attackers commonly rent to run scanners.
- About a quarter of events were `rdphoneypot.login`, so these bots were trying credentials, not just checking whether the port was open. ATT&CK: T1110 Brute Force, T1021.001 Remote Desktop Protocol.
- RDP attacks climbed from about 100 to 550 an hour during the afternoon while unique source IPs stayed flat, so the same few IPs were attacking harder.

**23. Two kinds of traffic.** Concentrated campaigns look like a huge spike in attacks with a flat unique-IP line (the 11 PM SMB burst: ~5,000 attacks/hour from ~400 IPs). Background scanning looks like many IPs each doing a little (1 PM: 2,195 attacks from 873 unique IPs). Comparing those two lines is a quick way to tell a targeted burst from internet noise.

**Defender takeaway (Day 3):** RDP and SMB should never be exposed to the internet. Put RDP behind a VPN or gateway with MFA and account lockout, and block 445 at the edge. Blocking or rate-limiting a handful of VPS IPs would have removed most of today's RDP traffic.

## 2026-11-04: Day 4

**24. A VNC campaign doubled traffic back to 151K.** Heralding (a honeypot that captures login attempts on many protocols) jumped to #1 with 49K, and port 5900 (VNC, remote desktop for Linux/Mac/Windows) became the top port. It ran as a flat block of about 1,500 hits an hour from roughly 3 AM until late evening, which looks like one long-running scanner rather than random noise.
- Suricata confirms it: "GPL INFO VNC server response" (98,675), "ET EXPLOIT VNC Server Not Requiring Authentication" (49,470) and "ET INFO VNC Authentication Failure" (49,324).
- Top CVE was **CVE-2006-2369** (49,474), a 20-year-old RealVNC authentication bypass. Bots are still checking for it.
- The most common password was **blank**, consistent with bots looking for VNC servers with no password set.
- ATT&CK: T1021.005 Remote Services: VNC, T1110 Brute Force, T1190 Exploit Public-Facing Application

**25. The 109.160.32.x range is back.** On Day 2 the Cowrie brute-force trio was 109.160.32.108/.114/.147 (AS198364 Banatsync). Today three *different* IPs from the same range are in the top 10: 109.160.32.31 (5,658), .168 (4,628) and .112 (2,412), and Banatsync is the #2 ASN with 30,978. Same network, new IPs: rotating addresses inside one provider range is a common way to dodge single-IP blocks, so blocking the whole range would work better than blocking individual IPs.

**26. A /24 cluster: 77.239.124.x.** Five IPs from the same subnet are in the top 10 (.155 with 6,004, then .43, .37, .39 and .64 with about 2,510-2,516 each). Four of them have almost identical counts, which suggests one tool splitting its work evenly across several addresses.

**27. Contabo is #1 by network (49,170) but has no IP in the top 10.** The traffic is spread across many Contabo servers, each sending under 2,400 hits. Contabo was also the top RDP source on Day 3 (213.136.90.193). Per-IP top-10 lists can hide this kind of distributed activity, so it's worth checking the ASN table too.

**28. Old Windows exploits still in use.** Suricata flagged "ET EXPLOIT DoublePulsar Backdoor installation communication" 7,918 times. DoublePulsar is the NSA-developed SMB backdoor leaked in 2017 and used alongside EternalBlue in WannaCry. Attackers are still trying it against port 445 nine years later. ATT&CK: T1210 Exploitation of Remote Services.

**Defender takeaway (Day 4):** Never expose VNC to the internet, and never run it without a strong password. Block by network range (ASN or /24), not just single IPs, because attackers rotate addresses within the same provider.
