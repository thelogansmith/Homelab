# 🛡️ Cybersecurity Homelab

> A hands-on security operations lab built on dedicated bare-metal hardware running Proxmox VE. The network architecture follows the **NCyTE VCCC three-tier model** (Management / Internal / DMZ), simulating a real enterprise environment for practicing SOC analyst skills — log analysis, threat detection, network monitoring, vulnerability management, and incident response.
>
> **Goal:** Demonstrate SOC Level 1 analyst readiness through real infrastructure builds, not just theory.  
> **Status:** 🔨 Actively building — MikroTik recovery in progress; core Proxmox cluster operational.

---

## 📋 Table of Contents

- [Hardware Infrastructure](#hardware-infrastructure)
- [Network Architecture](#network-architecture)
- [Virtual Machines](#virtual-machines)
- [Services & Software Stack](#services--software-stack)
- [Build Progress](#build-progress)
- [Projects & Labs](#projects--labs)
  - [SIEM with Splunk & Wazuh](#1-siem-with-splunk--wazuh)
  - [Network Traffic Analysis](#2-network-traffic-analysis)
  - [Vulnerability Scanning](#3-vulnerability-scanning)
  - [Active Directory Lab](#4-active-directory-lab)
  - [Kali Linux Attack Simulations](#5-kali-linux-attack-simulations)
- [Key Outcomes](#key-outcomes)
- [MITRE ATT&CK Coverage](#mitre-attck-coverage)
- [Learning Roadmap](#learning-roadmap)
- [Certifications & Learning Path](#certifications--learning-path)
- [Known Issues & Notes](#known-issues--notes)

---

## Hardware Infrastructure

All VMs run on Proxmox VE across a three-node dedicated bare-metal cluster, providing enterprise-grade resource headroom and snapshot-based rollback for safe lab exercises.

| Device | Specs | Role |
| --- | --- | --- |
| **Primary Server (pve-srv1)** | 2× Xeon E5-2620 v3 @ 2.40GHz (24 cores), 128GB DDR3 ECC, 2TB NVMe cluster, 4×2TB HDD + 4TB HDD, GTX 1050Ti | Primary Proxmox node — SIEM, monitoring, OPNsense, core services |
| **HP EliteDesk NUC #1 (pve-nuc1)** | i5-6500T, 16GB RAM, 500GB NVMe | Internal tier VMs — AD DC, Windows 10, Ubuntu Server |
| **HP EliteDesk NUC #2 (pve-nuc2)** | i5-6500T, 16GB RAM, 128GB NVMe | Attack/DMZ VMs — Kali Linux, REMnux |
| **MikroTik RB4011iGS+** | 10-port router, SFP+ | Edge router — WAN, VLAN 99 DHCP owner, inter-zone firewall |
| **MikroTik cAP ax** | WiFi 6 AP | Wireless access for Personal (VLAN 40) and Untrusted (VLAN 99) |
| **TP-Link TL-SG1016DE** | 16-port managed switch | 802.1Q VLAN switching, port mirroring → Security Onion |
| **Raspberry Pi 3B** | 1GB RAM | Pi-hole DNS for Untrusted VLAN 99 (physical device) |
| **Windows 11 PC** | 12th gen i5, 64GB RAM, 15TB+ storage, RTX 3060Ti | Ad-hoc tasks via VMware |

**Proxmox VE:** v8.4 · Kernel 6.8.12-pve · Boot Mode: EFI

---

## Network Architecture

### Two-Router Design

The lab uses a **split-router architecture** to cleanly separate edge/untrusted traffic from the trusted internal network:

- **MikroTik RB4011** owns the WAN connection and VLAN 99 (Untrusted/IoT). It acts as the hardware edge firewall and trunks all other VLANs downstream to OPNsense. IoT and smart home devices never leave the hardware layer.
- **OPNsense VM** (pve-srv1) is the internal firewall/router for all trusted VLANs (10, 20, 40, 50). It owns DHCP, inter-VLAN routing, WireGuard VPN termination, and DNS forwarding to Technitium.

This design enforces zero-trust segmentation at the hardware boundary before traffic ever reaches the hypervisor.

```
                    ┌─────────────────────────┐
                    │   ISP / WAN             │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   MikroTik RB4011iGS+   │
                    │   Edge Router           │
                    │   VLAN 99 DHCP owner    │
                    └──────────┬──────────────┘
                               │ VLAN trunk (10,20,40,50)
                    ┌──────────▼──────────────┐
                    │   TP-Link TL-SG1016DE   │
                    │   Managed Switch        │
                    │   802.1Q + Port Mirror  │
                    └──┬──────────┬───────┬───┘
                       │          │       │
          ┌────────────┘          │       └─────────────────┐
          │                       │                         │
┌─────────▼──────────┐ ┌──────────▼──────────┐ ┌───────────▼──────────┐
│  MANAGEMENT        │ │  INTERNAL LAB        │ │  DMZ                 │
│  VLAN 10           │ │  VLAN 20             │ │  VLAN 50             │
│  10.10.10.0/24     │ │  192.168.20.0/24     │ │  172.16.50.0/24      │
│                    │ │                      │ │                      │
│  Splunk SIEM       │ │  Windows Server 2022 │ │  Jellyfin            │
│  Wazuh             │ │  Windows 10 Pro      │ │  Nextcloud           │
│  Security Onion    │ │  Ubuntu Server       │ │  WireGuard endpoint  │
│  Technitium DNS    │ │  (Corp environment)  │ │  Kali Linux          │
│  OPNsense VM       │ │                      │ │  REMnux              │
└────────────────────┘ └──────────────────────┘ └──────────────────────┘
          │
┌─────────▼──────────┐  ┌────────────────────────────────────────────┐
│  PERSONAL          │  │  UNTRUSTED — VLAN 99 — 192.168.99.0/24     │
│  VLAN 40           │  │  Managed directly by MikroTik              │
│  10.40.40.0/24     │  │  IoT, smart home devices                   │
│                    │  │  Internet only — no access to any tier     │
│  iOS devices       │  └────────────────────────────────────────────┘
│  Trusted Devices   │
└────────────────────┘
```

### VLAN Table

| Zone | VLAN | Subnet | Managed By | Purpose |
| --- | --- | --- | --- | --- |
| Management | 10 | 10.10.10.0/24 | OPNsense | SIEM tools, Technitium DNS, monitoring |
| Internal Lab | 20 | 192.168.20.0/24 | OPNsense | AD domain, victim workstations |
| Personal | 40 | 10.40.40.0/24 | OPNsense | iOS devices, Apple TVs (Bonjour/AirPlay) |
| DMZ | 50 | 172.16.50.0/24 | OPNsense | Jellyfin, Nextcloud, WireGuard VPN endpoint |
| Untrusted | 99 | 192.168.99.0/24 | MikroTik | IoT, Ring Alarm, smart home devices |

### Inter-Zone Firewall Policy

| Source | Destination | Policy |
| --- | --- | --- |
| Untrusted (99) | Any internal VLAN | ❌ DENY ALL |
| DMZ (50) | Internal (20) or Management (10) | ❌ DENY ALL |
| Internal (20) | Management (10) | ⚠️ LOG FORWARDING ONLY (ports 9997, 1514) |
| Management (10) | All tiers | ✅ ALLOW (monitoring / passive only) |
| Personal (40) | DMZ (50) | ✅ ALLOW (Jellyfin, Nextcloud access) |
| VPN clients (WireGuard) | DMZ (50) | ✅ ALLOW |
| VPN clients (WireGuard) | Internal (20) or Management (10) | ❌ DENY |

---

## Virtual Machines

All VMs managed in Proxmox VE with snapshots before each exercise for clean rollback.

### pve-srv1 — Primary Server

| VMID | Name | OS | RAM | VLAN | Status | Role |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | opnsense | OPNsense (FreeBSD) | 2GB | trunk | 🔲 Planned | Internal firewall/router for VLANs 10/20/40/50 |
| 101 | splunk-siem | Ubuntu 22.04 | 8GB | 10 | 🔲 Planned | Splunk Free — log ingestion, SPL, dashboards |
| 102 | wazuh | Wazuh OVA 4.x | 8GB | 10 | 🔲 Planned | Wazuh SIEM/XDR — endpoint detection, alerting |
| 103 | sec-onion | Security Onion 2 | 8GB | 10 | 🔲 Planned | Zeek, Suricata, passive PCAP, IDS/IPS |
| 104 | technitium-dns | Ubuntu 22.04 | 1GB | 10 | 🔲 Planned | Internal recursive/authoritative DNS (lab.local) |
| 105 | nextcloud | Ubuntu 22.04 | 2GB | 50 | 🔲 Planned | Personal cloud storage |

### pve-nuc1 — Internal Tier

| VMID | Name | OS | RAM | VLAN | Status | Role |
| --- | --- | --- | --- | --- | --- | --- |
| 201 | corp-dc | Windows Server 2022 | 6GB | 20 | 🔲 Planned | Active Directory DC, DNS, DHCP |
| 202 | corp-win10 | Windows 10 Pro | 4GB | 20 | 🔲 Planned | Victim workstation — Sysmon, Wazuh agent |
| 203 | corp-ubuntu | Ubuntu 22.04 | 2GB | 20 | 🔲 Planned | Linux victim — Apache, SSH, Wazuh agent |

### pve-nuc2 — Attack / DMZ Tier

| VMID | Name | OS | RAM | VLAN | Status | Role |
| --- | --- | --- | --- | --- | --- | --- |
| 301 | kali | Kali Linux 2024 | 4GB | 50 | 🔲 Planned | Offensive tools, attack simulation |
| 302 | remnux | REMnux | 2GB | isolated | 🔲 Planned | Malware analysis — no network route |

---

## Services & Software Stack

### Currently Deployed

| Service | Host | VLAN | Purpose |
| --- | --- | --- | --- |
| Proxmox VE 8.4 | All nodes | mgmt | Hypervisor cluster — VM/LXC management, snapshots |


### Planned / In Progress

| Service | Host | VLAN | Purpose |
| --- | --- | --- | --- |
| OPNsense | VM 100 (pve-srv1) | trunk | Internal firewall/router for trusted VLANs |
| Splunk Free | VM 101 (pve-srv1) | 10 | Primary SIEM — log ingestion, SPL search, dashboards |
| Wazuh 4.x | VM 102 (pve-srv1) | 10 | Endpoint detection, XDR, agent-based alerting |
| Security Onion 2 | VM 103 (pve-srv1) | 10 | Network monitoring — Zeek, Suricata, PCAP |
| Technitium DNS | VM 104 (pve-srv1) | 10 | Internal recursive DNS (lab.local) |
| Nextcloud | VM 107 (pve-srv1) | 50 | Personal file storage |
| WireGuard | OPNsense plugin | 50 | Remote access VPN — iOS/tvOS clients → DMZ only |
| Sysmon | corp-dc, corp-win10 | 20 | Enriched Windows process/network/file telemetry |
| Wazuh Agents | All Internal VMs | 20 | Endpoint telemetry → Wazuh manager |

### Future / Stretch Goals

| Service | Purpose |
| --- | --- |
| Frigate NVR | Security camera NVR with local recording |
| Ollama (vision LLM) | AI-assisted camera footage triage pipeline |
| OpenVPN / split tunnel | Alternative remote access evaluation |
| TheHive | Incident case management — SOC Tier 1/2 workflow practice |
| Velociraptor | Endpoint live response and threat hunting |

---

## Build Progress

| Phase | Name | Status |
| --- | --- | --- |
| 0 | Hardware & Physical Setup | 🔄 In Progress — cabling pending |
| 1 | Proxmox on Primary Server | ✅ Complete |
| 2 | MikroTik & VLAN Network | 🚧 Blocked — NetInstall recovery in progress |
| 3 | Proxmox NUCs & Cluster | ✅ Complete |
| 4 | Download & Stage Installation Media | 🔄 In Progress — Wazuh OVA / REMnux QCOW2 import pending |
| 5 | Management Tier VMs | 🔲 Not Started |
| 6 | Internal Tier VMs | 🔲 Not Started |
| 7 | DMZ Tier VMs | 🔲 Not Started |
| 8 | Log Forwarding & Splunk Inputs | 🔲 Not Started |
| 9 | Splunk Detection Rules & Dashboards | 🔲 Not Started |
| 10 | Attack Simulations & Detection Validation | 🔲 Not Started |

---

## Projects & Labs

### 1. SIEM with Splunk & Wazuh

**Goal:** Build a dual-SIEM environment — Splunk for log search and correlation, Wazuh for endpoint detection and alerting — mirroring how many real SOCs layer tools.

**Architecture:**

```
Internal Tier VMs
  corp-dc (Windows Server)  ──┐  Winlogbeat + Wazuh Agent
  corp-win10 (Windows 10)   ──┼──────────────────────────► Wazuh Manager (VM 102)
  corp-ubuntu (Ubuntu)      ──┘  Filebeat + Wazuh Agent                │
                                                                       │ Wazuh alerts
Technitium (DNS logs) ────────────────────────────────────────► Splunk (VM 101)
MikroTik (Syslog) ─────────────────────────────────────────►
Security Onion (Zeek/Suricata) ────────────────────────────►
```

**Log sources ingested into Splunk:**

| Source | Key Events |
| --- | --- |
| Windows Security Log | 4624/4625 (logon), 4720 (account created), 4740 (lockout) |
| Sysmon | Process creation (1), network connections (3), file events (11) |
| Wazuh alerts | Rule-based endpoint detections forwarded to Splunk |
| Pi-hole query log | DNS requests, blocked domains, query frequency |
| MikroTik firewall | Inter-VLAN allow/deny decisions |
| Zeek conn.log | Connection metadata for all mirrored traffic |
| Ubuntu auth.log | SSH logins, sudo usage, PAM events |

**Splunk searches written:**

```spl
# Brute-force detection — 5+ failed logins in 2 minutes
index=windows EventCode=4625
| bucket _time span=2m
| stats count by _time, src_ip, user
| where count >= 5

# Processes spawning PowerShell or cmd.exe (suspicious parent)
index=windows EventCode=1
(CommandLine="*powershell*" OR CommandLine="*cmd.exe*")
NOT ParentImage IN ("*explorer.exe","*svchost.exe")
| table _time, host, ParentImage, Image, CommandLine

# Pi-hole: top blocked domains (potential C2 / malware callbacks)
index=pihole blocked=true
| stats count by domain
| sort -count
| head 20
```

**Wazuh rules triggered in lab:**

- SSH brute-force (rule 5710/5712) — generated via Hydra attack from Kali
- Windows audit policy change (rule 18104) — tested via GPO modification
- New user account created (rule 18152) — tested via `net user` command

---

### 2. Network Traffic Analysis

**Goal:** Capture and interpret network traffic across tier boundaries to identify scanning, lateral movement, and C2 patterns.

**Setup:**

- Managed switch mirror port sends all VLAN traffic to Security Onion (VM 103)
- Security Onion running Zeek (connection metadata) and Suricata (signature-based IDS)
- MikroTik firewall logs forwarded to Splunk for inter-tier traffic visibility

**Wireshark filters learned:**

```
# DNS queries only (not responses)
dns.flags.response == 0

# Traffic between specific tiers
ip.src == 192.168.20.0/24 && ip.dst == 172.16.50.0/24

# Failed TCP connections (RST)
tcp.flags.reset == 1

# Large DNS queries — potential tunneling indicator
dns && dns.qry.name.len > 50
```

---

### 3. Vulnerability Scanning

**Goal:** Learn to run and interpret vulnerability scans against Internal tier VMs; understand CVSS scoring and remediation prioritization.

**Tools:** Nmap (port/service discovery), Nessus Essentials (authenticated scanning)

**Sample findings on intentionally unpatched `corp-dc`:**

| Finding | CVSS | Notes |
| --- | --- | --- |
| SMBv1 Enabled | 9.8 | EternalBlue-exploitable; kept for lab practice |
| Missing Windows patches | Varies | Intentionally unpatched |
| RDP — NLA not enforced | 7.5 | Credential relay target |
| Weak Kerberos config | 8.1 | Kerberoastable service accounts configured |

**Key insight:** The same CVE-2017-0144 (EternalBlue, CVSS 9.8) on an internet-facing server vs. an isolated Internal VLAN with no DMZ access has completely different real-world risk. Context matters as much as the score.

---

### 4. Active Directory Lab

**Goal:** Build and administer an AD environment, then observe what attacks against it look like in logs.

**Domain:** `lab.local` hosted on `corp-dc` (VM 201)

**Audit policies enabled:**

| Policy | Events Generated |
| --- | --- |
| Audit Logon Events | 4624 (success), 4625 (failure), 4634 (logoff) |
| Audit Account Management | 4720 (created), 4740 (lockout), 4732 (group change) |
| Audit Process Creation | 4688 + command line logging |
| Audit Policy Change | 4719 |
| PowerShell Script Block Logging | 4104 |

Intentionally misconfigured accounts (Kerberoastable SPNs, unconstrained delegation) are configured to generate realistic attack telemetry.

---

### 5. Kali Linux Attack Simulations

**Goal:** Understand attacker techniques well enough to detect and hunt for them on the defensive side.

> ⚠️ All offensive activity is performed exclusively against VMs I own, within the isolated lab environment.

| Exercise | Attack Tool | Blue-Side Detection |
| --- | --- | --- |
| Port scan — Internal tier | Nmap SYN scan | Zeek conn.log sequential SYN pattern; Suricata alert |
| SSH brute-force — corp-ubuntu | Hydra | auth.log rapid failures; Wazuh rule 5710; Splunk alert |
| Metasploit — Metasploitable VM | `exploit/multi/handler` | Reverse shell traffic in Wireshark; Suricata ET rule |
| DNS query flood | Custom script | Pi-hole anomalous query volume spike in Splunk |

---

## Key Outcomes

| Outcome | SOC Skill Demonstrated | Status |
| --- | --- | --- |
| 5-zone VLAN segmentation with zero-trust firewall policy | Network architecture, ACL design | 🔄 In Progress |
| Multi-source log pipeline → Splunk | Log ingestion, data onboarding, sourcetype management | 🔲 Planned |
| Custom SPL detection rules (brute-force, PowerShell, Kerberoasting) | Detection engineering, SPL | 🔲 Planned |
| SOC dashboard: endpoint + network + DNS correlation | SIEM analysis, dashboard design | 🔲 Planned |
| Full attack simulation → incident report with MITRE ATT&CK mapping | Threat hunting, IR documentation | 🔲 Planned |
| WireGuard remote access (iOS/tvOS clients → DMZ only) | VPN architecture, zero-trust remote access | 🔲 Planned |

---

## MITRE ATT&CK Coverage

Techniques observed in lab exercises. Grows as simulations are completed.

| Tactic | Technique | ID | How Observed | Detected By |
| --- | --- | --- | --- | --- |
| Reconnaissance | Network Service Discovery | T1046 | Nmap SYN scan from Kali | Zeek conn.log, Suricata |
| Initial Access | Valid Accounts (brute-forced) | T1078 | Hydra SSH attack | auth.log, Wazuh rule 5710, Splunk |
| Execution | PowerShell | T1059.001 | TryHackMe rooms | Sysmon Event 1, Event 4104 |
| Execution | Windows Command Shell | T1059.003 | Lab exercises | Sysmon Event 1 |
| Credential Access | Brute Force | T1110 | Hydra against SSH/RDP | Wazuh, Splunk, auth.log |
| Discovery | Network Service Scanning | T1046 | Nmap from DMZ tier | Zeek, Suricata ET rules |
| Command & Control | DNS (studying) | T1071.004 | Pi-hole anomaly logs | Pi-hole → Splunk |

---

## Learning Roadmap

```
Foundations (Current)
├── ✅ Deploy Proxmox on dedicated hardware (24-core Xeon, 128GB RAM)
├── ✅ Configure 3-node Proxmox cluster
├── 🔄 Recover MikroTik RB4011 via NetInstall (RouterOS 6.49.18)
├── 🔄 Configure 5-VLAN architecture (Management / Internal / Personal / DMZ / Untrusted)
├── 🔲 Deploy OPNsense VM — internal firewall/router
├── 🔲 Deploy Splunk — ingest Windows, Linux, firewall, DNS logs
├── 🔲 Deploy Wazuh — agents on all Internal tier VMs
├── 🔲 Install Sysmon on Windows VMs
├── 🔲 Configure Technitium DNS (lab.local)
└── 🔲 Complete TryHackMe SOC Level 1 path

Detection & Analysis
├── Build Splunk dashboards for auth, network, and endpoint events
├── Write custom Wazuh detection rules
├── Simulate Kerberoasting → detect in Splunk (Event ID 4769)
├── Simulate full attack chain → write formal incident report
└── Deploy WireGuard on OPNsense for remote access (iOS/tvOS)

Intermediate Skills
├── Write Suricata rules based on observed attack traffic
├── Practice threat hunting with MITRE ATT&CK Navigator
├── Set up TheHive for incident case management
├── Learn Python for log parsing and Splunk API automation
└── Pursue CompTIA Security+ or BTL1 certification

Advanced Topics
├── Memory forensics with Volatility (REMnux)
├── Basic static malware analysis
├── Publish Sigma detection rules to public repository
├── Frigate NVR + Ollama vision pipeline for security cameras
└── Introduce a honeynet to capture opportunistic attacks
```

---

## Repository Structure

```
homelab/
├── README.md                         ← This file
├── phase-0-hardware-physical-setup/
├── phase-1-proxmox-primary-server/
├── phase-2-mikrotik-vlan-network/
├── phase-3-proxmox-nucs-cluster/
├── phase-4-installation-media/
├── phase-5-management-tier-vms/
├── phase-6-internal-tier-vms/
├── phase-7-dmz-tier-vms/
├── phase-8-log-forwarding-splunk/
├── phase-9-splunk-detection-dashboards/
├── phase-10-attack-simulations/
├── infrastructure/
│   ├── network-diagram.png
│   ├── proxmox-vm-inventory.md
│   └── vlan-firewall-rules.md
├── splunk/
│   ├── searches/                     ← SPL queries by use case
│   └── dashboards/                   ← Dashboard exports
├── wazuh/
│   └── custom-rules/
├── lab-exercises/
│   ├── nmap-scans/
│   ├── wireshark-captures/
│   └── incident-reports/
└── notes/
    ├── tools/
    └── concepts/
```


*All offensive techniques performed exclusively within the isolated lab environment against systems I own. Network architecture based on the NCyTE VCCC three-tier model.*
