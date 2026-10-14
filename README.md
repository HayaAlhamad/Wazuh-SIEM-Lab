
# Wazuh-SIEM-Lab

This repository documents a SIEM home lab I built to get hands-on experience with the defensive side of security — detection, log analysis, and incident triage — to complement the offensive exercises in my other repos.

## Architecture

- **Wazuh server** (Ubuntu Server, VirtualBox): all-in-one Wazuh deployment (indexer, manager, dashboard) acting as the SIEM
- **Monitored endpoint** (Ubuntu 22.04): runs the Wazuh agent, reporting logs back to the server
- **Attacker machine** (Kali Linux): used to generate real attack traffic against the monitored endpoint
- **Network:** all VMs run on an isolated VirtualBox host-only network (`192.168.104.0/24`), separate from NAT-based internet access, so lab traffic never touches my real network


## Build notes

Setting this up involved debugging a fair number of real infrastructure issues, which turned out to be as useful as the detection work itself:

- **Host-only networking** had to be recreated after VirtualBox lost track of the original adapter (`VERR_INTNET_FLT_IF_NOT_FOUND`), which meant reconfiguring every VM's second network adapter afterward
- **Wazuh's install repeatedly failed** with disk-full and OOM-style errors on a 25GB/2GB VM — root cause was simply under-provisioned resources for a SIEM stack; fixed by resizing the virtual disk to 50GB and increasing RAM to 8GB
- **Agent enrollment failed silently** at first — traced to Wazuh's manager ports (1514/1515) being blocked by `ufw`, which only allowed SSH by default
- **Agents intermittently showed "Disconnected"** after VM restarts — resolved by restarting the manager and agent services in sequence after each reboot

## First attack-and-detection cycle

With the lab running, I ran a simple reconnaissance-and-brute-force sequence from Kali against the monitored Ubuntu endpoint, to confirm the whole pipeline — attack → log → detect → alert — actually works end to end.

**What I ran, from Kali:**
```bash
nmap -sV 192.168.104.6
ssh wronguser@192.168.104.6   # repeated with intentionally wrong credentials
```

**What Wazuh detected:**

| Metric | Result |
|---|---|
| Total alerts (24h) | 129 |
| Authentication failures | 6 |
| Authentication successes | 24 |
| MITRE ATT&CK techniques flagged | Brute Force, Password Guessing, SSH, Valid Accounts, Sudo and Sudo Caching, Create Account, Remote Services |

![Wazuh dashboard showing alert counts and MITRE ATT&CK breakdown] ![Wazuh_dashboard.jpg]

### Reading the alerts

Opening the individual events under **Threat Hunting → Events**, filtered to `ssh`, showed each failed login as a distinct alert tied to the source IP (Kali) and the target account (`wronguser`). This confirms:

- Wazuh is correctly parsing `sshd` auth logs from the monitored endpoint
- Repeated failures from the same source are being correctly classified under the **Brute Force** and **Password Guessing** MITRE techniques, not just logged as generic failures
- The successful logins in the same window were my own legitimate SSH sessions to the box, picked up separately and correctly left unflagged as high-severity

### What I'd do next as an analyst

If this were a real alert queue rather than a lab, six failed logins followed by zero lockout wouldn't be severe on its own, but it's worth a closer look given it's paired with a preceding port scan from the same source:

- Correlate the recon scan and the brute-force attempts as one campaign from a single source IP, not two unrelated events
- Check whether any of the attempted logins succeeded
- Block or rate-limit the source IP at the firewall
- Review whether the targeted account (`wronguser`, or any real account) shows signs of being newly created, modified, or granted elevated privileges afterward, given the "Create Account" and "Sudo and Sudo Caching" techniques also appeared in the same window

## Next steps

- [ ] Write custom Wazuh detection rules for scenarios not covered by default rules
- [ ] Add a Windows endpoint with Sysmon for endpoint-level detection
- [ ] Add Suricata for network-layer IDS, correlated with Wazuh's host-based alerts
- [ ] Build a Python tool to enrich alert source IPs via VirusTotal/AbuseIPDB
