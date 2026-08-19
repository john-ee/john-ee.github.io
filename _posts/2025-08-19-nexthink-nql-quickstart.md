---
title: "NexThink NQL Quickstart"
date: 2025-08-19 13:00:00 +0000
categories: [Windows, Troubleshooting]
tags: [NexThink, NQL, quickstart]
draft: false
---

## Intro

NQL (Nexthink Query Language) is used in the **Investigations** module to query device, user, and application data. Queries are built as a pipeline of clauses chained with `|`.

## Basic Syntax Rules

- Every query starts with a **source** (a table/entity, e.g. `devices`, `package.installed_packages`).
- Every clause after the first line must start with `|`.
- String comparisons with `in` / `!in` use **square brackets**, not parentheses:
  ```
  where device.name in ["PC001", "PC002"]
  ```
- Use straight quotes (`"`) only — curly/smart quotes (from pasting out of Word or web pages) will silently break the parser downstream.

## Core Query Structure

```
<source>
| where <condition>
| include <related_table>
| summarize <aggregation> by <fields>
| sort <field> asc|desc
```

Note: the `list` clause is inconsistent in some editor versions. If it's not being recognized, use `summarize count() by <fields>` instead — it returns one row per unique combination of the `by` fields, which achieves the same result as a `list`.

## Common Queries

**List all devices:**
```
devices
| list device.name, device.entity
```

**Get installed software for a specific set of devices:**
```
package.installed_packages
| where device.name in ["PC001", "PC002", "PC003"]
| summarize c = count() by device.name, package.name, package.version
| sort device.name asc
```

**Filter to specific software across all devices:**
```
package.installed_packages
| where package.name in ["Software Name 1", "Software Name 2"]
| summarize c = count() by device.name, package.name, package.version, device.entity
| sort device.name asc
```

**Include last logged-in user:**
```
package.installed_packages
| where device.name in ["PC001", "PC002"]
| summarize c = count() by device.name, package.name, package.version, device.login.last_login_user_name
```

**Get per-application resource and network usage for a single device (7 days)**
```
execution.events during past 7d
| where device.name == "PC001"
| summarize execution_duration_ = execution_duration.sum(), CPU_usage_time = cpu_time.sum(), Memory_usage = memory.avg.sum(), Incoming_traffic_ = incoming_traffic.sum(), Outgoing_traffic_ = outgoing_traffic.sum(), Incoming_throughput_ = incoming_throughput.avg(), Outgoing_throughput_ = outgoing_throughput.avg(), Connection_establishment = connection_establishment_time.avg(), Established_connections = number_of_established_connections.sum(), Connections_where_host_was_not_available = number_of_no_host_connections.sum(), Connections_where_service_was_not_available = number_of_no_service_connections.sum(), Rejected_connections = number_of_rejected_connections.sum(), Page_faults = number_of_page_faults.sum() by application.name, binary.product_name, binary.name, binary.version
| sort CPU_usage_time desc
```

## Key Tables

| Table | Contains |
|---|---|
| `devices` | Core device inventory |
| `package.installed_packages` | Installed software per device (scanned hourly) |
| `device.login` | Login/session data, including last logged-in user |
| `device_performance.boots` | Boot/startup events |

## Gotchas

- **Software scan frequency**: Nexthink scans installed packages roughly once per hour — data isn't real-time to the second.
- **Bracket vs. parenthesis confusion**: NQL uses `[ ]` for `in`/`!in` lists. This is the opposite of SQL and of XQL (Cortex), so double-check when switching between tools.
- **User-based software licensing** (e.g. named-user apps) doesn't map cleanly onto device-based package tables — you may need a separate dashboard/query joining user and device data instead of a single query.
- **Autocomplete is not exhaustive.** The dropdown only suggests common next clauses; a valid keyword not appearing in the list doesn't mean it's invalid — but if it stays flagged as an error after typing it, it usually is.

## Building an `in [...]` list from a text file (PowerShell)

```powershell
'[' + ((Get-Content usernames.txt | Where-Object { $_.Trim() -ne '' } | ForEach-Object { '"' + $_.Trim() + '"' }) -join ',') + ']'
```

From Active Directory OU:
```powershell
$OU = "OU=Workstations,DC=contoso,DC=com"
Write-Output ('[' + ((Get-ADComputer -SearchBase $OU -Filter * | ForEach-Object { '"' + $_.Name + '"' }) -join ',') + ']')
```
