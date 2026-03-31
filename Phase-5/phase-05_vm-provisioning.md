# Phase 5 — VM Provisioning & OS Installation
**GitHub Dir:** phase-5-vm-provisioning

**Objective:** Create all VMs across the Proxmox cluster, install base operating systems, confirm correct VLAN assignment, and verify basic network connectivity via ping test. No application configuration, agent enrollment, or service setup occurs in this phase — that is handled in subsequent phases.

**Definition of done for each VM:**
- VM created in Proxmox with specified CPU, RAM, and disk specs
- OS installed and booted successfully
- Network interface confirmed on correct VLAN bridge
- IP address assigned (DHCP from OPNsense for trusted VLANs)
- Ping to VLAN default gateway passes ✅

**Prerequisites:**
- Phase 4 complete — all ISOs staged in Proxmox ISO storage, QCOW2/OVA files present on target host
- Phase 2 complete — MikroTik VLAN trunking operational
- OPNsense VM (VM 100) deployed and serving DHCP for VLANs 10, 20, 40, 50 *(VM 100 is the first task in this phase for this reason — it is the gateway dependency for all other VMs)*

> ⚠️ **Firewall rules are NOT configured in this phase.** Inter-VLAN communication may be open during provisioning — this is expected and intentional. Firewall policy is enforced in a later phase. Ping tests in this phase target only the VLAN default gateway, not cross-VLAN hosts.

---

## Proxmox Node Assignments

| Node     | VMs Provisioned in This Phase                                                                                            |
| -------- | ------------------------------------------------------------------------------------------------------------------------ |
| pve-srv1 | VM 100 (OPNsense), VM 101 (Splunk), VM 102 (Wazuh), VM 103 (Security Onion), VM 104 (Technitium DNS), VM 107 (Nextcloud) |
| pve-nuc1 | VM 201 (corp-dc), VM 202 (corp-win10), VM 203 (corp-ubuntu)                                                              |
| pve-nuc2 | VM 301 (kali), VM 302 (REMnux)                                                                                           |

---

## Tasks

### pve-srv1 — Core Infrastructure

---

#### VM 100 — OPNsense (Internal Firewall/Router)
*Deploy this first. All other VMs depend on OPNsense for DHCP and gateway connectivity.*

- [ ] **Create and install VM 100: OPNsense**
  - Skills: FreeBSD-based OS installation, Proxmox VM creation, multi-interface NIC assignment (WAN + LAN trunk), OPNsense initial setup wizard, VLAN interface configuration, DHCP server activation per VLAN
  - Specs:
    - CPU: 2 cores
    - RAM: 2GB
    - Disk: 32GB
    - NICs: 2 (WAN-facing to MikroTik trunk; LAN-facing as VLAN trunk for VLANs 10/20/40/50)
    - ISO: OPNsense
  - VLAN: Trunk (manages VLANs 10, 20, 40, 50)
  - Gateway IP: This VM *is* the gateway — assign static IPs per VLAN interface (e.g. 10.10.10.1, 192.168.20.1, 10.40.40.1, 172.16.50.1)
  - Verification: OPNsense web UI reachable from Management VLAN; DHCP leases issuing on all trusted VLANs

---

#### VM 101 — Splunk SIEM

- [ ] **Create and install VM 101: splunk-siem**
  - Skills: Ubuntu Server installation, static hostname configuration, Proxmox VLAN bridge assignment
  - Specs:
    - CPU: 4 cores
    - RAM: 8GB
    - Disk: 100GB (Splunk indexes grow; allocate generously)
    - ISO: Ubuntu Server 24.04
  - VLAN: 10 (Management) — `10.10.10.0/24`
  - Expected IP: DHCP from OPNsense (or reserve static lease)
  - Ping test: `ping 10.10.10.1` (OPNsense VLAN 10 interface) ✅
  - Note: Splunk application installation and index configuration happen in Phase 8.

---

#### VM 102 — Wazuh

