# ⚔ IronSentinel-FIM  v1.0.0

**Windows-Native File Integrity Monitoring & Privilege Abuse Detection Agent**  
*By Simon Etim*



## Overview

IronSentinel-FIM is a lightweight, production-grade security agent for Windows 10/11
and Windows Server.  It continuously monitors critical filesystem paths and Registry
keys for unauthorised modifications, detects credential-access attempts and permission
tampering, and generates behaviour-scored alerts mapped to the **MITRE ATT&CK** framework.

It can run interactively in a terminal **or** be installed as a persistent **Windows
Service** that starts automatically at boot with no user session required.

---

## MITRE ATT&CK Coverage

| Technique | Name                                    | Tactic               |
|-----------|-----------------------------------------|----------------------|
| T1003     | OS Credential Dumping                   | Credential Access    |
| T1222     | File & Directory Permissions Modification | Defense Evasion    |
| T1485     | Data Destruction                        | Impact               |
| T1565     | Data Manipulation                       | Impact               |
| T1112     | Modify Registry                         | Defense Evasion      |
| T1547     | Boot/Logon Autostart Execution          | Persistence          |

---

## Features

| Feature                          | Details                                                                 |
|----------------------------------|-------------------------------------------------------------------------|
| Real-time FIM                    | Watchdog-powered; CREATE / MODIFY / DELETE / MOVE events               |
| Registry monitoring              | Polls Run keys, Services, Winlogon, IFEO, LSA for value changes        |
| Credential-access detection      | SAM, SYSTEM, SECURITY, NTDS.dit, LSASS process proximity              |
| ACL/permission tampering         | DACL + owner comparison via pywin32 (SDDL diff)                        |
| Behaviour scoring                | 0–100 score per event; five severity bands                             |
| Rich console alerts              | Colour-coded panels, emojis, process context, MITRE references         |
| Process attribution              | PID, process name, exe path, username, session type (psutil)           |
| Baseline management              | SHA-256 hash + size + mtime + ACL snapshot; auto-update on change      |
| JSON-lines alert log             | One record per line — easy to ingest with SIEM / Elastic / Splunk      |
| Offline integrity check          | Diff current FS state against frozen baseline on demand                |
| Windows Service                  | Auto-start, silent background operation via pywin32                    |
| YAML configuration               | Fully configurable paths, keys, extensions, ignore lists               |

---

## Severity Bands

| Score  | Severity  | Colour  | Typical trigger                                     |
|--------|-----------|---------|-----------------------------------------------------|
| 0–25   | INFO      | Blue    | Minor system file event by known process            |
| 26–50  | LOW       | Green   | New file in monitored path, benign registry change  |
| 51–75  | MEDIUM    | Yellow  | Executable modified, Run key value changed          |
| 76–90  | HIGH      | Orange  | ACL tampered, SYSTEM file replaced                  |
| 91–100 | CRITICAL  | Red     | SAM/NTDS.dit touched, LSASS proximity, T1003 hit   |

---

## Project Structure

```
IronSentinel-FIM/
├── ironsentinel_agent.py   ← Core engine (FIM, registry, alert, baseline)
├── service.py              ← Windows Service wrapper + CLI management
├── config.yaml             ← All runtime configuration
├── requirements.txt        ← Python dependencies
└── README.md               ← This file
```

**Runtime artefacts created automatically:**
```
ironsentinel_baseline.json  ← Filesystem/ACL baseline snapshot
ironsentinel_alerts.jsonl   ← JSON-lines alert log (one record per line)
ironsentinel.log            ← Agent debug/info log
ironsentinel_service.log    ← Service-mode log (written when run as service)
```

---

## Prerequisites

| Requirement          | Notes                                          |
|----------------------|------------------------------------------------|
| Windows 10 / 11 / Server 2019+ | Tested on x64                     |
| Python 3.9 or later  | 3.11+ recommended                              |
| Administrator rights | Required for SAM/SYSTEM paths and ACL reading  |

---

## Installation

### 1. Clone / download

```powershell
git clone https://github.com/cyberchi3f/ironsentinel-fim.git
cd ironsentinel-fim
```

### 2. Install Python dependencies

Open an **elevated** PowerShell prompt (Run as Administrator):

```powershell
pip install -r requirements.txt
```

### 3. Post-install step for pywin32

```powershell
python -m pywin32_postinstall -install
```

This registers the pywin32 COM objects needed for the Windows Service.

---

## Running in Console Mode (Testing / Development)

```powershell
# Start real-time monitoring
python ironsentinel_agent.py --run

# Start with verbose debug output
python ironsentinel_agent.py --run --debug

# Use a custom config file
python ironsentinel_agent.py --run --config C:\SecurityTools\my_config.yaml

# Manually create / refresh baseline
python ironsentinel_agent.py --baseline-create

# Run offline integrity check (no live monitoring)
python ironsentinel_agent.py --baseline-check
```

