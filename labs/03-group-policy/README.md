# Lab 03 — Group Policy

**Date completed:** 11 October 2026

**Status:** Completed and validated — practical block PASS; cleanup confirmed

**Related AZ-802 areas:** Active Directory Domain Services, Group Policy management, security filtering, inheritance, loopback processing, WMI filtering, and troubleshooting

## Objective

Validate how Group Policy scope, permissions, inheritance, and client-side filtering affect the resulting user configuration on the existing `CLIENT01` workstation. Diagnose both successful application and deliberate exclusion, then restore normal lab behavior.

## Business Scenario

A domain workstation receives several overlapping policies. The administrator must explain why a policy applies, why another is denied, which setting wins a conflict, and how the workstation can influence user settings through loopback processing.

## Scope

The completed practical block covers:

- positive and negative security-filtering tests using `GG-IT-Users`;
- the distinction between `Read` and `Apply Group Policy`;
- `Denied (Security)` troubleshooting;
- inheritance and parent/child policy precedence;
- Block Inheritance and Enforced links;
- loopback processing in Merge and Replace modes;
- positive Windows 11 and negative domain-controller-only WMI filtering;
- `gpresult /r` and HTML RSoP reporting through `gpresult /h`;
- cleanup, including the final unlink that restored normal behavior.

This is the existing Lab 03 section. It extends the policy foundation recorded in [Lab 01](../01-active-directory/README.md) and the working client recorded in [Lab 05](../05-hyper-v/README.md).

## Existing Environment and Prerequisites

| Component | Confirmed context |
|---|---|
| Hyper-V host | `HV-HOST01`, hosting the existing domain lab |
| Domain controller | `DC01`, AD DS and DNS for `ad.petrovicinfra.com` |
| Management system | `MGMT01`, with the existing RSAT and GPMC tools |
| Test workstation | `CLIENT01`, Windows 11 Enterprise, domain joined |
| NetBIOS domain | `PETROVIC` |
| Standard user | `PETROVIC\vladimir.petrovic` |
| Target security group | `PETROVIC\GG-IT-Users` |
| Existing user OU | `OU=IT,OU=Users,OU=PetrovicInfra,DC=ad,DC=petrovicinfra,DC=com` |
| Existing computer OU | `OU=Standard,OU=Workstations,OU=PetrovicInfra,DC=ad,DC=petrovicinfra,DC=com` |
| Established user policies | `GPO-User-Baseline` and `GPO-DriveMap-IT` |

The prerequisite domain sign-in, user restrictions, and `I:` mapping were already validated in Lab 05. They were the established environment for this session, rather than a new deployment.

## Practical Tests and Observed Results

### 1. Security Filtering: positive and negative tests

The positive test demonstrated policy application for the intended `GG-IT-Users` scope. The negative test demonstrated exclusion and exposed `Denied (Security)` in the policy results.

