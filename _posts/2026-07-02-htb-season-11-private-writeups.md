---
layout: post
title: "Hack The Box Season 11 writeups"
date: 2026-07-02 00:30:00 +0100
categories: [HackTheBox, Labs]
tags: [hackthebox, season-11, linux, windows, active-directory, writeups]
description: "Tracker for my Hack The Box Season 11 writeup repository covering Reactor, DevHub, Connected, Checkpoint, Enigma, Paperwork, MakeSense, Bedside, DarkZeroReturns, Cohort and DanglingTree."
---

## Overview

My Hack The Box Season 11 writeups live in a public repository:

```text
https://github.com/ALLAKORI/htb-season-11-writeups
```

The season is over and the machines have retired, so the full notes are now public. The repository contains exploitation notes, CVE and vulnerability mapping, commands, evidence, attack-chain summaries, remediation guidance and redacted flags.

## Current Season 11 coverage

| Machine | OS | Difficulty | Status |
| --- | --- | --- | --- |
| Reactor | Linux | Easy | Writeup completed |
| DevHub | Linux | Medium | Writeup completed |
| Connected | Linux | Easy | Writeup completed |
| Checkpoint | Windows | Medium | Writeup completed |
| Enigma | Linux | Easy | Writeup completed |
| Paperwork | Linux | Easy | Writeup completed |
| MakeSense | Linux | Medium | Writeup completed |
| Bedside | Linux | Medium | Writeup completed |
| DarkZeroReturns | Windows / Active Directory | Hard | Writeup completed |
| Cohort | Linux | Easy | Writeup completed |
| DanglingTree | Windows / Active Directory | Medium | Writeup completed |

## Vulnerability coverage

Each writeup includes a dedicated CVE/vulnerability section when a named vulnerability was part of the chain.

This overview lists the vulnerability names and the broad stage only, not the exploit payloads or target-specific steps.

| Machine | CVEs / vulnerability names covered | Broad stage |
| --- | --- | --- |
| Reactor | CVE-2025-55182 React2Shell / React Server Components RCE; CVE-2025-66478 Next.js RSC advisory; Node.js Inspector misconfiguration | Initial access; privilege escalation |
| DevHub | CVE-2026-23744 MCPJam Inspector Remote Code Execution; exposed Jupyter token; hardcoded OPSMCP API key and hidden admin tool | Initial access; lateral movement; privilege escalation |
| Connected | CVE-2025-57819 FreePBX Endpoint Manager SQL injection to RCE; Incron / DAHDI local misconfiguration | Initial access; privilege escalation |
| Checkpoint | Active Directory object recovery, ACL abuse, dMSA / BadSuccessor abuse, memory forensics, Pass-the-Hash | Active Directory chain |
| Enigma | CVE-2026-38751 OpenSTAManager module-upload RCE PoC; related CVE-2025-69212 OpenSTAManager command-injection context; CVE-2026-27626 OliveTin password argument command injection | Foothold; privilege escalation |
| Paperwork | LPD command injection; PJL path traversal and arbitrary file write; `SCM_RIGHTS` file descriptor leak; password reuse | Initial access; user escalation; privilege escalation |
| MakeSense | Hardcoded client-side encryption key; stored XSS; WordPress administrator account creation; PHP reverse shell; credential reuse; internal OCR-to-PHP root RCE | Initial access; lateral movement; privilege escalation |
| Bedside | CVE-2025-64512 pdfminer.six pickle deserialization RCE; internal development-server path traversal; PyTorch checkpoint deserialization through `torch.load()`; shared datastore permission boundary failure | Initial access; container-to-host pivot; privilege escalation |
| DarkZeroReturns | CVE-2026-33937 Handlebars AST injection RCE; related CVE-2021-23369 Handlebars RCE context; Gitea Actions workflow trust abuse; Kerberos/AD ACL abuse; forest-trust ExtraSID and DCSync chain | Initial access; lateral movement; Linux root; domain and cross-forest compromise |
| Cohort | CVE-2026-39987 Marimo pre-auth Terminal WebSocket RCE; SSRF loopback bypass through `127.1`; nginx/vhost pivot to Marimo; CVE-2026-41651 PackageKit TOCTOU / Pack2TheRoot | Initial access; privilege escalation |
| DanglingTree | CVE-2026-23760 SmarterMail password reset authentication bypass; CVE-2026-24423 SmarterMail ConnectToHub RCE; Windows Admin Center pivoting; SmarterMail backup recovery; DPAPI credential decryption; ForceChangePassword ACL abuse; AD CS ESC1 certificate impersonation | Initial access; lateral movement; domain compromise |

## Documentation standard

Each writeup follows the same structure:

| Section | Purpose |
| --- | --- |
| Machine information | Quick platform, OS, difficulty and attack-focus context |
| Summary | Short executive explanation of the compromise path |
| CVEs and vulnerabilities used | Named vulnerabilities, affected products and where they fit in the chain |
| Exploitation steps | Reproducible notes with commands and evidence |
| Attack-chain summary | Compact end-to-end view of the compromise |
| Lessons learned | What the lab reinforced technically |
| Remediation | Defensive guidance mapped to the weaknesses found |
| Flags | Redacted user/root flag status |

## Season ended, repository now public

Hack The Box active-machine material includes live exploitation chains, credentials, target-specific paths and flags. Full notes were kept private while the machines were active to avoid spoiling the lab for other players.

Season 11 has now ended, the machines have retired, and the material has been reviewed for safe publication. The repository is public.

## Takeaway

Season 11 is useful practice for chaining realistic issues across Linux services, web applications and Windows Active Directory environments. The main value of the writeups is not just the final flag, but the discipline of documenting:

- what was observed,
- why a pivot made sense,
- which evidence confirmed the path,
- and how the same weakness could be remediated defensively.
