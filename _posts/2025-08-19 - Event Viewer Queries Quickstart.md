---
title: "Event Viewer Queries Quickstart"
date: 2025-08-19 13:00:00 +0000
categories: [Windows, Troubleshooting]
tags: [wevtutil, event viewer, quickstart]
draft: false
---

# Event Viewer Queries Quickstart
The event viewer is one of the most important tools to troubleshoot issues. Whether it's locally on a device or after a severe crash and you pull out the disk, using wevtutil is useful skill.

Extract a time-windowed slice of a `.evtx` log without opening Event Viewer.
 
## 1. Convert your timestamp to UTC
 
`.evtx` always stores `TimeCreated` as UTC internally, regardless of display timezone. Convert your target local time before writing the query, or you'll silently get zero results.
 
```
Local (Paris, CEST, UTC+2): 2026-08-12 13:37:45
UTC:                        2026-08-12 11:37:45
```
 
## 2. Build the XPath time filter
 
```powershell
$utcStart = "2026-08-12T11:35:00.000Z"
$utcEnd   = "2026-08-12T11:50:00.000Z"
 
$query = "*[System[TimeCreated[@SystemTime>='$utcStart' and @SystemTime<='$utcEnd']]]"
```
 
- `@SystemTime` must be UTC, ISO 8601, with trailing `Z`.
- Pad a few minutes either side — don't clip the edges of a sequence.
## 3. Export
 
```powershell
wevtutil epl "C:\Path\To\Source.evtx" "C:\Temp\Windowed.evtx" /lf:true /q:$query
```
 
- `epl` = export log. Works on live channels (`System`, `Application`, `Security`) or offline `.evtx` files.
- `/lf:true` = source is a log **file**, not a live channel name.
- Output is a real `.evtx` — opens normally in Event Viewer.
- **Security log exports need an elevated PowerShell session.**
## 4. Narrow by event ID (cuts noise fast)
 
```powershell
$ids = 4624,4648,4732,4741,4742,4768,4956
$idQuery = ($ids | ForEach-Object { "EventID=$_" }) -join " or "
 
$query = "*[System[TimeCreated[@SystemTime>='$utcStart' and @SystemTime<='$utcEnd'] and ($idQuery)]]"
 
wevtutil epl "C:\Path\To\Security.evtx" "C:\Temp\Security_key.evtx" /lf:true /q:$query
```
 
## 5. Convert to CSV / merge logs
 
```powershell
Get-WinEvent -Path "C:\Temp\Windowed.evtx" |
    Select-Object TimeCreated, Id, ProviderName, LevelDisplayName, Message |
    Export-Csv "C:\Temp\Windowed.csv" -NoTypeInformation -Encoding UTF8
```
 
Merge multiple logs into one timeline:
 
```powershell
$sys = Get-WinEvent -Path "C:\Temp\System_key.evtx" | Select-Object @{N='Source';E={'System'}}, TimeCreated, Id, ProviderName, Message
$sec = Get-WinEvent -Path "C:\Temp\Security_key.evtx" | Select-Object @{N='Source';E={'Security'}}, TimeCreated, Id, ProviderName, Message
 
@($sys + $sec) | Sort-Object TimeCreated | Export-Csv "C:\Temp\Timeline.csv" -NoTypeInformation -Encoding UTF8
```
 
## Gotchas
 
- **Display TZ ≠ source TZ.** `TimeCreated` renders in the *analysis machine's* local zone. Force UTC if unsure:
```powershell
  Get-WinEvent -Path "C:\Temp\log.evtx" | Select-Object @{N='UTC';E={$_.TimeCreated.ToUniversalTime()}}, Id, Message
```
- **Empty results ≠ nothing happened.** Check the file's actual coverage before trusting a blank result:
```powershell
  $e = Get-WinEvent -Path "C:\Temp\log.evtx"
  $e | Sort-Object TimeCreated | Select-Object -First 1 -ExpandProperty TimeCreated
  $e | Sort-Object TimeCreated | Select-Object -Last 1  -ExpandProperty TimeCreated
```
- **`.txt` exports from PowerShell are UTF-16.** Convert if needed: `iconv -f UTF-16LE -t UTF-8 in.txt -o out.txt`.
## Useful Event IDs
 
| ID | Log | Meaning |
|----|-----|---------|
| 1074 | System | Shutdown/restart initiated |
| 7040 | System | Service startup type changed |
| 4624 / 4625 | Security | Logon success / failure |
| 4648 | Security | Logon with explicit credentials |
| 4732 / 4733 | Security | Local group member added / removed |
| 4741 / 4742 | Security | Computer account created / changed |
| 4768 / 4769 | Security | Kerberos ticket issued |
| 4956 | Security | Firewall active profile changed |
| 1000 / 1002 | Application | Application crash / hang |
| 1001 | Application | Windows Error Reporting entry |