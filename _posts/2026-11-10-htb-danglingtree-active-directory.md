---
layout: post
title: "Hack The Box — DanglingTree | Windows & Active Directory Pentest"
date: 2026-11-10 12:00:00 +0100
categories: [HackTheBox, Active Directory]
tags: [hackthebox, windows, active-directory, adcs, kerberos, dpapi, pentest, privilege-escalation]
description: "A focused case study of DanglingTree (HTB Medium): Windows infrastructure assessment, application pivoting, Active Directory permissions and AD CS privilege escalation."
pin: true
---

> **Type:** Authorized Hack The Box laboratory · **Target:** DanglingTree · **OS:** Windows / Active Directory · **Difficulty:** Medium · **Result:** User and root objectives completed.

## Project overview

**DanglingTree** is a Windows and Active Directory penetration-testing lab on Hack The Box. I used it to practice a complete assessment workflow: identifying exposed services, validating an initial access path, pivoting through an internal application, analyzing identity and authorization weaknesses, and demonstrating the impact of an Active Directory Certificate Services (AD CS) misconfiguration.

This is **hands-on lab work**, not a commercial penetration test or an engagement performed for a client.

## Scope and objectives

- Enumerate the Windows infrastructure and Active Directory attack surface.
- Investigate reachable network services and internal applications.
- Evaluate account permissions, credential exposure and trust boundaries.
- Validate a privilege-escalation path through Active Directory and AD CS.
- Record the evidence, attack-chain reasoning and practical defensive lessons.

## Assessment summary

| Area | What I worked on |
| --- | --- |
| Network and service discovery | Nmap, SMB, LDAP, Kerberos and Windows service enumeration |
| Initial access and pivoting | Review of accessible shares, Windows Admin Center and access to an internal application |
| Application security | Analysis of SmarterMail weaknesses, including CVE-2026-23760 and CVE-2026-24423 in the lab environment |
| Credential security | Backup review and Windows DPAPI-protected credential analysis |
| Active Directory authorization | Account relationships, delegated permissions and ACL review, including `ForceChangePassword` |
| AD CS security | Certificate-template permissions, enrollment configuration and ESC1-style certificate impersonation |
| Verification | Certificate-based Kerberos authentication and validation of the final administrative access |

## Methodology and technical work

### 1. Reconnaissance and attack-surface mapping

I enumerated network services and identified a Windows environment with Active Directory components. SMB and LDAP enumeration helped connect exposed services to accounts, access permissions and possible entry points.

### 2. Windows and application assessment

The investigation progressed from Windows management functionality to an internally reachable mail application. I assessed the application security boundary, tested the relevant weaknesses in the lab, and evaluated the resulting level of access.

### 3. Identity and privilege-path analysis

Rather than treating privilege escalation as a local Windows exploit, I followed the relationships between application accounts, protected credentials, AD group membership and delegated permissions. This required interpreting DPAPI artifacts and AD access-control rules.

### 4. AD CS configuration review

The final stage focused on the consistency between published certificate templates and their directory objects, the effective permissions on those objects, and the security impact of certificate enrollment settings. The validated path reached administrative access through certificate-based Kerberos authentication.

## Tools and concepts

`Nmap` · `NetExec` · `SMB` · `LDAP` · `Kerberos` · `Impacket` · `Certipy` · `DPAPI` · `AD ACLs` · `AD CS` · `PKINIT`

## Outcome

- **Completed the lab:** both user and root objectives were achieved.
- **Documented an end-to-end attack path** across network services, application security, credential handling, Active Directory authorization and AD CS.
- **Produced a technical write-up** that distinguishes observed evidence, failed attempts, successful validation and remediation ideas.

No flags, live credentials or target-specific exploit commands are reproduced on this portfolio page.

## Defensive takeaways

1. Review SMB share access and avoid exposing sensitive assessment documents or credentials.
2. Restrict management interfaces and the ability to reach internal-only applications.
3. Protect backups and application secrets; assess their exposure to lower-privileged identities.
4. Audit delegated AD permissions and sensitive account-control rights.
5. Monitor AD CS template ownership, DACL changes and enrollment policies, including the ability to supply certificate subjects.
6. Correlate authentication and privilege changes across Windows, Kerberos and certificate services.

## Full technical documentation

For readers who want the detailed technical investigation, the machine-specific write-up is maintained separately:

**[DanglingTree — full Hack The Box write-up on GitHub](https://github.com/ALLAKORI/htb-season-11-writeups/blob/main/DanglingTree/README.md)**

This page is intentionally a **focused project case study** suitable for a CV or portfolio. It does not list or summarize the other Season 11 machines.
