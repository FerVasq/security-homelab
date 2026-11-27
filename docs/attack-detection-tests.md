# Attack and Detection Tests

Tested from Kali (Attacker, 10.10.10.144) against ubuntu-srv and dmz-web, 
with Wazuh monitoring both as agents.

## Test 1: Nmap scan against ubuntu-srv

**Command:** `sudo nmap -sV -A 10.10.20.10`

**Result:** Scan completed, only port 22/tcp (SSH) showed open; all 999 other 
ports reported as filtered (no response).

**Wazuh detection:** 0 alerts generated (searched `agent.name: ubuntu-srv` 
in Discover, 0 hits).

**Finding: Firewall-level prevention vs. host-level detection**

Checking OPNsense's Live View log confirmed why: the "Block attacker to internal" 
rule (written and tested in segmentation-tests.md) blocked every probed port in 
real time, dozens of distinct destination ports all logged as blocked from 
10.10.10.144 to 10.10.20.10. Because the traffic never reached ubuntu-srv, 
neither its local ufw logs nor Wazuh's host-based monitoring saw most of the 
scan, only the one ICMP packet type explicitly allowed through (echo/timestamp 
requests) appeared in ufw's log.

This demonstrates that network-layer segmentation can make host-based detection 
largely redundant for already-blocked traffic, while still leaving a clear audit 
trail at the firewall itself. Wazuh's host-based model (HIDS) relies on logs 
generated on the monitored machine; traffic stopped upstream by the firewall 
never produces those logs.

**Evidence:**

![OPNsense blocking the Nmap scan port by port](evidence/opnsense-nmap-scan-blocked.png)

## Test 2: SSH brute force against ubuntu-srv (Hydra)

**Command:** `hydra -l internal -P /tmp/passlist.txt ssh://10.10.20.10`

**Result:** 5 login attempts, 0 valid passwords found (custom wordlist used 
intentionally to avoid risk of an accidental successful login).

**Wazuh detection:** 10 alerts generated (one per failed attempt, plus 
related events), rule description `sshd: authentication failed`, rule level 5.

**MITRE ATT&CK mapping (automatic, built-in Wazuh rule):**
- T1110.001 — Password Guessing
- T1021.004 — Remote Services: SSH
- Tactics: Credential Access, Lateral Movement

Unlike the Nmap scan, SSH brute force traffic is allowed through the firewall 
(by design, since ubuntu-srv's only exposed service to the Attacker zone is SSH), 
so it reaches the host and generates real auth.log entries. Wazuh's default 
`sshd` decoder and rule set detected every failed attempt in real time and 
automatically mapped it to the correct MITRE techniques, no custom rule was 
needed for this detection.

**Evidence:**

![Wazuh detecting the SSH brute force attempt](evidence/wazuh-ssh-bruteforce-detected.png)

![MITRE ATT&CK mapping for the brute force alert](evidence/wazuh-ssh-bruteforce-mitre.png)

## Test 3: Web scan against dmz-web (Nikto)

**Command:** `nikto -h http://10.10.30.10`

**Result:** 8,167 requests made in 20 seconds. Findings included the nginx 
version (1.24.0, outdated vs. current 1.31.3) and 8 missing security headers.

**Wazuh detection:** 8,039 alerts generated, parsed directly from nginx's 
access log via Wazuh's built-in web-accesslog decoder, no additional 
configuration needed.

Checking `rule.description` breakdown across the scan window revealed at 
least 7 distinct rule types firing, each mapped to a different MITRE 
technique:

| Rule description | MITRE ID(s) | Level |
|---|---|---|
| Multiple web server 400 error codes from same source ip | T1595.002 | 10 |
| Suspicious URL access | T1055 | 6 |
| Multiple XSS attempts from same source ip | T1059 | 10 |
| XSS (Cross Site Scripting) attempt | T1059.007 | 6 |
| Common web attack | T1055, T1083, T1190 | 6 |
| SQL injection attempt | T1190 | 7 |
| PHP CGI-bin vulnerability attempt | T1210 | 6 |

**Finding:** Wazuh's default web ruleset is far more granular than expected, 
it doesn't just flag "a scan happened", it distinguishes reconnaissance 
(repeated 400 errors → T1595.002 Active Scanning) from actual exploitation 
attempts (SQLi → T1190, XSS → T1059.007, PHP CGI vulnerabilities → T1210), 
assigning appropriate severity and MITRE context to each category 
automatically, no custom rules required.

**Evidence:**

![Wazuh detecting the Nikto web scan via nginx access log](evidence/wazuh-web-scan-detected.png)

![Breakdown of distinct rule descriptions fired during the scan](evidence/wazuh-web-scan-rule-breakdown.png)
