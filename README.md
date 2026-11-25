# Security Homelab

Segmented virtual enterprise lab for practicing attack, detection, and hardening. Built in VMware Workstation as a self-directed project.

**Status:** In progress (started [date])

## Goals
- Build a segmented network with a firewall separating attacker, internal, and DMZ zones
- Run common attacks from Kali against machines I built myself
- Detect them in Wazuh and write custom rules mapped to MITRE ATT&CK
- Harden the systems and measure the difference by rerunning the attacks

## Architecture

| Zone | Network | Machines |
|------|---------|----------|
| Attacker | [subnet] | Kali Linux |
| Internal | [subnet] | Ubuntu Server 24.04, Wazuh |
| DMZ | [subnet] | [web server] |

All networks are VMware host-only and isolated from my home network.

## Tools
VMware Workstation, OPNsense, Ubuntu Server 24.04, Kali Linux, Wazuh, Nmap, Hydra

## Results

| Attack | Technique (MITRE ATT&CK) | Detected before hardening? | Detected after? |
|--------|--------------------------|----------------------------|-----------------|
| Nmap scan | [T1046] | [ ] | [ ] |
| SSH brute force | [T1110] | [ ] | [ ] |
| [add more] | | | |

**Summary:** [One or two sentences with your real numbers once you have them.]

## Repo layout
- `docs/` build notes, one file per step
- `detections/` custom Wazuh rules
- `attacks/` what I ran and what the logs showed
- `configs/` sanitized configs only

## Challenges and lessons learned
[Fill this in as you go: what broke, how you fixed it, what you'd do differently.]

## Safety
This lab is isolated and only attacks machines I built. No credentials, keys, or real IPs are committed to this repo.
