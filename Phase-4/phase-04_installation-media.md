# Phase 4_installation-media
**GitHub Dir:** phase-4_installation-media
**Objective:** Download all installation media required for Phase 5 VM deployment, verify file integrity where possible, and stage assets in the correct location on the Proxmox cluster. No VMs are created in this phase.

---
## Tasks

### ISOs — Upload to Proxmox ISO Storage
- [x]  **Ubuntu Server 24.04 LTS ISO**
    - Skills: LTS release cadence awareness, identifying correct image for VM workloads
    - Source: [https://ubuntu.com/download/server](https://ubuntu.com/download/server)
- [x]  **Windows 10 Pro ISO**
    - Skills: Microsoft Media Creation Tool, Windows licensing model
    - Attempted download via user agent spoofing, deadend, used media creation tool.
    - Source: [https://www.microsoft.com/en-us/software-download/windows10](https://www.microsoft.com/en-us/software-download/windows10) (Media Creation Tool)
    - Note: Microsoft does not publish SHA256 hashes for MCT-generated ISOs. Integrity relies on TLS from microsoft.com.
- [x]  **Windows Server 2022 ISO**
    - Skills: Microsoft Evaluation Center, server OS evaluation licensing
    - Source: [https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
    - Note: Same SHA256 caveat as Windows 10 — no published hash from Microsoft.
- [x]  **Kali Linux ISO**
    - Skills: Offensive distro sourcing, SHA256 verification
    - Source: [https://www.kali.org/get-kali/#kali-installer-images](https://www.kali.org/get-kali/#kali-installer-images)
- [x]  **Security Onion ISO**
    - Skills: NSM platform awareness, Security Onion architecture overview
    - Source: [https://github.com/Security-Onion-Solutions/securityonion/blob/2.4/main/DOWNLOAD_AND_VERIFY_ISO.md](https://github.com/Security-Onion-Solutions/securityonion/blob/2.4/main/DOWNLOAD_AND_VERIFY_ISO.md)
- [x]  **OPNsense ISO**
    - Skills: Firewall OS sourcing, FreeBSD-based OS awareness, SHA256 verification of extracted file from .bz2 archive
    - Source: [https://opnsense.org/download/](https://opnsense.org/download/)
    - Note: SHA256 verification used for (extracted) iso upload to proxmox for integrity.
- [x]  **VirtIO Drivers ISO**
    - Skills: Paravirtual drivers, Windows VM performance optimization
    - Source: [https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/)
    - Note: Required for Windows VMs (corp-dc, corp-win10) to use VirtIO disk and network drivers for performance.

---
### QCOW2 / OVA — Download to Proxmox Host
These formats are not uploaded to ISO storage. They are downloaded directly to the Proxmox host (e.g. `/var/lib/vz/images/` or a staging directory) and imported during Phase 5 VM creation.
 
- [ ] **REMnux QCOW2**
  - Skills: QCOW2 format awareness, SCP/wget file transfer to Proxmox host, understanding pre-built VM disk images vs installer ISOs
  - Source: https://docs.remnux.org/install-distro/get-virtual-appliance
  - Staging: Download to Proxmox host — import as VM disk in Phase 5 via `qm importdisk`
- [ ] **Wazuh OVA**
  - Skills: OVA format (VMware-origin appliance), `qm importovf` workflow, VMDK → QCOW2 conversion awareness (`qemu-img convert`), network adapter compatibility (VMXNET3 → VirtIO)
  - Source: https://documentation.wazuh.com/current/deployment-options/virtual-machine/virtual-machine.html
  - Staging: Download to Proxmox host — import in Phase 5
  - Note: Wazuh does not distribute a standalone ISO. The OVA is the official pre-built all-in-one appliance.
 
---
## Notes
Installation Locations:
Ubuntu Server: https://ubuntu.com/download/server
Kali Linux: https://www.kali.org/get-kali/#kali-installer-images
Security Onion: https://github.com/Security-Onion-Solutions/securityonion/blob/2.4/main/DOWNLOAD_AND_VERIFY_ISO.md
Windows 10: https://www.microsoft.com/en-us/software-download/windows10Tried 
	- User Agent trick (Linux/Firefox) got a download error, used media creation tool.
Windows Server: https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022
VirtIO Drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.285-1/
OPNsense: https://opnsense.org/download/


SHA256 verification check cmd via powershell: Get-FileHash (File location) -Algorithm SHA256
*SHA512 is required to verify the Wazuh download

| Download               | Downloaded Hash                                                                                                                  | Check Hash                                                                                                                       | Matches |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| Ubuntu Server          | E907D92EEEC9DF64163A7E454CBC8D7755E8DDC7ED42F99DBC80C40F1A138433                                                                 | e907d92eeec9df64163a7e454cbc8d7755e8ddc7ed42f99dbc80c40f1a138433                                                                 | ✅ TRUE  |
| Kali Linux             | 271477AD6EA2676C7346576971B9ACC2D32FABD9C2BBAF0E6302397626149306                                                                 | 271477ad6ea2676c7346576971b9acc2d32fabd9c2bbaf0e6302397626149306                                                                 | ✅ TRUE  |
| Security Onion         | CE6E61788DFC492E4897EEDC139D698B2EDBEB6B631DE0043F66E94AF8A0FF4E                                                                 | CE6E61788DFC492E4897EEDC139D698B2EDBEB6B631DE0043F66E94AF8A0FF4E                                                                 | ✅ TRUE  |
| REMnux (Proxmox QCOW2) | 95ADCFD293B29AEE77C0C95B2D0A9A7F8F2F7829C49F20B3DEF16B5B28638E93                                                                 | 95adcfd293b29aee77c0c95b2d0a9a7f8f2f7829c49f20b3def16b5b28638e93                                                                 | ✅ TRUE  |
| OPNsense (.bz2)        | 8B81427B049CA291BED982A85C6EB821E9887F70B79C1D8183C24721E037F938                                                                 | 8b81427b049ca291bed982a85c6eb821e9887f70b79c1d8183c24721e037f938                                                                 | ✅ TRUE  |
| Wazuh OVA (SHA512)     | 546C468E7481652F8DAEE8C2CD276070A6DCAFCEFAB90B5E99B269E85A62E5A65DEEABA765FF30710A1D23D9B3B4ED2E645C491DCBA6A87D30553F2E7A752CC8 | 546c468e7481652f8daee8c2cd276070a6dcafcefab90b5e99b269e85a62e5a65deeaba765ff30710a1d23d9b3b4ed2e645c491dcba6a87d30553f2e7a752cc8 | ✅ TRUE  |

*Hashes compared programmatically in Excel; case-normalized comparison returned TRUE for all verified files.*

*Windows and VirtIO ISOs sourced directly from authoritative Microsoft and Fedora-affiliated CDNs over HTTPS. No SHA256 hashes are published by these vendors for these specific artifacts. Integrity assurance relies on TLS transport security and file size validation.*