- [ ] **Import and configure VM 102: wazuh**
  - Skills: OVA import into Proxmox (`qm importovf` or `qemu-img convert` VMDK → QCOW2), VM hardware adjustment post-import (RAM, network adapter VMXNET3 → VirtIO), boot order verification
  - Specs (post-import adjustment):
    - CPU: 4 cores
    - RAM: 8GB
    - Disk: as shipped in OVA (~50GB)
    - Source: Wazuh OVA (pre-built all-in-one appliance)
  - VLAN: 10 (Management) — `10.10.10.0/24`
  - Expected IP: DHCP from OPNsense (or reserve static lease)
  - Ping test: `ping 10.10.10.1` (OPNsense VLAN 10 interface) ✅
  - Note: Wazuh manager configuration and agent enrollment happen in Phase 8.

---

#### VM 103 — Security Onion

- [ ] **Create and install VM 103: sec-onion**
  - Skills: Security Onion installation wizard, monitoring interface setup (promiscuous mode, no IP), management interface configuration, standalone deployment mode selection
  - Specs:
    - CPU: 4 cores
    - RAM: 8GB
    - Disk: 200GB (PCAP and log storage)
    - NICs: 2 (management interface on VLAN 10; monitoring interface connected to switch mirror port — no IP)
    - ISO: Security Onion 2
  - VLAN: 10 (Management) — `10.10.10.0/24` (management NIC only)
  - Expected IP: DHCP or static on management interface
  - Ping test: `ping 10.10.10.1` from management interface ✅
  - Note: Zeek/Suricata configuration and log forwarding to Splunk happen in Phase 8.

---

#### VM 104 — Technitium DNS

- [ ] **Create and install VM 104: technitium-dns**
  - Skills: Ubuntu Server installation, .NET runtime on Linux, Technitium DNS web installer, service verification (`systemctl status dns.service`), distinguishing recursive/authoritative DNS from filtering-only DNS (Pi-hole)
  - Specs:
    - CPU: 1 core
    - RAM: 1GB
    - Disk: 16GB
    - ISO: Ubuntu Server 24.04
  - VLAN: 10 (Management) — `10.10.10.0/24`
  - Expected IP: Static recommended (DNS servers should not rely on DHCP)
  - Install Technitium after OS install:
    ```bash
    curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
    ```
  - Verify service running: `systemctl status dns.service`
  - Ping test: `ping 10.10.10.1` (OPNsense VLAN 10 interface) ✅
  - Note: DNS zone configuration, forwarder rules, and lab.local records are configured in a later phase.

---

#### VM 107 — Nextcloud

- [ ] **Create and install VM 107: nextcloud**
  - Skills: Ubuntu Server installation, LAMP stack awareness (Apache/MySQL/PHP), Proxmox VLAN bridge assignment
  - Specs:
    - CPU: 2 cores
    - RAM: 2GB
    - Disk: 64GB
    - ISO: Ubuntu Server 24.04
  - VLAN: 50 (DMZ) — `172.16.50.0/24`
  - Expected IP: DHCP from OPNsense VLAN 50
  - Ping test: `ping 172.16.50.1` (OPNsense VLAN 50 interface) ✅
  - Note: Nextcloud application installation and storage configuration happen in a later phase.

---

### pve-nuc1 — Internal Tier

---

#### VM 201 — corp-dc (Windows Server 2022)

- [ ] **Create and install VM 201: corp-dc**
  - Skills: Windows Server 2022 installation, VirtIO driver installation during setup (load driver from VirtIO ISO), initial Windows configuration (hostname, static IP), Proxmox SPICE/VNC console usage for Windows installer
  - Specs:
    - CPU: 4 cores
    - RAM: 6GB
    - Disk: 80GB
    - ISOs: Windows Server 2022 + VirtIO Drivers (attach both)
  - VLAN: 20 (Internal Lab) — `192.168.20.0/24`
  - Expected IP: DHCP from OPNsense VLAN 20 (set static reservation after deployment)
  - Ping test: `ping 192.168.20.1` (OPNsense VLAN 20 interface) ✅
  - Note: Active Directory promotion, DNS, DHCP, GPOs, and Sysmon installation happen in Phase 6.

