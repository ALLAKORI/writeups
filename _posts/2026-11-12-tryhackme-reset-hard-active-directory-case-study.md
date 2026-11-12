---
layout: post
title: "TryHackMe — Reset (Hard) | Active Directory Pentest: Technical Case Study"
date: 2026-11-12 19:30:00 +0100
categories: [TryHackMe, Active Directory]
tags: [tryhackme, active-directory, windows, kerberos, bloodhound, acl, smb, pentest, red-team, delegation]
description: "Evidence-based technical case study of TryHackMe Reset (Hard): Windows/AD enumeration, BloodHound methodology, Kerberos, authorization boundaries, reporting and detection."
mermaid: true
---

> **Platform:** TryHackMe | **Room:** [Reset](https://tryhackme.com/room/resetui) | **Difficulty:** Hard | **Completed:** 30 October 2026 | **Author:** Kossi Richard Allado (allakori)

## Executive summary

I completed the **Reset** Active Directory challenge, rated *Hard*, and verified both objectives on **30 October 2026**. My experience was unexpected: once I understood the domain's authorization relationships, I found the exercise much more straightforward than its difficulty label suggested. I described it afterward as the easiest Active Directory lab I had tackled.

What made it valuable was **the reasoning between tools**. A service scan cannot prove a vulnerable privilege boundary; a working credential does not guarantee WinRM access; and an edge in BloodHound is not itself a demonstrated compromise. The task is to connect observations, formulate hypotheses, validate actual permissions, and explain the impact.

**Publication boundary:** This is a technical, recruiter-readable **non-spoiler case study**. The room is not confirmed retired, so I am not distributing its specific attack chain, account-to-account escalation path, target-specific exploitation commands, passwords, answers or flags. TryHackMe's current [Acceptable Use Policy](https://tryhackme.com/legal/acceptable-use-policy) prohibits complete walkthroughs of active content. The content below still documents real findings and practical audit methodology. Generic command examples are labeled as examples; they are **not represented as copied outputs from my terminal**.

## 1. Assessment scope and evidence standard

This was an authorized, isolated training environment, **not a client engagement**. The domain used Microsoft Windows and Active Directory, exposing a range of authentication, file-sharing and management services.

I used the following evidence categories in my notes:

| Category | Meaning | Appropriate language in a report |
| --- | --- | --- |
| **Observed** | A saved command or terminal result supports the fact directly | “The test returned…” |
| **Validated** | The operation succeeded and demonstrated a specific permission or capability | “This account could perform…” |
| **Inferred** | The hypothesis is plausible but the exact evidence is missing | “This may indicate…” |
| **Reference** | A general technique confirmed by documentation or independent research | “The tool supports…” |

**Recorded outcome:** I completed both objectives. My retained notes also document an initial network assessment, SMB/document inspection, an offline password-recovery stage, BloodHound relationship analysis, and a later Kerberos-authenticated service-access stage. I have preserved the machine-specific sequence separately.

## 2. Reconnaissance — make every port answer a question

Windows domain assessments generate a lot of noise. A useful first pass answers three questions:

1. What services are **reachable**?
2. Which ones expose information to **anonymous or low-privileged** identities?
3. Which ones can support **authenticated security validation**, if I obtain authorized test credentials?

My original assessment found Windows services including Kerberos, LDAP, SMB and RPC. I also checked the management and TLS-facing surfaces rather than assuming every advertised port was usable.

### Tooling reference: targeted Nmap inventory

The following is a **generic authorized-lab example**, not the original room-specific command:

~~~bash
# Example only. Set LAB_IP to your own authorized target.
nmap -Pn -sV -p 53,88,135,139,389,445,464,636,3268,3269,5985 LAB_IP
~~~

| Typical port | Function | Why I investigate it |
| --- | --- | --- |
| 53 | DNS | Domain discovery and name resolution |
| 88 / 464 | Kerberos | Realm, authentication and credential-management boundaries |
| 135 / 139 / 445 | RPC / NetBIOS / SMB | File-sharing permissions, RPC interfaces and Windows environment context |
| 389 / 636 | LDAP / LDAPS | Directory objects, identity relations and transport protection |
| 3268 / 3269 | Global Catalog | Cross-domain directory discovery, where available |
| 5985 / 5986 | WinRM | Remote-management exposure **and** authorization requirements |

**Interpretation:** Seeing TCP/445 open is a finding about exposure; it does **not** establish that a share can be read, that files are sensitive or that the host is compromised.

### Evidence capture

For a professional report I normally retain the date/time, the exact command, the network scope and a brief interpretation. Service banners can help identify a technology stack but should not be treated as definitive proof of a vulnerability without verification.

## 3. SMB review — listing a share is not the same as reading it

SMB access deserves multiple checks. An unauthenticated session, guest mapping and a domain-authenticated session can all have different effective rights.

A useful permission matrix is:

| Test | Question | Correct conclusion if successful |
| --- | --- | --- |
| Enumerate share names | Which resources are advertised? | Names are visible to the session |
| List a share directory | Can this identity list contents? | Directory-listing permission is present |
| Read a file | Does the identity have effective file-read rights? | That particular file is readable |
| Examine the content | Is exposed data actually sensitive? | A finding depends on what was disclosed |
| Retest another identity | Is the permission appropriately restricted? | Compare the impact of different security contexts |

### Tooling reference: SMB authorization checks

~~~bash
# Generic examples, not a copied Reset solution.
smbclient -L //LAB_IP -N
nxc smb LAB_IP -u '' -p '' --shares
# With an authorized test identity:
nxc smb LAB_IP -u TEST_USER -p 'TEST_PASSWORD' --shares
~~~

**A key observation from my lab work** was that the value of shared documents comes from their content and associated access rights, not simply from their filenames. A file that looks like ordinary onboarding or operations material should be examined for excessive disclosure, while avoiding the publication of real credentials.

For document review in an authorized environment, a useful **generic** local command is:

~~~bash
pdftotext -layout local_document.pdf -
~~~

This produces text for inspection. It does **not** determine the sensitivity of a file by itself: that requires a human review of its purpose, access permissions and content.

**Defensive angle:** remove secrets from onboarding documents and scripts, restrict anonymous read/write access, and rotate any exposed credentials. A share ACL is a security control; an administrative naming convention is not.

## 4. Identity testing — authentication is not authorization

A frequent source of mistakes in beginner write-ups is treating a recovered password as though it automatically provides interactive access to a server.

It doesn't.

| Security boundary | Evidence required |
| --- | --- |
| Correct account/password | Authentication accepted by the relevant service |
| Remote management allowed | Successful authorized WinRM, RDP or equivalent session |
| Access to a share | Actual directory/file operation succeeds |
| Control over a user object | Authorized operation within the object's effective ACL |
| Sensitive service access | The target service accepts the identity or ticket for the tested action |

This distinction helped keep the Reset assessment reproducible and the conclusions defensible.

### Kerberos pre-authentication and offline password security

One general identity risk is a domain account configured without Kerberos pre-authentication. Such a setting can allow an attacker to obtain authentication material and attempt an **offline dictionary attack**; successful cracking still says nothing, by itself, about WinRM permissions or domain administration.

The following is a **tool syntax reference** for a permitted lab, not the target-account command from Reset:

~~~bash
# Offline analysis of an AS-REP test capture from an authorized lab.
john --format=krb5asrep \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  asrep.hashes
~~~

The relevant security controls are to require pre-authentication where possible, enforce strong passwords, review exception accounts and correlate unusual Kerberos requests with subsequent authentication attempts.

**Do not interpret a cracked password as an automatic compromise of the domain.** Report the concrete rights validated with that credential.

## 5. BloodHound — understanding the permission graph

This was the most instructive part of the exercise for me. BloodHound changes the focus from individual tools to **who can control whom, under what conditions**.

My retained collection output reported:

- **42 users**
- **55 groups**
- **3 GPOs**
- **222 organizational units**

These are **counts from my original collection**, not a claim that all those objects were vulnerable.

### What to look for in an AD graph

| Edge / condition | Security significance | What must be confirmed |
| --- | --- | --- |
| **GenericAll** | Broad control over an object may make security-sensitive modifications possible | Source principal, target object, actual effective permission |
| **ForceChangePassword** | An actor may have delegated ability to reset another identity's password | The target, permission scope and whether the action is authorized |
| **WriteDACL** | An actor may be able to alter an object's access-control list | The rights granted and possible downstream effect |
| **GenericWrite** | Write permission over some object attributes | Which writable attributes have security impact |
| **Group membership** | Membership can confer access through nested groups | Effective, not merely nominal, group privileges |
| **Delegation attributes** | A service identity may be allowed to act for users in a defined context | Delegation type, target services and identity restrictions |

The right question is not just **“Where is the shortest path?”** It is:

> What exact control exists, why does it exist, can it be exercised in the current context, and what security boundary would be crossed?

### Interpreting a BloodHound result

A graph edge may reflect configuration, historical collection data or a right with operational restrictions. For a professional assessment, I would:

1. Capture the relevant edge and its underlying object attributes.
2. Identify the assigning principal and whether the permission is direct or inherited.
3. Evaluate the business need for that delegation.
4. Test the effective right only within the approved scope.
5. Distinguish a **theoretical path** from an **observed authorization failure**.

**Why it matters:** The best remediation is often to remove one excessive delegated right that enables several potential escalation paths, rather than attempting to detect every individual exploitation tool.

The specific account names and edge-by-edge route from Reset are **not** shown in this public document.

## 6. Kerberos tickets and constrained delegation

Kerberos delegation allows certain services to access other services on a user's behalf. It is legitimate enterprise functionality, but its trust boundaries require careful review.

Three technical terms are important:

- **TGT:** Ticket-Granting Ticket, used to obtain service tickets.
- **TGS/service ticket:** ticket intended for a particular service principal name (**SPN**).
- **S4U:** Kerberos extensions supporting service-side impersonation and delegation scenarios under applicable configuration and policy.

The distinction between **holding a ticket** and **being able to use it** matters. A ticket may be valid for one SPN and still be useless for another operation.

### Ticket inspection without publishing secrets

The following is generic diagnostic syntax for a ticket cache in an authorized assessment:

~~~bash
# Generic Kerberos cache inspection. Do not publish the actual cache.
export KRB5CCNAME=/path/to/authorized-lab-ticket.ccache
klist
~~~

A useful note should record the ticket's intended service, identity context, validity and the exact permitted operation tested — without uploading a reusable ticket, password, hash or flag.

### What defenders should audit

- Accounts allowed to delegate and the list of permitted services.
- Changes to delegation-related directory attributes.
- Permissions over sensitive service accounts.
- Whether privileged identities are protected from inappropriate delegation.
- Service-ticket requests inconsistent with expected application behavior.

**Impact must be service-specific.** A ticket accepted by one service does not automatically imply control of all services or knowledge of a privileged account's password.

## 7. Investigation workflow: what to do when stuck

This is an **audit decision workflow**, not a diagram of the room's solution:

~~~mermaid
flowchart TD
    A["Collect authorized scope and service evidence"] --> B["Identify tested security boundary"]
    B --> C{"Is the observation reproducible?"}
    C -- "No" --> D["Check name resolution, rights, errors and assumptions"]
    D --> B
    C -- "Yes" --> E["Document actual identity, capability and limits"]
    E --> F{"Is the security impact demonstrated?"}
    F -- "Not yet" --> G["Create a narrower, testable hypothesis"]
    G --> B
    F -- "Yes" --> H["Write finding, remediation and retest criteria"]
~~~

When a test fails, I do not simply replace the command with another tool. I ask whether the failure comes from:

- Authentication: invalid or incorrectly formatted credentials.
- Authorization: a valid identity without permission to perform the action.
- Service configuration: the protocol is unavailable or disabled.
- Kerberos naming: DNS, domain, SPN or ticket-cache mismatch.
- Evidence quality: the information collected does not support the current hypothesis.

This practice also prevents a common reporting mistake: converting a failed hypothesis into a claimed success after reading someone else's write-up.

## 8. Detection and logging opportunities

The following are **defender-oriented investigation ideas**, not a claim that these exact event IDs appeared in my retained Reset logs.

| Activity to investigate | Relevant Windows telemetry | Interpretation caution |
| --- | --- | --- |
| Unusual authentication requests | **4768**, **4769** | Correlate with normal service behavior and identity baseline |
| Account password resets | **4724** | Legitimate help-desk changes can create similar events |
| Sensitive account changes | **4738**, **5136** when applicable and audited | Requires appropriate directory auditing |
| Access to file shares | **5140**, **5145** with suitable auditing | Share access alone may be expected |
| Changes to sensitive directory rights | **5136**, **4670** in relevant contexts | Check whether the object and auditing settings support the event |
| Sensitive directory replication activities | **4662** with relevant SACLs | Not all 4662 events indicate malicious replication |

A meaningful alert often needs **context across sources**: who initiated the change, which object was targeted, whether it was in a maintenance window, what happened next, and whether the event reflects a normal business process.

## 9. Converting lab work into a professional finding

The report should show what was *verified*, not simply list impressive tool names. Here is the reporting format I use.

| Field | Content |
| --- | --- |
| **Finding** | Specific misconfiguration or excessive permission |
| **Scope** | Affected identity or service class, with sensitive details removed |
| **Preconditions** | Access level needed before the issue can be exercised |
| **Evidence** | Saved output, its time, relevant sanitized fields |
| **Root cause** | Why the configuration permitted the unexpected behavior |
| **Impact** | Exactly which security boundary was crossed |
| **Severity** | Based on prerequisites and demonstrated impact, not tool reputation |
| **Remediation** | Implementable changes with an accountable owner |
| **Retest** | Observable condition that proves the control now works |

### Finding A: excessive access to operational material

**Risk:** Overly broad access to documents or scripts may expose information useful to an attacker.

**Recommendation:** Restrict access by role, remove embedded secrets, audit anonymous and guest permissions, and monitor read/write operations on sensitive shares.

**Retest:** Use an unauthorized low-privilege test identity and verify that sensitive file reads are rejected while legitimate workflows still function.

### Finding B: transitive delegated control in Active Directory

**Risk:** Rights that appear harmless in isolation may become high impact when they connect multiple objects.

**Recommendation:** Review and minimize GenericAll, GenericWrite, WriteDACL and delegated password-management permissions. Document why each assignment is necessary; remove unused grants.

**Retest:** Recollect object permissions and verify that unneeded control relationships disappear without breaking legitimate administration.

### Finding C: overbroad Kerberos delegation

**Risk:** Excessive delegation or poor protection of delegating identities can weaken the boundary between an application account and privileged service access.

**Recommendation:** Restrict delegation to necessary SPNs, protect privileged identities, review control over delegating accounts and monitor configuration changes.

**Retest:** Verify that approved service operations continue to work while unauthorized impersonation and service access are refused.

## 10. Outcome, lessons and evidence limits

**Outcome verified:** Both Reset objectives completed on **30 October 2026** in the assigned TryHackMe environment. No production systems were tested.

**What I learned:**

1. **Difficulty labels are subjective.** Once I followed the actual identity relationships, this Hard room felt easier to me than several other AD labs.
2. **Authentication, authorization and service access are different proofs.** A login result cannot substitute for evidence of an effective privilege.
3. **A permission graph is not an exploit log.** Graph findings must be assessed and validated before being called vulnerabilities.
4. **Delegation is a trust boundary, not merely a Kerberos feature.** Its safety depends on who controls the service identity and what the delegation authorizes.
5. **A professional write-up preserves provenance.** Recorded outputs, reconstructed examples and external documentation must never be silently mixed.

### About the full solution

I retain the detailed technical notes for my own lab history. I have **not** placed the challenge-specific credential and escalation sequence, reusable ticket material, passwords or flags in this public article, because the room is still available and not verified retired. An older third-party walkthrough existing online is not permission to publish another complete solution under TryHackMe's current rules.

**Reference:** [TryHackMe Acceptable Use Policy, §8: Content integrity](https://tryhackme.com/legal/acceptable-use-policy).

---

*This post describes authorized CTF training and generalized security methodology. Generic command samples are explicitly labeled and must only be used on systems the reader is authorized to assess.*
