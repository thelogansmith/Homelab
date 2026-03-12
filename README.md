# 🛡️ Cybersecurity Homelab

> A hands-on security operations lab built on dedicated bare-metal hardware running Proxmox VE. The network architecture follows the **NCyTE VCCC three-tier model** (Management / Internal / DMZ), simulating a real enterprise environment for practicing SOC analyst skills — log analysis, threat detection, network monitoring, vulnerability management, and incident response.
>
> **Status:** 🔨 Actively building — documenting progress as I learn.

---

## 📋 Table of Contents

- [Hardware Infrastructure](#hardware-infrastructure)
- [Network Architecture](#network-architecture)
- [Virtual Machines](#virtual-machines)
- [Skills in Progress](#skills-in-progress)
- [Projects & Labs](#projects--labs)
  - [SIEM with Splunk & Wazuh](#1-siem-with-splunk--wazuh)
  - [Network Traffic Analysis](#2-network-traffic-analysis)
  - [Vulnerability Scanning](#3-vulnerability-scanning)
  - [Active Directory Lab](#4-active-directory-lab)
  - [Kali Linux Attack Simulations](#5-kali-linux-attack-simulations)
- [CTF & Platform Writeups](#ctf--platform-writeups)
- [Tools & Technologies](#tools--technologies)
- [Learning Roadmap](#learning-roadmap)
- [Certifications & Learning Path](#certifications--learning-path)
- [MITRE ATT&CK Coverage](#mitre-attck-coverage)

---

## Hardware Infrastructure

All VMs run on Proxmox VE across dedicated bare-metal hardware, providing enterprise-grade resource headroom and snapshot-based rollback for safe lab exercises.

| Device | CPU | RAM | Storage | Role |
|---|---|---|---|---|
| **Primary Server** | 2× Intel Xeon E5-2620 v3 @ 2.40GHz (24 cores) | 125.81 GB | 1.23 TB | Primary Proxmox VE host — SIEM, monitoring, core services |
| **HP EliteDesk NUC #1** | [CPU] | [RAM] | [Storage] | Proxmox node — Internal tier VMs |
| **HP EliteDesk NUC #2** | [CPU] | [RAM] | [Storage] | Proxmox node — Attack & DMZ VMs |
| **HP EliteDesk NUC #3** | [CPU] | [RAM] | [Storage] | Proxmox node — spare / future expansion |
| **MikroTik Router** | — | — | — | Core routing, VLAN trunking, inter-tier firewall rules |
| **Dedicated Switch** | — | — | — | 802.1Q VLAN segmentation, port mirroring for PCAP |

**Proxmox VE:** v8.4.14 · Kernel 6.8.12-15-pve · Boot Mode: EFI · Uptime: 18+ days

---

## Network Architecture

The lab implements the **NCyTE Virtual Cybersecurity Career Challenge (VCCC) three-tier network model**, segmented into Management, Internal, and DMZ tiers. A fourth isolated VLAN handles untrusted devices (IoT, personal devices). All inter-tier traffic is controlled by MikroTik firewall rules. The managed switch mirrors traffic from all VLANs to the monitoring host for passive packet capture.

```
                         ┌──────────────────────┐
                         │    MikroTik Router   │
                         │  (VLAN Trunk + FW)   │
                         └──────────┬───────────┘
                                    │
                         ┌──────────┴───────────┐
                         │    Managed Switch    │
                         │  (802.1Q + Mirror)   │
                         └──┬───────┬───────┬───┘
                            │       │       │
           ┌────────────────┘       │       └──────────────────┐
           │                        │                          │
┌──────────▼─────────┐  ┌───────────▼──────────┐  ┌───────────▼──────────┐
│   TIER 1           │  │   TIER 2             │  │   TIER 3             │
│   MANAGEMENT       │  │   INTERNAL           │  │   DMZ                │
│   VLAN 10          │  │   VLAN 20            │  │   VLAN 30            │
│   10.10.10.0/24    │  │   192.168.20.0/24    │  │   172.16.30.0/24     │
│                    │  │                      │  │                      │
│   Splunk SIEM      │  │   Windows 10 Client  │  │   Kali Linux         │
│   Wazuh            │  │   Windows Server     │  │   (Attack Sim)       │
│   Security Onion   │  │   Ubuntu Server      │  │                      │
│   Pi-hole          │  │   (Corp environment) │  │   Internet-facing    │
│                    │  │                      │  │   services (future)  │
└────────────────────┘  └──────────────────────┘  └──────────────────────┘
                                                            │
                                              ┌─────────────▼────────────┐
                                              │   VLAN 99 — UNTRUSTED    │
                                              │   192.168.99.0/24        │
                                              │                          │
                                              │   IoT devices, smart TVs │
                                              │   Personal devices       │
                                              │   Internet only — no     │
                                              │   access to other tiers  │
                                              └──────────────────────────┘
```

### Tier Definitions & Firewall Policy

| Tier | VLAN | Subnet | Description | Allowed To Reach |
|---|---|---|---|---|
| **Management** | 10 | `10.10.10.0/24` | Security tools, monitoring, SIEM | All tiers (read-only/monitor) |
| **Internal** | 20 | `192.168.20.0/24` | Simulated corporate environment | Management (log forwarding only) |
| **DMZ** | 30 | `172.16.30.0/24` | Attack simulation, internet-exposed services | Internet only; blocked from Internal & Management |
| **Untrusted** | 99 | `192.168.99.0/24` | IoT, personal devices, smart TVs | Internet only; fully isolated |

**Key firewall rules enforced by MikroTik:**
- Untrusted → any internal tier: **DENY ALL**
- DMZ → Internal or Management: **DENY ALL**
- Internal → Management: **LOG FORWARDING ONLY** (ports 9997, 1514)
- Management → all tiers: **ALLOW** (monitoring/passive only)

---

## Virtual Machines

All VMs managed in Proxmox VE with snapshots before each exercise for clean rollback. VM IDs follow a structured numbering scheme by tier.

### Tier 1 — Management (VLAN 10)

| VM ID | Name | OS | RAM | Role |
|---|---|---|---|---|
| 101 | `splunk-siem` | Ubuntu 22.04 | 8 GB | Splunk Free — log ingestion, search, dashboards |
| 102 | `wazuh` | OVA (Wazuh 4.x) | 8 GB | Wazuh SIEM/XDR — endpoint detection, alerts |
| 103 | `sec-onion` | Security Onion 2 | 8 GB | IDS/IPS, Zeek, Suricata, PCAP |
| 104 | `pihole` | Debian (Pi-hole) | 1 GB | DNS filtering, query logging, threat intel blocklists |

### Tier 2 — Internal (VLAN 20)

| VM ID | Name | OS | RAM | Role |
|---|---|---|---|---|
| 201 | `corp-dc` | Windows Server 2022 | 6 GB | Active Directory DC, DNS, DHCP |
| 202 | `corp-win10` | Windows 10 Pro | 4 GB | Victim workstation — Sysmon, Wazuh agent |
| 203 | `corp-ubuntu` | Ubuntu 22.04 | 2 GB | Linux victim — Apache, SSH, Wazuh agent |

### Tier 3 — DMZ (VLAN 30)

| VM ID | Name | OS | RAM | Role |
|---|---|---|---|---|
| 301 | `kali` | Kali Linux 2024 | 4 GB | Offensive tools, attack simulation |
| 302 | `remnux` | REMnux | 2 GB | Isolated malware analysis (no network route) |

---

## Skills in Progress

> Documenting my learning journey as a complete beginner. Each item reflects real hands-on time in the lab.

| Skill | Tool(s) | Status | Notes |
|---|---|---|---|
| Log analysis & SIEM | Splunk, Wazuh | 🔄 Learning | Splunk Fundamentals 1 complete; Wazuh agents deployed |
| Endpoint detection | Wazuh agents | 🔄 Learning | Agents installed on Internal tier VMs |
| Packet analysis | Wireshark, tcpdump | 🔄 Learning | Analyzing PCAPs from TryHackMe labs |
| Intrusion detection | Suricata, Snort | 🔄 Learning | Running via Security Onion |
| Vulnerability scanning | Nmap, Nessus Essentials | 🔄 Learning | Scanning Internal tier VMs |
| Hypervisor management | Proxmox VE | 🔄 Learning | VM creation, snapshots, VLAN config |
| Windows administration | Windows Server 2022 | 🔄 Learning | AD domain, GPOs, audit policies |
| Network segmentation | MikroTik, 802.1Q | ✅ Functional | 4-VLAN architecture deployed and routing correctly |
| DNS security | Pi-hole | ✅ Functional | Blocking malicious domains; query logs forwarded to Splunk |
| Linux CLI | Bash | 🔄 Learning | Comfortable with navigation, grep, pipes, log parsing |

---

## Projects & Labs

### 1. SIEM with Splunk & Wazuh

**Goal:** Build a dual-SIEM environment — Splunk for log search and correlation, Wazuh for endpoint detection and alerting — mirroring how many real SOCs layer tools.

**Architecture:**

```
Internal Tier VMs
  corp-dc (Windows Server)  ──┐  Winlogbeat + Wazuh Agent
  corp-win10 (Windows 10)   ──┼─────────────────────────────► Wazuh Manager (VM 102)
  corp-ubuntu (Ubuntu)      ──┘  Filebeat + Wazuh Agent               │
                                                                        │ Wazuh alerts
Pi-hole (DNS logs) ─────────────────────────────────────────► Splunk (VM 101)
MikroTik (Syslog) ──────────────────────────────────────────►
Security Onion (Zeek/Suricata) ─────────────────────────────►
```

**Log sources ingested into Splunk:**

| Source | Key Events |
|---|---|
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

**Next steps:** Build a Splunk dashboard correlating Wazuh endpoint alerts with network events from Zeek; write first custom Wazuh rule.

---

### 2. Network Traffic Analysis

**Goal:** Capture and interpret network traffic across tier boundaries to identify scanning, lateral movement, and C2 patterns.

**Setup:**
- Managed switch mirror port sends all VLAN traffic to Security Onion (VM 103)
- Security Onion running Zeek (connection metadata) and Suricata (signature-based IDS)
- MikroTik firewall logs forwarded to Splunk for inter-tier traffic visibility

**What I've practiced:**
- Capturing traffic with `tcpdump` on the Security Onion monitoring interface
- Identifying an Nmap SYN scan in Wireshark — sequential ports, SYN-only packets, no handshake completion
- Reading Zeek `conn.log` fields: `id.orig_h`, `id.resp_p`, `proto`, `conn_state`, `duration`
- Correlating Pi-hole blocked queries with connection attempts in Zeek

**Wireshark filters learned:**

```
# DNS queries only (not responses)
dns.flags.response == 0

# Traffic between specific tiers
ip.src == 192.168.20.0/24 && ip.dst == 172.16.30.0/24

# HTTP GET requests
http.request.method == "GET"

# Failed TCP connections (RST)
tcp.flags.reset == 1

# Large DNS queries — potential tunneling indicator
dns && dns.qry.name.len > 50
```

**Next steps:** Analyze a C2 beacon PCAP from TryHackMe; write a Suricata rule based on observed traffic patterns.

---

### 3. Vulnerability Scanning

**Goal:** Learn to run and interpret vulnerability scans against Internal tier VMs; understand CVSS scoring and remediation prioritization.

**Tools:** Nmap (port/service discovery), Nessus Essentials (authenticated scanning)

**Scanning workflow:**

```bash
# Phase 1: Discover live hosts in Internal tier
nmap -sn 192.168.20.0/24

# Phase 2: Service and version enumeration
nmap -sV -sC -p- 192.168.20.0/24 -oN internal-scan.txt

# Phase 3: Vulnerability scripts
nmap --script vuln 192.168.20.10

# Phase 4: Authenticated Nessus scan
# (Configured via Nessus UI with domain credentials)
```

**Sample findings on intentionally unpatched `corp-dc`:**

| Finding | CVSS | Notes |
|---|---|---|
| SMBv1 Enabled | 9.8 | EternalBlue-exploitable; kept for lab practice |
| Missing Windows patches | Varies | Intentionally unpatched |
| RDP — NLA not enforced | 7.5 | Easy credential relay target |
| Weak Kerberos config | 8.1 | Kerberoastable service accounts configured |

**Key insight so far:** The same CVE-2017-0144 (EternalBlue, CVSS 9.8) on an internet-facing server vs. an isolated Internal VLAN with no DMZ access has completely different real-world risk. Context matters as much as the score.

**Next steps:** Document full remediation steps for each finding; re-scan after patching to verify.

---

### 4. Active Directory Lab

**Goal:** Build and administer an AD environment, then observe what attacks against it look like in logs.

**Domain:** `lab.local` hosted on `corp-dc` (VM 201)

**What I've configured:**
- Domain controller with DNS and DHCP
- `corp-win10` domain-joined
- Organizational Units: IT, Finance, HR (simulating business structure)
- User accounts with varying privilege levels
- Group Policy Objects: audit policy, PowerShell logging, AppLocker (in progress)
- Intentionally misconfigured accounts for practice: Kerberoastable SPN, unconstrained delegation

**Audit policies enabled:**

| Policy | Events Generated |
|---|---|
| Audit Logon Events | 4624 (success), 4625 (failure), 4634 (logoff) |
| Audit Account Management | 4720 (created), 4740 (lockout), 4732 (group change) |
| Audit Process Creation | 4688 + command line logging |
| Audit Policy Change | 4719 |
| PowerShell Script Block Logging | 4104 |

**Why this matters for SOC work:** Active Directory is present in the vast majority of enterprise environments. The misconfigured accounts (Kerberoastable SPNs, unconstrained delegation) are intentional — the goal is to generate realistic attack telemetry and then find it in Splunk and Wazuh.

**Next steps:** Simulate a Kerberoasting attack from Kali, then hunt for it in Splunk using Event ID 4769.

---

### 5. Kali Linux Attack Simulations

**Goal:** Understand attacker techniques well enough to detect and hunt for them on the defensive side.

> ⚠️ All offensive activity is performed exclusively against VMs I own, within the lab environment. The DMZ VLAN has no route to Internal or Management tiers — Kali cannot reach production systems.

**Tools used:** Nmap, Metasploit, Hydra, Netcat, CrackMapExec (learning)

**Exercises completed:**

| Exercise | Attack Tool | What I Found on the Blue Side |
|---|---|---|
| Port scan — Internal tier | Nmap SYN scan | Zeek conn.log: high volume SYN to sequential ports; Suricata alert fired |
| SSH brute-force — corp-ubuntu | Hydra | auth.log: rapid 4xx failures; Wazuh rule 5710 triggered; Splunk alert |
| Metasploit — Metasploitable VM | `exploit/multi/handler` | Observed reverse shell traffic in Wireshark; Suricata ET rule fired |
| DNS query flood | custom script | Pi-hole logs: anomalous query volume; Splunk dashboard spike |

**Key insight:** Running the attack and then finding it in the logs is the most effective learning method. After running a Hydra brute-force and seeing exactly which log lines it generates, writing the Splunk detection query becomes intuitive rather than theoretical.

**Next steps:** Chain scan → exploit → persistence against a lab VM, then write a complete incident report documenting the full attack timeline from the defensive perspective.

---

## CTF & Platform Writeups

TryHackMe, Hack The Box, and other CTF writeups are tracked in a dedicated repository:

> 📁 **[github.com/thelogansmith/CTFs](https://github.com/thelogansmith/CTFs)**

That repo contains room writeups, challenge solutions, and platform progress. Labs completed there that involve tools deployed in this homelab (Splunk, Wazuh, Wireshark, Nmap) are cross-referenced in the exercises above.

---

## Tools & Technologies

### Currently Deployed

| Tool | Tier | Purpose |
|---|---|---|
| Proxmox VE 8.4 | Infrastructure | Hypervisor, VM management, snapshots |
| Splunk Free | Management | Log ingestion, SPL search, dashboards, alerting |
| Wazuh 4.x | Management | Endpoint detection, SIEM/XDR, agent-based alerting |
| Security Onion 2 | Management | Zeek, Suricata, passive PCAP, IDS/IPS |
| Pi-hole | Management | DNS filtering, query logging, blocklist-based threat intel |
| Sysmon | Internal | Enriched Windows process/network/file telemetry |
| Wazuh Agents | Internal | Endpoint telemetry forwarded to Wazuh manager |
| Wireshark / tcpdump | Management | Manual packet inspection |
| Nmap | DMZ/Management | Port scanning, service enumeration |
| Nessus Essentials | Management | Authenticated vulnerability scanning |
| Kali Linux | DMZ | Attack simulation only — no Internal/Management access |
| MikroTik RouterOS | Infrastructure | VLAN trunking, inter-tier firewall, syslog forwarding |

### On the Learning List

| Tool | Why It Matters for SOC |
|---|---|
| Sigma Rules | Vendor-agnostic detection rules — portable across SIEM platforms |
| MITRE ATT&CK Navigator | Maps lab exercises to industry-standard TTP framework |
| TheHive | Incident case management — common in Tier 1/2 SOC workflows |
| Velociraptor | Endpoint live response and threat hunting |
| Python | Log parsing, IOC extraction, SIEM API automation |

---

## Learning Roadmap

```
Phase 1 — Foundations (Current)
├── ✅ Deploy Proxmox on dedicated hardware (24-core Xeon, 125 GB RAM)
├── ✅ Configure 4-VLAN architecture (Management / Internal / DMZ / Untrusted)
├── ✅ Deploy Splunk — ingest Windows, Linux, firewall, DNS logs
├── ✅ Deploy Wazuh — agents on all Internal tier VMs
├── ✅ Install Sysmon on Windows VMs
├── ✅ Configure Pi-hole with query logging → Splunk
├── ✅ Isolate IoT / personal devices to Untrusted VLAN
├── 🔄 Complete TryHackMe SOC Level 1 path
└── 🔄 Write first Splunk alert rules

Phase 2 — Detection & Analysis
├── Build Splunk dashboards for auth, network, and endpoint events
├── Write custom Wazuh detection rules
├── Simulate Kerberoasting → detect in Splunk (Event ID 4769)
├── Simulate full attack chain → write formal incident report
└── Analyze C2 beacon PCAP from TryHackMe challenge

Phase 3 — Intermediate Skills
├── Write Suricata rules based on observed attack traffic
├── Practice threat hunting with MITRE ATT&CK Navigator
├── Set up TheHive for incident case management
├── Learn Python for log parsing and Splunk API automation
└── Pursue CompTIA Security+ or BTL1 certification

Phase 4 — Advanced Topics
├── Memory forensics with Volatility (REMnux)
├── Basic static malware analysis (FlareVM)
├── Publish Sigma detection rules to public repository
└── Introduce a honeynet to capture and analyze opportunistic attacks
```

---

## MITRE ATT&CK Coverage

Techniques observed in lab exercises. Grows as simulations are completed.

| Tactic | Technique | ID | How Observed | Detected By |
|---|---|---|---|---|
| Reconnaissance | Network Service Discovery | T1046 | Nmap SYN scan from Kali | Zeek conn.log, Suricata |
| Initial Access | Valid Accounts (brute-forced) | T1078 | Hydra SSH attack | auth.log, Wazuh rule 5710, Splunk |
| Execution | PowerShell | T1059.001 | TryHackMe rooms | Sysmon Event 1, Event 4104 |
| Execution | Windows Command Shell | T1059.003 | Lab exercises | Sysmon Event 1 |
| Credential Access | Brute Force | T1110 | Hydra against SSH/RDP | Wazuh, Splunk, auth.log |
| Discovery | Network Service Scanning | T1046 | Nmap from DMZ tier | Zeek, Suricata ET rules |
| Command & Control | DNS (studying) | T1071.004 | Pi-hole anomaly logs | Pi-hole → Splunk |

---

## Repository Structure

```
homelab/
├── README.md                        ← This file
├── infrastructure/
│   ├── network-diagram.png
│   ├── proxmox-vm-inventory.md
│   ├── vlan-firewall-rules.md       ← MikroTik rule documentation
│   └── sysmon-config.xml
├── splunk/
│   ├── searches/                    ← SPL queries by use case
│   └── dashboards/                  ← Dashboard exports
├── wazuh/
│   └── custom-rules/                ← Local rule overrides
├── lab-exercises/
│   ├── nmap-scans/                  ← Saved scan output files
│   ├── wireshark-captures/          ← Annotated PCAPs
│   └── incident-reports/            ← Post-exercise IR writeups
└── notes/
    ├── tools/                       ← Notes per tool as learned
    └── concepts/                    ← Networking, AD, protocols
```

---

*Actively learning and building. All offensive techniques performed exclusively within the isolated lab environment against systems I own. Network architecture based on the NCyTE VCCC three-tier model.*
