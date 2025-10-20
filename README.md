# SOC Homelab — Segmentation, Hierarchy, Tools, and Networks

## Source

This Homelab is based on the Pratik-IT SOC Homelab:

[https://pratik-it.com/soc-homelab-concevoir-un-centre-operationnel-de-securite-en-environnement-virtualise/](https://pratik-it.com/soc-homelab-concevoir-un-centre-operationnel-de-securite-en-environnement-virtualise/)

## Goal

This lab is designed to enable simultaneous blue team and red team exercises in a realistic, segmented environment. The objective is to practice detection, response, and mitigation on the defensive side while providing realistic offensive scenarios and targets for attack simulation.

## Checklist

### Priority tasks (current focus)

| Task                                    | Description                                                        | Status      | Notes                                                                                                         |                                         
| --------------------------------------- | ------------------------------------------------------------------ | ----------- | ------------------------------------------------------------------------------------------------------------- | 
| Validate nested virtualization          | Confirm `nested-hw-virt` is enabled and test KVM within Proxmox VM | In progress | `VBoxManage modifyvm "Proxmox" --nested-hw-virt on`; verify with `egrep -c "(vmx\|svm)" /proc/cpuinfo`and`kvm-ok`or`dmesg` logs`  |
| Create Proxmox bridges                  | Create `vmbr0`–`vmbr4` and assign to network interfaces            | To do       | Use `/etc/network/interfaces` or Proxmox GUI; ensure VirtualBox network mode supports multiple bridges        |                                           
| Deploy perimeter and internal firewalls | Deploy FortiGate (WAN/DMZ) and pfSense (DMZ/Intermediate/LAN)      | To do       | Configure minimal rules: deny-by-default, allow HTTP/HTTPS to DVWA and SSH to Cowrie; enable Snort on pfSense |                                              
| Deploy SOC core and logging pipeline    | Deploy Wazuh and Elastic Stack, verify agents can forward logs     | To do       | Start with co-located Wazuh+ELK VM; verify Syslog/Filebeat/Wazuh agent ingestion                              |                                             

### Completed and existing

* Proxmox installed inside VirtualBox
* Wazuh manager deployed and receiving local logs

### Additional essential items

* Backup strategy and snapshot schedule
* Baseline images for quick restore
* Access control for management interfaces (VPN/MFA)

## Computer Specifications

CPU: AMD Ryzen 9 5950x

RAM: Corsair LPX 64 GB DDR4-3600 CL18

GPU: AMD Radeon RX6950XT

MOBO: MSI Tomahawk B450

## Architecture Overview

The SOC Homelab is divided into isolated network zones connected through virtual bridges in Proxmox. FortiGate manages perimeter control, pfSense handles internal segmentation, and the SOC centralizes log collection and incident response.

## Network Zones

| Zone             | Bridge | Subnet          | Description                            |
| ---------------- | ------ | --------------- | -------------------------------------- |
| WAN              | vmbr0  | 10.0.2.0/24     | Simulated Internet (VirtualBox NAT)    |
| SOC              | vmbr4  | 192.168.11.0/24 | Security Operations Center             |
| DMZ              | vmbr1  | 192.168.14.0/24 | Exposed services and honeypots         |
| Intermediate     | vmbr2  | 192.168.20.0/24 | Internal firewall and inspection layer |
| Active Directory | vmbr3  | 192.168.30.0/24 | Internal domain and clients            |

---

## Machine List and Roles

| Machine                           | Zone         | Role                                    |
| --------------------------------- | ------------ | --------------------------------------- |
| FortiGate (NGFW)                  | Perimeter    | Firewall, NAT, L7 filtering             |
| pfSense + Snort                   | Intermediate | IDS/IPS, VPN, internal firewall         |
| Wazuh                             | SOC          | SIEM, log collection                    |
| Elastic Stack (ELK)               | SOC          | Data analytics, anomaly detection       |
| Splunk                            | SOC          | Visualization and dashboards (optional) |
| Shuffle                           | SOC          | SOAR and incident automation            |
| DVWA                              | DMZ          | Vulnerable web app (attack simulation)  |
| ModSecurity                       | DMZ          | Web Application Firewall                |
| Cowrie                            | DMZ          | SSH honeypot                            |
| Active Directory (Windows Server) | AD           | Authentication and GPO management       |
| Windows Client                    | AD           | Domain user workstation                 |

---

## Network Interfaces

### Bridge Assignments

| Bridge | Purpose                |
| ------ | ---------------------- |
| vmbr0  | WAN (VirtualBox NAT)   |
| vmbr1  | DMZ                    |
| vmbr2  | Intermediate / pfSense |
| vmbr3  | AD / LAN               |
| vmbr4  | SOC                    |

### Example Interface Mapping

| VM             | Interface                                | Bridge |
| -------------- | ---------------------------------------- | ------ |
| FortiGate      | eth0 → vmbr0, eth1 → vmbr1               |        |
| pfSense        | eth0 → vmbr1, eth1 → vmbr2, eth2 → vmbr3 |        |
| Wazuh          | eth0 → vmbr4                             |        |
| DVWA           | eth0 → vmbr1                             |        |
| Cowrie         | eth0 → vmbr1                             |        |
| AD DC          | eth0 → vmbr3                             |        |
| Windows Client | eth0 → vmbr3                             |        |

---

## Static IP Plan

| Zone         | Host               | IP            |
| ------------ | ------------------ | ------------- |
| SOC          | Wazuh              | 192.168.11.10 |
| SOC          | ELK                | 192.168.11.11 |
| SOC          | Shuffle            | 192.168.11.12 |
| DMZ          | FortiGate (DMZ IF) | 192.168.14.1  |
| DMZ          | DVWA               | 192.168.14.10 |
| DMZ          | ModSecurity        | 192.168.14.11 |
| DMZ          | Cowrie             | 192.168.14.20 |
| Intermediate | pfSense            | 192.168.20.1  |
| AD           | Domain Controller  | 192.168.30.10 |
| AD           | Windows Client     | 192.168.30.21 |

---

## Firewall and Network Rules

### WAN → DMZ (FortiGate)

* Allow HTTP/HTTPS to DVWA (192.168.14.10)
* Allow SSH to Cowrie (192.168.14.20)
* Block all other traffic

### DMZ → SOC

* Allow Syslog (UDP/TCP 514)
* Allow Filebeat (TCP 5044)
* Allow Wazuh Agent → Manager (1514/1515)

### SOC → AD

* Allow secure log collection (Winlogbeat, Wazuh Agent)

### pfSense ↔ SOC

* Snort alerts → Wazuh
* Shuffle API → pfSense (block automation)

### AD → Internet

* No direct access (through pfSense → FortiGate only)

---

## Log Pipeline

1. Agents (Wazuh/Filebeat) send logs to Wazuh Manager.
2. Wazuh forwards data to Elastic Stack (ELK).
3. Splunk (optional) collects forwarded events.
4. Shuffle monitors alerts and triggers automated responses.

---

## Test Scenarios

| # | Scenario                      | Objective                               |
| - | ----------------------------- | --------------------------------------- |
| 1 | SSH Brute Force (Cowrie)      | Detect and block attacker IP            |
| 2 | SQL Injection (DVWA)          | Detect via ModSecurity, alert via Wazuh |
| 3 | AD Authentication Brute Force | Detect failed logins via Sysmon/Winlog  |
| 4 | Suspicious Process on Client  | Detect new process and isolate VM       |

---

## Recommended VM Resources

| VM             | vCPU | RAM (GB) | Disk (GB) |
| -------------- | ---- | -------- | --------- |
| FortiGate      | 2    | 2        | 10        |
| pfSense        | 2    | 4        | 20        |
| Wazuh + ELK    | 4    | 16       | 200       |
| Splunk         | 4    | 12       | 100       |
| DVWA (LXC)     | 1    | 2        | 8         |
| ModSecurity    | 1    | 2        | 8         |
| Cowrie (LXC)   | 1    | 1        | 8         |
| AD DC          | 2    | 4        | 40        |
| Windows Client | 2    | 4        | 40        |

---

## Deployment Steps

1. Create bridges `vmbr0`–`vmbr4` in Proxmox.
2. Deploy FortiGate and configure WAN/DMZ.
3. Deploy pfSense and configure DMZ/Intermediate/LAN.
4. Deploy SOC (Wazuh + ELK).
5. Deploy DMZ services and agents.
6. Deploy Active Directory and a Windows client.
7. Verify log flow and dashboards.
8. Deploy Shuffle and connect it to Wazuh + pfSense.
9. Run attack simulations.

---

## Backup and Snapshots

* Take snapshots before major changes.
* Export key VMs regularly using `vzdump`.
* Keep rollback points for dashboards and playbooks.

---

## Automation

* Ansible for Wazuh agent and service deployment.
* Terraform (Proxmox Provider) for VM provisioning.

---

## Nested Virtualization Notes

* Enable `nested-hw-virt` on the VirtualBox VM running Proxmox.
* Monitor CPU and RAM usage.
* Test network routing carefully with VirtualBox NAT and bridges.

---

## Pre-Test Checklist

* [ ] Wazuh Manager receives agent logs
* [ ] Snort sends alerts to Wazuh
* [ ] ModSecurity logs HTTP events
* [ ] Cowrie logs SSH attempts
* [ ] Shuffle has pfSense API access

---

## Reference Ports

| Service  | Port        |
| -------- | ----------- |
| Wazuh    | 1514 / 1515 |
| Syslog   | 514         |
| Filebeat | 5044        |
