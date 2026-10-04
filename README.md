
# AZ-802 Windows Server Hybrid Lab

A hands-on Windows Server 2025 homelab built to develop and demonstrate practical skills in identity, virtualization, remote administration, networking, storage, security, troubleshooting, and hybrid infrastructure.

The repository is aligned with AZ-802 subject areas, but it is designed as a professional systems-engineering portfolio rather than an exam-notes repository. Every completed lab records the architecture, implementation commands, validation evidence, troubleshooting process, limitations, and next steps.

## Current Status

The first infrastructure milestone was completed and validated on 4 October 2026.

| Component | Current state |
|---|---|
| `HV-HOST01` | Windows Server 2025 Datacenter Core; Hyper-V, Windows Admin Center gateway, and Server Core App Compatibility installed |
| Host storage | Kingston A400 120 GB for `C:`; WD SN720 512 GB as `V:` ReFS datastore labelled `HyperV-VMs` |
| Virtual networking | External Hyper-V switch `vSwitch-External`; host management address `192.168.1.10` |
| `DC01` | Windows Server 2025 Standard Core; AD DS and DNS; `192.168.1.11` |
| Directory | Forest/domain `ad.petrovicinfra.com`; NetBIOS name `PETROVIC` |
| `MGMT01` | Windows Server 2025 Standard Desktop Experience; domain joined; RSAT and RDP; `192.168.1.13` |
| Identity | Dedicated standard and privileged accounts, structured OUs, Global and Domain Local security groups |
| Policy | Domain password/lockout policy and two custom GPOs tested successfully |

Detailed implementation and validation notes are in [Lab 01: Core Infrastructure Foundation](labs/01-active-directory/README.md).

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
            FS["FS01 - next milestone<br/>File Services + AGDLP permissions"]
        end
    end

    MAC -->|"HTTPS / Windows Admin Center"| OS
    MAC -->|"RDP"| MGMT
    ROUTER --- VSW
    OS --- STORAGE
    OS --- VSW
    VSW --- DC
    VSW --- MGMT
    VSW -.-> FS
    MGMT -->|"ADUC / GPMC / DNS Manager / PowerShell"| DC
    DC -->|"AD DS, DNS and Group Policy"| MGMT
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

## Repository Roadmap

| Area | Status | Next evidence |
|---|---|---|
| Core infrastructure, AD DS, DNS, management, and initial GPO | In progress; foundation validated | Add a second DC and a client VM |
| File services | Next | Build `FS01`; test share and NTFS permissions with AGDLP |
| AD resilience and recovery | Planned | Replication, FSMO, DNS redundancy, and recovery exercises |
| Hyper-V operations | In progress | Export/import, recovery, and repeatable inventory |
| Network services and VPN | Planned | DHCP, routing, DNS scenarios, and private remote access |
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

Build `FS01`, join it to the domain, place it in the File Servers OU, and implement departmental shares using the following model:

```text
Account -> Global role group -> Domain Local resource group -> Share and NTFS permissions
```

The lab will include both allowed and denied access tests, Effective Access validation, and a clear comparison of share permissions versus NTFS permissions.