---

#### VM 202 — corp-win10 (Windows 10 Pro)

- [ ] **Create and install VM 202: corp-win10**
  - Skills: Windows 10 installation, VirtIO driver loading, local account setup (avoid Microsoft account requirement during install), hostname configuration
  - Specs:
    - CPU: 2 cores
    - RAM: 4GB
    - Disk: 60GB
    - ISOs: Windows 10 Pro + VirtIO Drivers (attach both)
  - VLAN: 20 (Internal Lab) — `192.168.20.0/24`
  - Expected IP: DHCP from OPNsense VLAN 20
  - Ping test: `ping 192.168.20.1` (OPNsense VLAN 20 interface) ✅
  - Note: Domain join, Sysmon installation, and Wazuh agent deployment happen in Phase 6.

---

#### VM 203 — corp-ubuntu (Ubuntu Server 24.04)

- [ ] **Create and install VM 203: corp-ubuntu**
  - Skills: Ubuntu Server installation, OpenSSH server setup during install, hostname and static IP configuration
  - Specs:
    - CPU: 2 cores
    - RAM: 2GB
    - Disk: 32GB
    - ISO: Ubuntu Server 24.04
  - VLAN: 20 (Internal Lab) — `192.168.20.0/24`
  - Expected IP: DHCP from OPNsense VLAN 20
  - Ping test: `ping 192.168.20.1` (OPNsense VLAN 20 interface) ✅
  - Note: Apache, SSH hardening, Filebeat, and Wazuh agent deployment happen in Phase 6.

---

### pve-nuc2 — Attack / DMZ Tier

---

#### VM 301 — Kali Linux

- [ ] **Create and install VM 301: kali**
  - Skills: Kali Linux installation, offensive distro awareness, VLAN isolation verification
  - Specs:
    - CPU: 2 cores
    - RAM: 4GB
    - Disk: 60GB
    - ISO: Kali Linux 2024
  - VLAN: 50 (DMZ) — `172.16.50.0/24`
  - Expected IP: DHCP from OPNsense VLAN 50
  - Ping test: `ping 172.16.50.1` (OPNsense VLAN 50 interface) ✅
  - Note: Attack simulations and tool configuration happen in Phase 10.

---

#### VM 302 — REMnux

- [ ] **Import and configure VM 302: remnux**
  - Skills: QCOW2 disk import (`qm importdisk`), VM shell creation without installer media, VirtIO disk attachment, boot order configuration, SPICE display setup, network isolation verification (no IP / isolated bridge)
  - Specs (configure after import):
    - CPU: 2 cores
    - RAM: 2GB
    - Disk: As shipped in QCOW2
    - Source: REMnux QCOW2
  - VLAN: **Isolated / no network** — REMnux has no route to any VLAN by design (malware analysis environment)
  - Verification: VM boots successfully to REMnux desktop; confirm no network interface is assigned ✅
  - Note: REMnux has no ping test — network isolation is the verified state.

---

## Verification Summary

Complete this table as each VM is provisioned. All VMs must pass before proceeding to Phase 6.

| VMID | Name | Node | VLAN | Gateway | Ping Pass | OS Confirmed |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | opnsense | pve-srv1 | trunk | N/A (is gateway) | N/A | 🔲 |
| 101 | splunk-siem | pve-srv1 | 10 | 10.10.10.1 | 🔲 | 🔲 |
| 102 | wazuh | pve-srv1 | 10 | 10.10.10.1 | 🔲 | 🔲 |
| 103 | sec-onion | pve-srv1 | 10 | 10.10.10.1 | 🔲 | 🔲 |
| 104 | technitium-dns | pve-srv1 | 10 | 10.10.10.1 | 🔲 | 🔲 |
| 107 | nextcloud | pve-srv1 | 50 | 172.16.50.1 | 🔲 | 🔲 |
| 201 | corp-dc | pve-nuc1 | 20 | 192.168.20.1 | 🔲 | 🔲 |
| 202 | corp-win10 | pve-nuc1 | 20 | 192.168.20.1 | 🔲 | 🔲 |
| 203 | corp-ubuntu | pve-nuc1 | 20 | 192.168.20.1 | 🔲 | 🔲 |
| 301 | kali | pve-nuc2 | 50 | 172.16.50.1 | 🔲 | 🔲 |
| 302 | remnux | pve-nuc2 | isolated | N/A | N/A — isolated | 🔲 |

