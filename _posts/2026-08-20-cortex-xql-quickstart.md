---
title: "Cortex XQL Quickstart"
date: 2025-08-20 11:00:00 +0000
categories: [Windows, Troubleshooting]
tags: [Cortex, XQL, quickstart]
draft: false
---

## Intro

XQL (XDR Query Language) is used in Cortex XDR/XSIAM's **Query Center / XQL Search** to query security and endpoint datasets. Like NQL, it's a pipeline of clauses chained with `|`, but the syntax differs in several key ways.

## Basic Syntax Rules

- Every query starts with `dataset = <dataset_name>`, optionally preceded by a `config` line.
- Clauses use `filter`, not `where`.
- `in` lists use **parentheses**, not brackets:
  ```
  filter host_name in ("PC001", "PC002")
  ```
- Dataset names are always treated as lowercase regardless of how they're typed.
- Add `config case_sensitive = false` at the top if string matching should ignore case.
- `sort` requires the direction **before** the field name:
  ```
  | sort asc host_name
  ```

## Core Query Structure

```
config timeframe = <duration>
| dataset = <dataset_name>
| filter <condition>
| arrayexpand <array_field>
| alter <new_field> = json_extract(<field>, "$.<json_key>")
| fields <field1>, <field2>
| sort asc|desc <field>
```

## Common Queries

**List all hosts:**
```
config timeframe = 7d
| dataset = endpoints
| fields endpoint_name, ip_address
```

**Get installed software for a specific set of hosts:**
```
config timeframe = 7d
| dataset = host_inventory
| dedup host_name by desc _time
| filter host_name in ("PC001", "PC002", "PC003")
| filter applications != null
| arrayexpand applications
| alter software_name = json_extract(applications, "$.application_name"),
        software_version = json_extract(applications, "$.version")
| fields host_name, software_name, software_version
| sort asc host_name
```

**Filter to specific software (exact match):**
```
| filter software_name in ("Exact Name 1", "Exact Name 2")
```

**Filter to specific software (fuzzy match — recommended, since patch/version suffixes vary between machines):**
```
| filter software_name contains "Product Name"
```

**Find reboot events:**
```
config timeframe = 7d
| dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id in (1074, 6005, 6006, 6008)
| filter agent_hostname in ("PC001", "PC002")
| fields agent_hostname, action_evtlog_event_id, action_evtlog_message, _time
| sort desc _time
```
Event ID reference: `1074` = restart requested, `6005` = system started, `6006` = clean shutdown, `6008` = unexpected shutdown.

## Key Datasets

| Dataset | Contains |
|---|---|
| `endpoints` | Core endpoint/host inventory |
| `host_inventory` | Installed applications, OS details (requires Host Insights / XDR Pro) |
| `xdr_data` | Raw agent telemetry: processes, event logs, network, registry, etc. |
| `application_inventory` | Flatter app inventory on some tenants — check Query Builder if available, it skips the `arrayexpand`/`json_extract` steps |

## Gotchas

- **`host_inventory` requires Host Insights** (XDR Pro add-on). Without it, application data won't populate even if the query is syntactically correct.
- **Known duplication bug** in some versions (host_inventory returning stale/duplicate historical rows) — always `dedup host_name by desc _time` near the top of the query as a safeguard.
- **Nested JSON fields** (like `applications`) must be unpacked with `arrayexpand` + `json_extract` before you can filter or display individual values — you can't filter directly on the raw array field.
- **Exact string matching often fails silently**, returning zero rows with no error, when software names include inconsistent patch/version suffixes across machines. Use `contains` instead of `in` when unsure of exact formatting.
- **Bracket vs. parenthesis confusion**: XQL uses `( )` for `in` lists — the opposite of Nexthink NQL's `[ ]`.

## Building an `in (...)` list from a text file (PowerShell)

```powershell
'(' + ((Get-Content hostnames.txt | Where-Object { $_.Trim() -ne '' } | ForEach-Object { '"' + $_.Trim() + '"' }) -join ',') + ')'
```
