# Security Homelab

Segmented virtual enterprise lab for practicing attack, detection, and hardening. Built in VMware Workstation as a self-directed project.

**Status:** In progress (started November 24, 2026)

## Goals
- Build a segmented network with a firewall separating attacker, internal, and DMZ zones
- Run common attacks from Kali against machines I built myself
- Detect them in Wazuh and write custom rules mapped to MITRE ATT&CK
- Harden the systems and measure the difference by rerunning the attacks

## Architecture

![Network diagram](docs/network-diagram.svg)

| Zone | Network | Machines |
|------|---------|----------|
| Attacker | 10.10.10.0/24 | Kali Linux |
| Internal | 10.10.20.0/24 | Ubuntu Server 24.04 (ubuntu-srv), Wazuh (SIEM, custom detection rules) |
| DMZ | 10.10.30.0/24 | dmz-web (nginx) |

The three lab zones are isolated VMware networks. Only the OPNsense WAN connects out, through VMware NAT.

## Tools
VMware Workstation, OPNsense, Ubuntu Server 24.04, Kali Linux, Wazuh, Nmap, Hydra

## Results

**Segmentation tests:** firewall rules were tested from Kali and ubuntu-srv, and all six tests passed. See [docs/segmentation-tests.md](docs/segmentation-tests.md).

**Attack and detection tests:** three attacks tested against ubuntu-srv and dmz-web (Nmap scan, SSH brute force, web vulnerability scan). Two were detected entirely by Wazuh's default rules with full MITRE ATT&CK mapping; the third (network scanning) required a custom decoder and correlation rule, also built and documented. See [docs/attack-detection-tests.md](docs/attack-detection-tests.md).

**Attack and detection results** (in progress):

| Attack | Technique (MITRE ATT&CK) | Detected before hardening? | Detected after? |
|--------|--------------------------|----------------------------|-----------------|
| Nmap scan | [T1046] | [ ] | [ ] |
| SSH brute force | [T1110] | [ ] | [ ] |
| [add more] | | | |

**Summary:** The three lab zones are isolated VMware networks. Only the OPNsense WAN connects out, through VMware NAT.

## Repo layout

- `docs/segmentation-tests.md` — firewall rule tests and results (Wednesday)
- `docs/attack-detection-tests.md` — attack simulations, Wazuh detections, 
  and the custom detection rule (Friday/Saturday)
- `docs/evidence/` — supporting screenshots for both test files
- `docs/network-diagram.svg` — current network diagram

## Challenges and lessons learned

- **Dnsmasq interface list:** ubuntu-srv got no DHCP lease on the Internal segment. Cause: Dnsmasq's interface list only included ATTACKER and DMZ. Fix: added INTERNAL to the interface list under Services > Dnsmasq DNS & DHCP > General.

- **DHCP reservations need a reboot:** after adding a host reservation in OPNsense (binding a VM's MAC to a fixed address outside the dynamic pool), running `networkctl renew` on the VM did not pick up the new address. A full reboot did. Reservations should be added before building a VM, or expect to reboot after adding one.

- **Ubuntu installer ISO errors look worse than they are:** after the Wazuh VM's install finished and the "Reboot Now" prompt appeared, leaving the installer ISO connected through the reboot printed a wall of `SQUASHFS error: Failed to read block` messages. The install had already completed; the errors were the installer losing access to its own ISO during shutdown, not a failed install. Disconnecting the CD/DVD drive (Connect at power on unticked) before rebooting avoids the scare.

- **Wazuh agent version should match the manager:** Wazuh recommends keeping agents on the same or an older version than the manager, since a newer agent can lose compatibility. After installing the agent on ubuntu-srv, I pinned its version with `apt-mark hold wazuh-agent` so a routine `apt upgrade` can't silently update it out of sync with the server.

- **Host-only routing only covers directly-connected segments:** the laptop can reach Internal (VMnet3) directly since it has a host adapter there, but has no route to DMZ (VMnet4) since that adapter is intentionally disconnected for isolation. Pings and SSH to dmz-web from the laptop failed with no firewall log entry at all, confirming the packets never left the laptop. Rather than add a static route (which would undermine the isolation), management of DMZ/Attacker hosts is done from Kali instead, which already has firewall-permitted access.

- **Custom Wazuh decoders need explicit parent relationships:** building a custom decoder for ufw firewall logs initially failed silently, Wazuh kept matching events against its built-in generic `kernel` decoder instead of mine, with no error shown. The fix was declaring my decoder as a child (`<parent>kernel</parent>`) rather than competing with the built-in one on the same `program_name`. A second false alarm came from checking in the wrong place: `wazuh-logtest` confirmed the rule logic worked, but live traffic produced no visible alerts until I enabled `<logall>yes</logall>` on the manager and confirmed via `archives.log` that events were in fact arriving correctly the whole time.
  
## Safety
This lab is isolated and only attacks machines I built. No credentials, keys, or real IPs are committed to this repo.