---

## Notes
<!-- configs, issues, decisions -->

Made note of the MAC addresses associated with the NIC interfaces in the proxmox console.
Changed the interfaces in the shell via proxmox
Changed the default WAN IP address to DHCP, automatically set to 192.168.0.34
Changed the default LAN IP address to 10.10.10.1
Entered the OPNsense shell and disabled the packet filter to access the OPNsense webgui from the 192.168.0.X network.
Used the installation wizard to setup the WAN (OPNsense trunk) and the LAN (VLAN10)
Configured the remaining interfaces (VLANs) in OPNsense
Enable DHCP on all four interfaces

### OPNsense — Initial Configuration
**VM:** 100 (opnsense) on pve-srv1  
**Phase:** 5 — VM Provisioning & OS Installation  
**Scope:** Interface assignment, VLAN routing, and DHCP configuration. Firewall rules are NOT configured in this document — that is handled in a later phase.

#### NIC Architecture — Per-Bridge Design

OPNsense VM 100 was provisioned with **five dedicated NICs**, each mapped to a separate Proxmox bridge rather than a single trunk interface with VLAN sub-interfaces. This is a deliberate design choice: each VLAN gets its own virtual wire directly into OPNsense, which simplifies troubleshooting and avoids tagged frame handling inside the VM.

| Device | Proxmox Bridge | Role |
|--------|---------------|------|
| vtnet0 | vmbr0 | WAN (upstream to AIO router / MikroTik) |
| vtnet1 | vmbr10 | VLAN 10 — Management (LAN) |
| vtnet2 | vmbr20 | VLAN 20 — Internal Lab |
| vtnet3 | vmbr40 | VLAN 40 — Personal |
| vtnet4 | vmbr50 | VLAN 50 — DMZ |

> 📷 **Photo:** Proxmox VM 100 hardware view showing all five network devices with their bridge assignments and MAC addresses.

Because each VLAN has a dedicated NIC, **no VLAN sub-interfaces are required inside OPNsense.** vtnet2/3/4 are assigned directly as OPNsense interfaces — not as tagged children of a parent trunk.

#### Setup Wizard

The OPNsense setup wizard was completed via the web UI accessed from the temporary AIO router's network (`192.168.0.x`). The packet filter was temporarily disabled via the OPNsense shell (`pfctl -d`) to allow web UI access across the WAN interface before firewall rules were established.

##### General Information

| Field | Value |
|-------|-------|
| Hostname | OPNsense |
| Domain | lab.local |
| Language | English |
| Timezone | Etc/UTC |
| DNS Servers | 127.0.0.1 (Unbound resolver, localhost) |
| Override DNS | ✅ Enabled |
| Enable Resolver (Unbound) | ✅ Enabled |
| Enable DNSSEC | ☐ Disabled |

> 📷 **Photo:** OPNsense wizard — General Information tab showing hostname, domain, and DNS settings.

DNS is pointed at `127.0.0.1` to use OPNsense's built-in Unbound resolver for now. This will be updated to forward to Technitium DNS (VM 104, `10.10.10.x`) once that VM is provisioned in Phase 5 and configured in a later phase.

##### Network — WAN

| Field | Value |
|-------|-------|
| Type | DHCP |
| Block RFC1918 Private Networks | ✅ Enabled |
| Block Bogon Networks | ✅ Enabled |
| DHCP Hostname | DHCP.OPNsense |