The permission review distinguished two separate requirements: `Read` permits access to the GPO; `Apply Group Policy` permits application to an eligible target. Read-only access does not authorize application. Both permissions are required for the target to apply the GPO, subject to scope and other filters. See [Microsoft's filtering reference](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/Policy/filtering-the-scope-of-a-gpo).

For user policies, the computer also needs read access to retrieve the GPO. Removing `Authenticated Users` from Security Filtering removes its read access as well as its application permission; read access can be granted separately through Delegation. This distinction supports troubleshooting without broadening the intended user scope. See [Microsoft's user-GPO permissions guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/group-policy/cannot-apply-user-gpo-when-computer-objects-dont-have-read-permissions).

The negative result is a successful boundary test, not an unresolved lab failure. The session record does not establish that an explicit Deny ACL or a newly created negative-test account was used.

### 2. Inheritance and precedence

A parent/child GPO conflict was tested, and the effective result was checked on `CLIENT01`. Under ordinary inheritance, the later child-OU policy won the conflicting setting. This confirmed that a linked GPO and the winning configuration are different questions.

The exact temporary GPO names and conflicting setting values were not retained in the available completion record, so this document records the validated behavior without inventing those identifiers.

### 3. Block Inheritance and Enforced

Block Inheritance was tested to exclude ordinary inherited policies from the affected OU. The Enforced test then demonstrated that an enforced parent link survived that block and preserved its conflicting setting against the child policy.

Block Inheritance is an OU/domain property; Enforced is a GPO-link property. Enforced does not bypass security or WMI filtering. The processing rules are described in [Microsoft's Group Policy processing reference](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing).

### 4. Loopback processing: Merge and Replace

Both loopback modes were tested on `CLIENT01`, comparing the effective user-policy result:

| Mode | Validated behavior |
|---|---|
| Merge | Normal user-scope policies were combined with user settings selected through the computer's scope; the latter took precedence in a conflict |
| Replace | User settings were selected through the computer's scope instead of the normal user-OU policy list |

These tests demonstrated how the same user can receive a different user-policy result depending on the workstation. The policy location and mode definitions are documented in [Microsoft's loopback processing guidance](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing#loopback-processing-mode).

### 5. WMI filtering: positive and negative tests

The Windows 11 filter produced the positive result on `CLIENT01`. The domain-controller-only filter produced the negative result on that workstation, with `Denied (WMI Filter)` reported.

The negative test was performed on `CLIENT01`; it does not claim a positive execution test on `DC01`. The exact WQL statements, namespaces, and filter identifiers were not retained in the available completion record and are not reconstructed here.

WMI conditions are evaluated on the destination computer. A GPO must satisfy its WMI condition as well as its scope and permissions. See [Microsoft's filtering guidance](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-processing#group-policy-filtering).

### 6. RSoP and troubleshooting

The session used these reporting methods:

```text
gpresult /r
gpresult /h <HTML-report-path>
```

`/r` provided the policy summary. The HTML report supported inspection of applied and denied policies, denial reasons, and the effective settings when inheritance or loopback changed the result. The report destination is shown as a placeholder because its exact path was not retained. See [Microsoft's gpresult command reference](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/gpresult).

The completed troubleshooting distinguished permission-based exclusion (`Denied (Security)`) from a false WMI condition (`Denied (WMI Filter)`) and used the resulting policy data to resolve precedence and loopback behavior.

## Validation Evidence

| Test | Recorded outcome | Status |
|---|---|---|
| Security Filtering — positive | Policy applied for the intended `GG-IT-Users` scope | PASS |
| Security Filtering — negative | Policy excluded; `Denied (Security)` identified | PASS |
| Read vs Apply Group Policy | Read access distinguished from permission to apply the GPO | PASS |
| Inheritance / precedence | Effective winner confirmed in a parent/child conflict | PASS |
| Block Inheritance | Ordinary inherited policy excluded from the affected OU | PASS |
| Enforced | Enforced parent link survived blocking and the conflicting child policy | PASS |
| Loopback Merge | Normal and computer-scope user policies combined; conflict precedence confirmed | PASS |
| Loopback Replace | Computer-scope user-policy list replaced the normal user-OU list | PASS |
| WMI — Windows 11 positive | Filter allowed policy application on `CLIENT01` | PASS |
| WMI — domain-controller-only negative | Filter excluded `CLIENT01`; `Denied (WMI Filter)` identified | PASS |
| Summary reporting | `gpresult /r` used for RSoP troubleshooting | PASS |
| HTML reporting | `gpresult /h` report generated and reviewed | PASS |
| Cleanup / restoration | Normal behavior confirmed after the final test link was unlinked | PASS |

## Cleanup and Restoration

Cleanup was completed at the end of the practical block. A remaining test link still affected the result until it was unlinked. The user then confirmed that normal behavior was restored.

The final unlink is recorded as the confirmed restoration step. Unlinking a GPO removes that link; it does not mean that the GPO object was deleted. The available record does not contain a full post-cleanup link, ACL, WMI, or loopback export, so it does not support asserting every final object-level value.

The established drive-map design remains documented in [Lab 04](../04-file-services/README.md): `GPO-DriveMap-IT` uses item-level targeting for `GG-IT-Users`, and file authorization uses the existing AGDLP chain.

## Evidence Boundaries

The results above are the completed practical tests recorded in the 11 October 2026 session and confirmed in its completion request. This documentation update did not rerun the Windows lab. Raw session screenshots, the generated HTML report, temporary GPO identifiers, exact conflict values, and WQL queries are not included in this repository.

Windows LAPS, Defender Firewall, advanced auditing, local administrator management, and PowerShell logging remain separate planned work. Completing this Group Policy block does not mark the wider hardening project complete.

## Key Takeaways

- A linked GPO can still be excluded by security or WMI filtering.
- Read permission and application permission must be examined separately.
- Inheritance, enforcement, and loopback can change which configuration wins.
- RSoP results explain the effective configuration more reliably than a link list alone.
- Cleanup must include remaining test links and a check of restored client behavior.

## Completion and Follow-up

The Group Policy practical block is completed and validated, including its positive and negative tests and confirmed restoration. Its documentation occupies the existing `labs/03-group-policy` section.

The continuation recorded at the end of the session is the existing `FILE01` advanced practical block; broader security and infrastructure milestones remain tracked in the Practical Lab Hub.
