---
layout: post
title: "Hack The Box Season 11 private writeups"
date: 2026-07-02 00:30:00 +0100
categories: [HackTheBox, Labs]
tags: [hackthebox, season-11, linux, windows, active-directory, writeups]
description: "Non-spoiler tracker for my private Hack The Box Season 11 writeup repository covering Reactor, DevHub, Connected, Checkpoint, Enigma, Paperwork, MakeSense, Bedside, DarkZeroReturns and Cohort."
---

## Overview

I maintain a private repository for my Hack The Box Season 11 writeups:

```text
https://github.com/ALLAKORI/htb-season-11-writeups
```

The repository is intentionally private while the machines are active. It contains full exploitation notes, CVE and vulnerability mapping, commands, evidence, attack-chain summaries, remediation guidance and redacted flags.

This public post is only a safe tracker. It does not publish active-machine solutions.

## Current Season 11 coverage

| Machine | OS | Difficulty | Status |
| --- | --- | --- | --- |
| Reactor | Linux | Easy | Private writeup completed |
| DevHub | Linux | Medium | Private writeup completed |
| Connected | Linux | Easy | Private writeup completed |
| Checkpoint | Windows | Medium | Private writeup completed |
| Enigma | Linux | Easy | Private writeup completed |
| Paperwork | Linux | Easy | Private writeup completed |
| MakeSense | Linux | Medium | Private writeup completed |
| Bedside | Linux | Medium | Private writeup completed |
| DarkZeroReturns | Windows / Active Directory | Hard | Private writeup completed |
| Cohort | Linux | Easy | Private writeup completed |

## Public vulnerability coverage

The private writeups now include a dedicated CVE/vulnerability section when a named vulnerability was part of the chain.

This table is intentionally non-spoiler: it lists the vulnerability names and the broad stage only, not the exploit payloads or target-specific steps.

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

## Documentation standard

Each private writeup follows the same structure:

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

## Why the details stay private

Hack The Box active-machine material can include live exploitation chains, credentials, target-specific paths and flags. Publishing that publicly before retirement would spoil the lab for other players.

For that reason, the full notes stay in the private repository until the machines retire and the material can be reviewed for safe publication.

## Safe public takeaway

Season 11 is useful practice for chaining realistic issues across Linux services, web applications and Windows Active Directory environments. The main value of the writeups is not just the final flag, but the discipline of documenting:

- what was observed,
- why a pivot made sense,
- which evidence confirmed the path,
- and how the same weakness could be remediated defensively.