> 📷 **Photo:** OPNsense wizard — Network [WAN] tab showing DHCP type and default policies.

WAN is set to DHCP. During this temporary configuration, the upstream device is an AIO router (not the MikroTik RB4011, which is pending NetInstall recovery). The WAN interface received a lease from the AIO router's pool. When the MikroTik is restored and placed back in the path, OPNsense will obtain a lease from MikroTik instead — no WAN reconfiguration required since both use DHCP.

Block RFC1918 and Block Bogon are both enabled. This is correct for a WAN interface facing an upstream NAT device.

##### Network — LAN

| Field | Value |
|-------|-------|
| IP Address | 10.10.10.1/24 |
| Configure DHCP Server | ✅ Enabled |

> 📷 **Photo:** OPNsense wizard — Network [LAN] tab showing 10.10.10.1/24 and DHCP enabled.

The LAN interface maps to vtnet1 (vmbr10) and serves as the VLAN 10 Management gateway. The wizard assigns an allow-all firewall rule to LAN by default — this is the only interface with a default allow rule. All OPT interfaces (VLAN 20/40/50) default to block-all until rules are explicitly added in a later phase.

#### Interface Assignment

After the wizard, vtnet2/3/4 were unassigned. They were added and named via **Interfaces → Assignments**.

> 📷 **Photo:** Interfaces → Assignments showing all five interfaces assigned: VLAN10MGMT (lan/vtnet1), VLAN20INTERNAL (opt1/vtnet2), VLAN40PERSONAL (opt2/vtnet3), VLAN50DMZ (opt3/vtnet4), WAN (wan/vtnet0).

| Interface | Identifier | Device | Description |
|-----------|-----------|--------|-------------|
| [VLAN10MGMT] | lan | vtnet1 | Management — LAN (wizard-configured) |
| [VLAN20INTERNAL] | opt1 | vtnet2 | Internal Lab |
| [VLAN40PERSONAL] | opt2 | vtnet3 | Personal |
| [VLAN50DMZ] | opt3 | vtnet4 | DMZ |
| [WAN] | wan | vtnet0 | WAN upstream |

#### Interface Configuration

Each OPT interface was enabled and assigned a static IPv4 address via **Interfaces → [interface name]**.

##### VLAN10MGMT (LAN)
Configured by the setup wizard. No changes required post-wizard.

| Field | Value |
|-------|-------|
| IPv4 Address | 10.10.10.1/24 |

##### VLAN20INTERNAL (OPT1)

| Field | Value |
|-------|-------|
| Enable | ✅ |
| IPv4 Config Type | Static IPv4 |
| IPv4 Address | 192.168.20.1/24 |

##### VLAN40PERSONAL (OPT2)

| Field | Value |
|-------|-------|
| Enable | ✅ |
| IPv4 Config Type | Static IPv4 |
| IPv4 Address | 10.40.40.1/24 |

##### VLAN50DMZ (OPT3)

| Field | Value |
|-------|-------|
| Enable | ✅ |
| IPv4 Config Type | Static IPv4 |
| IPv4 Address | 172.16.50.1/24 |

#### DHCP Configuration — Kea DHCPv4

OPNsense 24.x ships with **Kea DHCPv4** as the default DHCP server (replacing the legacy ISC DHCPv4 backend). Subnets are configured individually under **Services → Kea DHCPv4 → Subnets**.

> 📷 **Photo:** Kea DHCPv4 — Subnets tab showing all four configured subnets with descriptions and pool ranges.

> 📷 **Photo:** Kea DHCPv4 — Service settings showing Enabled, all four interfaces selected in the Interfaces field, and Firewall Rules auto-generation checked.

