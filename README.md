# AZ-802 Windows Server Hybrid Lab

A hands-on Windows Server 2025 homelab built to develop and demonstrate practical skills in identity, virtualization, remote administration, networking, storage, security, troubleshooting, and hybrid infrastructure.

The repository is aligned with AZ-802 subject areas, but it is designed as a professional systems-engineering portfolio rather than an exam-notes repository. Every completed lab records the architecture, implementation commands, validation evidence, troubleshooting process, limitations, and next steps.

## Current Status

The core infrastructure milestone was completed on 4 October 2026. The `FILE01` file-services foundation was completed and validated on 6 October 2026; read-only positive/negative access, hard-quota exhaustion, ABE, and cleanup were validated on 11 October, followed by DFS Namespace/GPP integration and cleanup on 12 October. The `CLIENT01` Windows 11 Enterprise deployment and domain-integration lab was completed and validated on 10 October 2026. The Lab 03 Group Policy practical block was completed and validated on 11 October 2026, including positive and negative tests and cleanup.

| Component | Current state |
|---|---|
| `HV-HOST01` | Windows Server 2025 Datacenter Core; Hyper-V, Windows Admin Center gateway, and Server Core App Compatibility installed |
| Host storage | Kingston A400 120 GB for `C:`; WD SN720 512 GB as `V:` ReFS datastore labelled `HyperV-VMs` |
| Virtual networking | External Hyper-V switch `vSwitch-External`; host management address `192.168.1.10` |
| `DC01` | Windows Server 2025 Standard Core; AD DS and DNS; `192.168.1.11` |
| Directory | Forest/domain `ad.petrovicinfra.com`; NetBIOS name `PETROVIC` |
| `MGMT01` | Windows Server 2025 Standard Desktop Experience; domain joined; RSAT and RDP; `192.168.1.13` |
| `FILE01` | Domain-joined file server in the File Servers OU; dedicated 40 GB dynamic data VHDX; NTFS `D:` volume labelled `Data` |
| `CLIENT01` | Windows 11 Enterprise workstation; Secure Boot and vTPM; joined to the Standard workstation OU; domain logon, Kerberos, GPO, RDP, and file mapping validated |
| Identity | Dedicated standard and privileged accounts, structured OUs, Global and Domain Local security groups |
| File services | `\\FILE01\IT`; AGDLP-controlled SMB/NTFS access with RO validation; ABE; FSRM hard-quota exhaustion and screening; VSS restore; Domain V2 DFS Namespace; GPP `I:` → `\\ad.petrovicinfra.com\Shares\IT` |
| Policy | Domain password/lockout policy, server/user baselines, and `GPO-DriveMap-IT` tested successfully |

Detailed implementation and validation notes are in:

- [Lab 01: Core Infrastructure Foundation](labs/01-active-directory/README.md)
- [Lab 03: Group Policy](labs/03-group-policy/README.md)
- [Lab 04: FILE01 File Services](labs/04-file-services/README.md)
- [Lab 05: CLIENT01 Windows 11 Enterprise Domain Integration](labs/05-hyper-v/README.md)

## Architecture

```mermaid
flowchart TB
    MAC["MacBook<br/>Browser and RDP client"]
    ROUTER["LAN Gateway<br/>192.168.1.1"]

    subgraph HOST["HV-HOST01 - 192.168.1.10"]
        OS["Windows Server 2025 Datacenter Core<br/>Hyper-V + WAC Gateway<br/>App Compatibility / Explorer"]
        STORAGE["C: Kingston A400 120 GB - Host OS<br/>V: WD SN720 512 GB ReFS - HyperV-VMs"]
        VSW["vSwitch-External"]

        subgraph VMS["Generation 2 virtual machines"]
            DC["DC01 - 192.168.1.11<br/>Windows Server 2025 Standard Core<br/>AD DS + DNS"]
            MGMT["MGMT01 - 192.168.1.13<br/>Windows Server 2025 Standard Desktop<br/>RSAT + RDP"]
            FS["FILE01<br/>File Server + FSRM + DFS Namespaces<br/>D: Data / \\\\FILE01\\IT"]
            CLIENT["CLIENT01<br/>Windows 11 Enterprise<br/>Secure Boot + vTPM"]
        end
    end

    MAC -->|"HTTPS / Windows Admin Center"| OS
    MAC -->|"RDP"| MGMT
    ROUTER --- VSW
    OS --- STORAGE
    OS --- VSW
    VSW --- DC
    VSW --- MGMT
    VSW --- FS
    VSW --- CLIENT
    MGMT -->|"ADUC / GPMC / DNS Manager / PowerShell"| DC
    DC -->|"AD DS, DNS and Group Policy"| MGMT
    DC -->|"AD DS, DNS and Group Policy"| FS
    DC -->|"AD DS, DNS, Kerberos and Group Policy"| CLIENT
    MGMT -->|"RSAT / GPMC / SMB administration"| FS
    CLIENT -->|"I: / domain-based DFS UNC / SMB"| FS
```

## Implemented Identity and Policy Model

```mermaid
flowchart LR
    USER["vladimir.petrovic<br/>standard account"]
    GG["GG-IT-Users<br/>Global security group"]
    DL["DL-MGMT01-RDP<br/>Domain Local security group"]
    LOCAL["MGMT01<br/>Remote Desktop Users"]
    RDP["RDP access"]

    USER --> GG --> DL --> LOCAL --> RDP
```

The privileged account is separated from daily-use identity:

```text
adm-vpetrovic
    -> GG-Tier0-Admins
        -> Domain Admins
```