> **Tip:** Run from an Administrator command prompt to access protected paths.

---

## Installing as a Windows Service

All service commands require an **elevated (Administrator) PowerShell / CMD**.

### Install (registers the service, sets to auto-start)

```powershell
python service.py install
```

### Start the service

```powershell
python service.py start
```

Or via the Services MMC snap-in (`services.msc`) — look for
**"IronSentinel File Integrity Monitor"**.

### Check status

```powershell
python service.py status
```

### Stop the service

```powershell
python service.py stop
```

### Uninstall

```powershell
python service.py uninstall
```

### Debug service code without installing

```powershell
python service.py debug
```

Runs the agent in foreground console mode with debug output — identical
logic to full service mode, safe to Ctrl+C.

---

### Service Notes

| Item              | Detail                                                               |
|-------------------|----------------------------------------------------------------------|
| Service name      | `IronSentinelFIM`                                                    |
| Startup type      | Automatic (starts at boot, before user logon)                        |
| Logon account     | Local System (required for registry and protected file access)       |
| Config path       | Same directory as `service.py`                                       |
| Logs              | `ironsentinel_service.log`, `ironsentinel.log`, `ironsentinel_alerts.jsonl` |
| Console output    | Disabled automatically in service mode                               |

---

## Configuration Reference  (`config.yaml`)

```yaml
monitor:
  paths:           [list of absolute Windows paths to watch]
  registry_keys:   [HIVE\subkey entries to poll]
  extensions:      [file extensions to alert on; empty = all]
  recursive:       true/false
  registry_poll_interval: 10   # seconds

baseline:
  file:            ironsentinel_baseline.json
  auto_save:       true         # update entry after each alert
  hash_algorithm:  sha256       # sha256 | sha512 | md5

alerts:
  min_severity:    INFO         # INFO | LOW | MEDIUM | HIGH | CRITICAL
  log_to_file:     true
  log_file:        ironsentinel_alerts.jsonl
  console_output:  true         # auto-disabled in service mode

ignore:
  paths:      [substring match against event path]
  processes:  [process names to suppress entirely]
  extensions: [file extensions to suppress]

logging:
  level: INFO
  file:  ironsentinel.log
```

---

## Alert Log Format (JSON-Lines)

Each line in `ironsentinel_alerts.jsonl` is a standalone JSON object:

```json
{
  "timestamp":         "2025-08-15T03:47:22.413192",
  "tool":              "IronSentinel-FIM",
  "version":           "1.0.0",
  "event_type":        "FILE_MODIFIED",
  "path":              "C:\\Windows\\System32\\config\\SAM",
  "changes":           ["HASH_CHANGED", "MTIME_CHANGED"],
  "mitre_techniques":  ["T1003", "T1565"],
  "score":             95,
  "severity":          "CRITICAL",
  "severity_emoji":    "🔴",
  "process_context": {
    "pid":           4312,
    "process_name":  "python.exe",
    "exe_path":      "C:\\Python311\\python.exe",
    "username":      "WORKSTATION\\Attacker",
    "session_type":  "INTERACTIVE",
    "cmdline":       "python mimisecrets.py --dump"
  }
}
```

This format is natively ingestible by **Elastic/Filebeat**, **Splunk**,
**Microsoft Sentinel**, and most SIEMs.

---

## Example Console Alert Output

```
╭─ ⚔  IronSentinel-FIM  — CRITICAL Alert ─────────────────────────────────╮
│ 🔴  CRITICAL  FILE_MODIFIED  Score: 95/100                               │
│ Timestamp : 2025-08-15T03:47:22.413192                                   │
│ Path/Key  : C:\Windows\System32\config\SAM                               │
│ Changes   : HASH_CHANGED  ·  MTIME_CHANGED                               │
│ MITRE     : T1003 (OS Credential Dumping)  ·  T1565 (Data Manipulation)  │
│ Process   : python.exe (PID 4312)                                        │
│ User      : WORKSTATION\Attacker [INTERACTIVE]                           │
│ Exe       : C:\Python311\python.exe                                      │
╰──────────────────────────────────────────────────────────────────────────╯

╭─ ⚔  IronSentinel-FIM  — HIGH Alert ────────────────────────────────────╮
│ 🟠  HIGH  REGISTRY_CREATED  Score: 82/100                               │
│ Timestamp : 2025-08-15T03:51:07.001034                                  │
│ Path/Key  : HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run\Backdoor │
│ Changes   : VALUE_ADDED                                                  │
│ MITRE     : T1112 (Modify Registry)  ·  T1547 (Boot Autostart)          │
│ Old Value : N/A                                                          │
│ New Value : C:\Users\Public\svchost32.exe -s                            │
╰─────────────────────────────────────────────────────────────────────────╯

╭─ ⚔  IronSentinel-FIM  — HIGH Alert ────────────────────────────────────╮
│ 🟠  HIGH  FILE_MODIFIED  Score: 78/100                                  │
│ Timestamp : 2025-08-15T03:55:14.782901                                  │
│ Path/Key  : C:\Windows\System32\drivers\etc\hosts                       │
│ Changes   : HASH_CHANGED  ·  SIZE_CHANGED  ·  ACL_CHANGED               │
│ MITRE     : T1222 (Permissions Modification)  ·  T1565 (Data Manip.)    │
│ Process   : cmd.exe (PID 9104)                                          │
│ User      : DOMAIN\jsmith [INTERACTIVE]                                 │
│ Exe       : C:\Windows\System32\cmd.exe                                 │
╰─────────────────────────────────────────────────────────────────────────╯
```