| Subnet | Description | Pool Start | Pool End | Interface |
|--------|-------------|------------|----------|-----------|
| 10.10.10.0/24 | Management Subnet | 10.10.10.100 | 10.10.10.200 | VLAN10MGMT |
| 192.168.20.0/24 | Internal VLAN | 192.168.20.100 | 192.168.20.200 | VLAN20INTERNAL |
| 10.40.40.0/24 | Personal VLAN | 10.40.40.100 | 10.40.40.200 | VLAN40PERSONAL |
| 172.16.50.0/24 | DMZ VLAN | 172.16.50.100 | 172.16.50.200 | VLAN50DMZ |

The lower portion of each pool (`x.x.x.1` through `x.x.x.99`) is reserved for static assignments — gateway IPs, servers, and infrastructure hosts that will receive fixed addresses in later phases.

DNS server fields are left blank at this stage. Kea will fall back to OPNsense's Unbound resolver. Technitium DNS (VM 104) will be configured as the authoritative DNS server for `lab.local` in a later phase, at which point DHCP DNS options will be updated.

#### Issue Encountered — Kea Startup Failure

Kea failed to start after initial subnet entry. Root cause: the DMZ subnet was entered as `176.16.50.0/24` instead of `172.16.50.0/24` — a typo that caused Kea to reject the configuration because the subnet did not fall within the range of the VLAN50DMZ interface IP (`172.16.50.1`). Correcting the subnet to `172.16.50.0/24` resolved the issue and Kea started successfully.

> ⚠️ Kea will refuse to start if any subnet entry does not encompass its interface's configured IP address. Validate all subnet/interface IP pairs before applying.

#### Current State & Limitations

OPNsense routing and DHCP configuration is complete. The following validation is deferred until the MikroTik RB4011 is restored and the TP-Link switch is VLAN-configured:

- DHCP lease issuance to VMs on VLANs 20/40/50 cannot be tested — the switch is not yet passing tagged traffic to the Proxmox bridges for those VLANs
- VLAN 99 (Untrusted/IoT) is MikroTik-owned and not configured here — it will be brought up as part of Phase 2 MikroTik configuration
- Firewall rules are not configured — OPT interfaces (VLAN 20/40/50) default to block-all until explicitly permitted in a later phase

VLAN 10 (Management) is functional from any host directly connected to vmbr10, since the LAN default allow rule is in place and the Proxmox management network uses this bridge.

---
 
### VM 101 — splunk-siem
 
**Node:** pve-srv1  
**VLAN:** 10 (Management) — `10.10.10.0/24`  
**Storage:** rpool-data
 
#### Configuration
 
| Setting | Value |
|---------|-------|
| Machine | q35 |
| BIOS | OVMF (UEFI) |
| CPU | 4 cores (host) |
| RAM | 8GB |
| Disk | VirtIO Block, 100GB, rpool-data |
| Network | vmbr10, VirtIO |
| OS | Ubuntu Server 24.04 LTS |
 
#### Verification
 
- IP assigned via DHCP from OPNsense VLAN 10: `10.10.10.x`
- `ping 10.10.10.1` — ✅ Pass
 
---
 
### VM 102 — Wazuh
 
**Node:** pve-srv1  
**VLAN:** 10 (Management) — `10.10.10.0/24`  
**Storage:** rpool-data  
**Source:** Wazuh OVA (pre-built all-in-one appliance)
 
#### Final Working Configuration
 
| Setting | Value |
|---------|-------|
| Machine | i440fx |
| BIOS | SeaBIOS |
| CPU | 4 cores (host) |
| RAM | 8GB |
| Disk | **SCSI (scsi0)**, 25GB, rpool-data |
| Network | vmbr10, VirtIO |
| OS | Wazuh OVA (Ubuntu-based) |
 
#### Import Process
 
The Wazuh OVA was extracted and the VMDK converted to QCOW2 prior to import:
 
```bash
# Extract OVA
cd /var/lib/vz/images/
tar -xvf wazuh-*.ova
 
# Convert VMDK to QCOW2
qemu-img convert -f vmdk -O qcow2 wazuh-*.vmdk wazuh.qcow2
 
# Verify — confirmed clean (no backing file, corrupt: false)
qemu-img info /var/lib/vz/images/wazuh.qcow2
# file format: qcow2
# virtual size: 25 GiB
# disk size: 5.06 GiB
 
# Import to rpool-data
qm importdisk 102 /var/lib/vz/images/wazuh.qcow2 rpool-data
```
 