The built-in domain Administrator is retained as a recovery account rather than used for routine administration.

## Validated Outcomes

- Hyper-V uses the dedicated ReFS datastore for default VM and VHDX paths.
- The external virtual switch preserves host management connectivity and connects the VMs to the LAN.
- `DC01` hosts the `ad.petrovicinfra.com` forest/domain and AD-integrated DNS.
- AD DS, DNS, Netlogon, NTDS, DFSR, SYSVOL, LDAP SRV records, and local DNS dynamic update were validated.
- The WAC/WinRM DNS issue was traced to an IPv6 DNS path through the router, not to the AD DNS zone.
- A WAC remote `dcdiag` warning was isolated to missing delegated Kerberos credentials; the same secure dynamic-update test passed locally on `DC01`.
- `MGMT01` was joined to the domain and configured with RSAT and RDP.
- `GPO-SRV-Management-Baseline` is applied to `MGMT01`.
- `GPO-User-Baseline` is applied to the standard user; Control Panel, Command Prompt, and Run were confirmed blocked.
- RDP access for the standard user works through the documented group-nesting model.
- `FILE01` is domain joined and placed in `OU=File Servers,OU=Servers,OU=PetrovicInfra,...`.
- A dedicated 40 GB dynamic VHDX provides the NTFS `D:` data volume, separate from the guest operating-system disk.
- `\\FILE01\IT` uses `GG-IT-Users`, `GG-Tier0-Admins`, and the `DL-FILE01-IT-RW/RO/FC` resource groups for AGDLP-based SMB and NTFS authorization.
- FSRM enforces a 5 GB hard quota and an active `Executable Files` file screen; a text file was accepted and an executable file was blocked.
- Read-only access allowed reads and denied create, modify, and delete operations. Hard-quota exhaustion rejected writes above 5 GB, and cleanup returned quota usage to `0.01 MB`.
- ABE before/after validation confirmed the intended RW/RO folder visibility; the `IT` share remains `AccessBased`, and temporary test identities and data were removed.
- Shadow Copies are scheduled for `D:` and a Previous Versions restore was completed successfully.
- The Domain V2 namespace `\\ad.petrovicinfra.com\Shares` is Online with root target `\\FILE01.ad.petrovicinfra.com\DFSRoot$`; DFS folder `IT` refers clients to `\\FILE01\IT`.
- `GPO-DriveMap-IT` maps `I:` to `\\ad.petrovicinfra.com\Shares\IT` for members of `GG-IT-Users` through the existing item-level targeting. `CLIENT01` mapping checks and read/write/delete validation passed; final test-file cleanup returned `False`.
- `CLIENT01` runs Windows 11 Enterprise with Secure Boot and vTPM, uses `DC01` for domain services, and is joined to the Standard workstation OU.
- `PETROVIC\vladimir.petrovic` signed in to `CLIENT01`; RDP and a valid Kerberos TGT through `DC01.ad.petrovicinfra.com` were confirmed.
- `GPO-User-Baseline` and `GPO-DriveMap-IT` applied on `CLIENT01`; Command Prompt, Run, and Control Panel were blocked, while `I:` provided working access through the existing AGDLP chain. The initial direct mapping was subsequently changed to the validated DFS UNC.
- Lab 03 validated Security Filtering, Read vs Apply Group Policy, inheritance and precedence, Block Inheritance, Enforced, Loopback Merge/Replace, positive and negative WMI filtering, and `gpresult`/HTML RSoP troubleshooting on `CLIENT01`; normal behavior was restored after cleanup.

## Repository Roadmap

| Area | Status | Next evidence |
|---|---|---|
| Core AD domain and CLIENT01 integration | Completed and validated | Continue with the separate AD resilience project: second DC, replication, and DNS redundancy |
| Group Policy — Lab 03 | Practical block completed and validated; cleanup confirmed | Preserve the documented scope and filtering behavior in later regression tests |
| File services | `FILE01` foundation, RO/quota/ABE, DFS Namespace/GPP, and cleanup completed and validated | TrueNAS, storage integration, and independent backup remain planned; DFS Replication and HA have not been implemented |
| AD resilience and recovery | Planned | Replication, FSMO, DNS redundancy, and recovery exercises |
| Hyper-V operations | In progress; `CLIENT01` deployment validated | Export/import, recovery, and repeatable inventory |
| Network services and VPN | VPN01/NPS is the next practical block | Inventory and infrastructure prerequisites for VPN01/NPS; DHCP remains planned |
| Security and monitoring | Planned | LAPS, Defender, auditing, Zabbix, and incident exercises |
| Hybrid integration | Planned | Azure Arc and controlled Azure services within budget |
| Backup and disaster recovery | Planned | Measured file, directory, and service restoration |

## Documentation Standards

- Every lab includes prerequisites, architecture, exact commands, validation, troubleshooting, lessons learned, and limitations.
- Results are marked complete only when they have been tested.
- Passwords, tokens, private keys, certificates, public home IP addresses, VM disks, ISO images, and raw sensitive exports are never committed.
- Screenshots are optional supporting evidence; commands and observed outcomes provide the primary technical record.
- This is a homelab. Multiple VMs on one physical host do not demonstrate host-level high availability.

## Next Step

Continue with the existing VPN01/NPS practical block, starting with inventory and infrastructure prerequisites. The completed FILE01 RO/quota/ABE and DFS tests remain closed. The separate Windows workstation security baseline remains planned, beginning with Windows LAPS, then Defender Firewall, auditing, local administrator group management, and PowerShell logging. DHCP and the broader TrueNAS/storage work remain planned.