---

## Integrating with a SIEM

### Elastic / Filebeat

Add a Filebeat `log` input pointing at `ironsentinel_alerts.jsonl`:

```yaml
filebeat.inputs:
  - type: log
    paths: ["C:/SecurityTools/IronSentinel-FIM/ironsentinel_alerts.jsonl"]
    json.keys_under_root: true
    json.add_error_key: true
```

### Splunk Universal Forwarder

```ini
[monitor://C:\SecurityTools\IronSentinel-FIM\ironsentinel_alerts.jsonl]
sourcetype = ironsentinel_fim
index      = security
```

### Microsoft Sentinel

Forward `ironsentinel_alerts.jsonl` via the **Azure Monitor Agent** Custom
Text Log connector, or use the **Custom Log Ingestion API**.

---

## Troubleshooting

| Symptom                              | Fix                                                                 |
|--------------------------------------|---------------------------------------------------------------------|
| `PermissionError` on SAM / SYSTEM    | Run as Administrator                                                |
| `ImportError: No module named win32` | `pip install pywin32` then `python -m pywin32_postinstall -install` |
| Service won't start                  | Check `ironsentinel_service.log` for exceptions                     |
| No ACL monitoring                    | pywin32 not installed; ACL features silently disabled               |
| Very high event volume               | Narrow `paths`, restrict `extensions`, or add to `ignore.paths`    |
| Registry monitor missing changes     | Decrease `registry_poll_interval` (e.g. 5 seconds)                 |
| `watchdog.observers.api.ObservedWatch` error | Path no longer exists at start; check config paths         |

---

## Security Hardening Notes

- Store `ironsentinel_baseline.json` on **read-only media** or a remote share
  to prevent an attacker from updating the baseline to hide their tracks.
- Protect `config.yaml` and all log files with restrictive ACLs
  (`icacls ironsentinel_alerts.jsonl /grant Administrators:F /inheritance:r`).
- Consider running the service under a dedicated low-privilege account with
  explicit `SeSecurityPrivilege` rather than `LOCALSYSTEM` in production.
- Rotate the alert log periodically; it is append-only and can grow large on
  busy systems.

---

## Name Change Notes

> **ShadowGuard-FIM** → **IronSentinel-FIM**

### Why IronSentinel?

- **Iron** — evokes strength, persistence, Windows kernel / NTFS hardness, and
  the "iron fist" metaphor for strict integrity enforcement.
- **Sentinel** — universally understood security term; clear function signal.
- The name is **product-line scalable**: IronSentinel-FIM, IronSentinel-EDR,
  IronSentinel-NDR could form a DexAShield product suite.

### Alternative Names Considered

| Name                  | Pros                                  | Cons                              |
|-----------------------|---------------------------------------|-----------------------------------|
| **ForgeWatch-FIM**    | "Forge" echoes Windows internals      | Less intuitive security meaning   |
| **CrestGuard-FIM**    | Premium/heraldic feel, suits brand    | "Crest" is less direct            |
| **WinGarrison-FIM**   | Explicit Windows focus                | Slightly verbose                  |
| **IronSentinel-FIM**  | ✔ Strong, memorable, brand-extensible | ✔ Chosen                          |

### DexAShield Branding Suggestions

1. **Tagline:** *"Driven by Technology, Defined by Security"* — already fits.
2. **Product line naming:** `DexAShield IronSentinel™ FIM` for formal docs;
   `IronSentinel-FIM` as the short/code name.
3. **GitHub repo:** `dexashield/ironsentinel-fim`
4. **Colour palette:** Port the crimson / navy from existing DexAShield assets
   to the console banner (currently cyan; trivial to change in `_print_banner`).

---

## License

MIT — see `LICENSE` (add your own).  
Commercial use permitted; attribution appreciated.

---

*IronSentinel-FIM is provided for authorised defensive security use only.
Always obtain written permission before deploying on systems you do not own.*