#### Issues Encountered
 
##### Issue 1 — Blank screen / blinking cursor on boot (first attempt)
 
**Symptom:** VM displayed the Wazuh splash screen, then a blank screen with a blinking cursor. No boot output, no GRUB menu, no kernel messages.
 
**Root cause:** The shell VM was initially created with `q35` + `OVMF (UEFI)`. The Wazuh OVA is built for legacy BIOS. UEFI firmware could not locate a bootable EFI partition on the OVA disk, causing a silent hang.
 
**Fix attempted:** Switched to `i440fx` + `SeaBIOS`, removed EFI disk. Issue persisted.
 
##### Issue 2 — Blank screen / blinking cursor persisted after BIOS fix
 
**Symptom:** Same blinking cursor behavior after correcting to `i440fx` + `SeaBIOS`.
 
**Root cause:** Disk was initially imported to `local` storage (directory-based, `/var/lib/vz/`) rather than `rpool-data` (ZFS pool). VM was deleted and disk re-imported to `rpool-data`. Issue still persisted, pointing to a second independent problem.
 
**Root cause (confirmed):** The disk was attached as `virtio0` (VirtIO Block). The Wazuh OVA's GRUB bootloader is configured to boot from `/dev/sda` (SCSI device naming). When the disk is presented as a VirtIO device, the OS sees it as `/dev/vda` — GRUB cannot find its root partition and hangs silently, producing only a blinking cursor.
 
**Fix:** Detached the VirtIO disk and re-attached as SCSI:
 
```bash
qm set 102 --delete virtio0
qm set 102 --scsi0 rpool-data:vm-102-disk-0,iothread=1
qm set 102 --boot order=scsi0
```
 
VM booted successfully after this change.
 
#### Key Lessons
 
- **Wazuh OVA requires `i440fx` + `SeaBIOS`** — not `q35` / OVMF. The OVA has no EFI partition.
- **Wazuh OVA requires SCSI disk bus** — not VirtIO Block. GRUB inside the OVA targets `/dev/sda`. Presenting the disk as VirtIO (`/dev/vda`) causes a silent GRUB hang with no error output.
- **Import target matters** — use `rpool-data` (ZFS) on pve-srv1, not `local` (directory). Directory-based storage works for ISOs and source files; ZFS-backed storage is the correct target for VM disks.
- **First boot takes several minutes** — Wazuh initializes the manager, indexer, and dashboard sequentially on first boot. The blinking cursor during this period is normal after GRUB hands off to the kernel.
 
#### Verification
 
- IP assigned via DHCP from OPNsense VLAN 10: `10.10.10.x`
- `ping 10.10.10.1` — ✅ Pass

---
 
### VM 103 — Security Onion
 
**Node:** pve-srv1
**VLAN:** 10 (Management) — `10.10.10.0/24`
**Storage:** rpool-data
 
#### Configuration
 
| Setting | Value |
|---------|-------|
| Machine | q35 |
| BIOS | OVMF (UEFI) |
| CPU | 4 cores (host) |
| RAM | 16GB |
| Disk | VirtIO Block, 200GB, rpool-data |
| NIC 1 (management) | vmbr10, VirtIO — `10.10.10.0/24` |
| NIC 2 (monitoring) | mirror port bridge, VirtIO — no IP |
| OS | Security Onion 2.4.211 |
 
#### Installation
 
Security Onion 2.4 uses a two-stage install: a base OS install followed by the `so-setup` wizard on first boot.
 
**Setup wizard selections:**
 
