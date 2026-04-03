# Cloud SIEM Lab — Wazuh on Azure with Automated Threat Response

![Wazuh](https://img.shields.io/badge/Wazuh-4.7.5-00a9e0?style=flat-square)
![Azure](https://img.shields.io/badge/Microsoft_Azure-Student_Credits-0078D4?style=flat-square)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square)
![Status](https://img.shields.io/badge/Lab_Status-Complete-0D9488?style=flat-square)

---

## Overview

This lab documents the deployment of a functional **Security Information and Event Management (SIEM)** system on Microsoft Azure using Wazuh — an open-source security monitoring platform. The lab demonstrates real-time log collection, threat detection, MITRE ATT&CK mapping, and automated active response against a simulated brute-force attack.

This project was built as part of the **CloudGuard GD** concept — a managed cloud security monitoring service for Caribbean SMEs — developed for IT Entrepreneurship (COMP312) at St. George's University.

---

## Objectives

- Deploy a Wazuh all-in-one stack (Manager + Indexer + Dashboard) on Azure
- Enroll a remote Linux endpoint as a monitored Wazuh agent
- Configure automated active response to detect and block SSH brute-force attempts
- Demonstrate real-time threat detection mapped to MITRE ATT&CK T1110.001

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Microsoft Azure                       │
│                                                         │
│  ┌──────────────────────┐    ┌──────────────────────┐  │
│  │   wazuh-server        │    │   wazuh-endpoint      │  │
│  │   Standard_B2s        │    │   Standard_B1s        │  │
│  │   Ubuntu 22.04 LTS    │    │   Ubuntu 22.04 LTS    │  │
│  │   10.0.0.4            │    │   10.1.0.4            │  │
│  │                       │    │                       │  │
│  │  - Wazuh Manager      │◄───│  - Wazuh Agent 4.7.5  │  │
│  │  - Wazuh Indexer      │    │                       │  │
│  │  - Wazuh Dashboard    │    │  Ports: 1514, 1515     │  │
│  │                       │    │                       │  │
│  └──────────────────────┘    └──────────────────────┘  │
│           │                            ▲                │
│      Port 443                          │                │
│           │                    SSH brute-force          │
│           ▼                     simulation              │
│      [ Browser ]              [ Attacker Machine ]      │
└─────────────────────────────────────────────────────────┘
```

---

## Tools & Technologies

| Component | Details |
|---|---|
| Cloud Platform | Microsoft Azure (Azure for Students — $100 credit) |
| SIEM | Wazuh 4.7.5 (Manager, Indexer, Dashboard) |
| OS | Ubuntu 22.04.5 LTS |
| VM — Server | Standard_B2s (2 vCPUs, 4 GB RAM) |
| VM — Endpoint | Standard_B1s (1 vCPU, 1 GB RAM) |
| Detection Rules | Wazuh rules 5710, 5763 |
| Response | Wazuh firewall-drop active response (iptables) |
| MITRE ATT&CK | T1110.001 — Brute Force: Password Guessing |

---

## Lab Walkthrough

### Phase 1 — Provision Azure VMs

Two Ubuntu 22.04 LTS VMs were created in the Azure portal.

**wazuh-server** configuration:
- Size: Standard_B2s (minimum 4 GB RAM required for Wazuh indexer)
- NSG inbound rules: 22 (SSH), 443 (Dashboard), 1514 (Agent events), 1515 (Agent registration)

**wazuh-endpoint** configuration:
- Size: Standard_B1s
- NSG inbound rules: 22 (SSH)

> **Why B2s for the server?** Wazuh runs three services simultaneously — the manager, indexer, and dashboard. The indexer is memory-intensive. Anything below 4 GB RAM causes silent failures or dashboard crashes under load.

---

### Phase 2 — Install Wazuh (All-in-One)

SSH into the server and run the official installer:

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

The `-a` flag deploys all three components on a single node. Installation takes approximately 10–20 minutes. On completion, the installer prints admin credentials:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<ip>:443
    User: admin
    Password: <generated>
INFO: Installation finished.
```

---

### Phase 3 — Access the Dashboard

Navigate to `https://<wazuh-server-public-ip>` in a browser. Accept the self-signed certificate warning and log in with the admin credentials printed by the installer.

**Screenshot 1:** Wazuh dashboard overview — confirm the UI is accessible and the manager is running.

---

### Phase 4 — Enroll the Endpoint Agent

The two VMs were provisioned on separate Azure Virtual Networks (10.0.0.0/24 and 10.1.0.0/24), meaning they had no direct private network path between them.

**Resolution:** The agent was registered and configured to communicate with the manager via its public IP rather than the private IP, with ports 1514 and 1515 open on the server NSG.

On the endpoint VM, generate the install command from the Wazuh dashboard (**Agents → Deploy new agent**) and run it:

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.5-1_amd64.deb \
  && sudo WAZUH_MANAGER='<server-public-ip>' \
  WAZUH_AGENT_NAME='wazuh-endpoint' \
  dpkg -i ./wazuh-agent_4.7.5-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Register the agent with the manager:

```bash
sudo /var/ossec/bin/agent-auth -m <server-public-ip>
```

**Screenshot 2:** Agents page showing wazuh-endpoint with Active (green) status.

---

### Phase 5 — Configure Active Response

Active response allows Wazuh to automatically execute a script when a detection rule fires. The `firewall-drop` command adds an iptables DROP rule on the monitored endpoint, blocking all traffic from the offending IP for a configurable timeout.

On the Wazuh server, add the following block to `/var/ossec/etc/ossec.conf` just before the closing `</ossec_config>` tag:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5710,5763</rules_id>
  <timeout>180</timeout>
</active-response>
```

Restart the manager:

```bash
sudo systemctl restart wazuh-manager
```

**Rule breakdown:**
- **5710** — SSH attempt using a non-existent username (fires on each individual attempt)
- **5763** — Multiple consecutive authentication failures from the same IP (aggregated pattern)
- **timeout: 180** — Block lifted automatically after 3 minutes to prevent permanent lockouts

---

### Phase 6 — Simulate the Attack

From an external machine, simulate a brute-force SSH attack against the endpoint:

```bash
for i in {1..10}; do ssh wronguser@<endpoint-public-ip>; done
```

This fires 10 failed SSH attempts using a non-existent username, triggering rule 5710 on each attempt.

---

## Results

### Detection

Wazuh detected the brute-force attempts within seconds, classifying them under:

| Field | Value |
|---|---|
| Rule ID | 5710 |
| Rule Description | sshd: Attempt to login using a non-existent user |
| MITRE Tactic | Credential Access, Lateral Movement |
| MITRE Technique | T1110.001 — Brute Force: Password Guessing |
| Rule Level | 5 |

**Screenshot 3:** Security Events showing rule 5710 alerts with MITRE ATT&CK tags.

### Automated Response

Within seconds of detection, Wazuh executed the firewall-drop active response on the endpoint, blocking the attacking IP via iptables with zero manual intervention.

**Screenshot 4:** Security Events showing "Host Blocked by firewall-drop Active Response" alert (rule 651).

---

## Key Takeaways

**Network isolation matters.** Azure VMs on separate Virtual Networks cannot communicate over private IPs by default. In a production deployment, VNet Peering would be configured to keep traffic off the public internet.

**Rule hierarchy in Wazuh.** Rule 5763 aggregates repeated 5710 events. Depending on timing and frequency thresholds, individual attempt rules may fire more reliably for active response triggers than their aggregated counterparts.

**Active response is firewall-level.** The `firewall-drop` command doesn't just block SSH — it drops all IP traffic from the attacker at the iptables level. This is a hard block, not an application-level reject.

**Open-source SIEM is viable.** This entire stack — detection, indexing, dashboarding, and automated response — was deployed in under two hours at a cost of approximately $10 USD in cloud credits. Commercial equivalents (Splunk, QRadar, Microsoft Sentinel) cost orders of magnitude more.

---

## Screenshots Index

| # | Description |
|---|---|
| 1 | Azure portal — both VMs running (Virtual Machines list) |
| 2 | Wazuh dashboard overview — manager running, events populating |
| 3 | Agents page — wazuh-endpoint showing Active status |
| 4 | Security Events — rule 5710 alerts with MITRE ATT&CK T1110.001 tags |
| 5 | Security Events — firewall-drop active response alert (rule 651) |
| 6 | Terminal — active response config block in ossec.conf |

---

## Cost

| Resource | Duration | Estimated Cost (USD) |
|---|---|---|
| Standard_B2s (wazuh-server) | Demo period only | ~$8 |
| Standard_B1s (wazuh-endpoint) | Demo period only | ~$2 |
| **Total** | | **~$10 USD** |

Both VMs were deleted immediately after the demonstration to preserve Azure for Students credits.

---

## Author

**Gerron**
Aspiring Cloud Security Engineer
BSc Information Technology — St. George's University (2025–2027)
[LinkedIn](https://www.linkedin.com/in/) | [GitHub](https://github.com/)

---

## References

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/index.html)
- [MITRE ATT&CK T1110.001](https://attack.mitre.org/techniques/T1110/001/)
- [Azure Virtual Network Peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