| Prompt | Selection |
|--------|-----------|
| Install type | Standard (internet access) |
| Node type | Standalone |
| Management NIC | First VirtIO NIC (vmbr10) |
| Management IP | DHCP from OPNsense VLAN 10 |
| Monitoring NIC | Second VirtIO NIC — no IP, monitor only |
| Home networks | `10.10.10.0/24`, `192.168.20.0/24`, `10.40.40.0/24`, `172.16.50.0/24` |
| Allowed management access | `10.10.10.0/24` |
 
The allowed management access range was set to VLAN 10 only (`10.10.10.0/24`). The Security Onion web UI is a management interface and should not be reachable from Internal, Personal, or DMZ VLANs directly — analysts access Security Onion data through Splunk in later phases.
 
#### Issue Encountered — Setup Exits with Memory Error
 
**Symptom:** Selecting "Standard" install type caused the wizard to immediately exit with the message:
 
```
This install type will fail with less than 16 GB of memory. Exiting setup.
```
 
**Root cause:** Security Onion 2.4 enforces a hard 16GB RAM minimum for the Standard installation type. The VM was initially spec'd at 8GB, which is half the required minimum.
 
**Fix:** Shut down the VM, updated RAM from 8192 MB to 16384 MB in Proxmox hardware settings, rebooted, and re-ran `so-setup`. The wizard proceeded normally after the memory increase.
 
> ⚠️ Security Onion 2.4 requires a minimum of **16GB RAM** for Standard/Standalone installs. The original 8GB spec was insufficient.

#### Verification
 
- IP assigned via DHCP from OPNsense VLAN 10: `10.10.10.x`
- `ping 10.10.10.1` — ✅ Pass
 
---
 
### VM 104 — Technitium DNS
 
**Node:** pve-srv1
**VLAN:** 10 (Management) — `10.10.10.0/24`
**Storage:** rpool-data
 
#### Configuration
 
| Setting | Value |
|---------|-------|
| Machine | q35 |
| BIOS | OVMF (UEFI) |
| CPU | 1 core (host) |
| RAM | 1GB |
| Disk | VirtIO Block, 16GB, rpool-data |
| Network | vmbr10, VirtIO |
| OS | Ubuntu Server 24.04 LTS |
| IP | Static — `10.10.10.10/24` |
 
Static IP was set via Netplan post-install. DNS servers should not rely on DHCP — a DNS server waiting on a DHCP lease creates a circular dependency at boot.
 
```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: no
      addresses:
        - 10.10.10.10/24
      routes:
        - to: default
          via: 10.10.10.1
      nameservers:
        addresses: [127.0.0.1]
```
 
Technitium DNS was installed via the official installer script:
 
```bash
curl -sSL https://download.technitium.com/dns/install.sh | sudo bash
```
 
#### Issue Encountered — Web UI Login Failure (Keyboard Input)
 
**Symptom:** After configuring credentials during OS install and saving the password to a password manager, login to the Technitium web UI at `http://10.10.10.10:5380` was rejected. Credentials appeared correct.
 
**Root cause:** The Proxmox console was accessed through LibreWolf. During Ubuntu installation, the **Shift key was not registering correctly** in the LibreWolf console session. Any uppercase characters typed during password setup were silently dropped — the password was recorded without capitals, but the password manager entry retained the intended (capitalized) version. The stored password and the actual set password did not match.
 
**Fix:** No credential reset was required. The password manager entry was reviewed and corrected to remove all capital letters, matching what was actually set during installation. Login succeeded after updating the entry.
 
> ⚠️ When accessing the Proxmox console through a browser (particularly LibreWolf or Firefox-based), modifier keys such as Shift, Ctrl, and Alt may not register reliably. Use SPICE client or verify keyboard input carefully when setting passwords during OS installation. Consider using an all-lowercase alphanumeric password during initial setup and changing it once SSH access is confirmed.
 
#### Verification
 
- Static IP confirmed: `10.10.10.10/24`
- `ping 10.10.10.1` — ✅ Pass
- `systemctl is-active dns.service` — ✅ active
 
> **Note:** DNS zone configuration, `lab.local` records, and forwarder rules are configured in a later phase.
