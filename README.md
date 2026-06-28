# volsink 🧠

### Automated Memory Forensics Collection & Processing Framework
**Built by Clint Marsden** · [dfirinsights.com](https://dfirinsights.com) · [TLP Digital Forensics Podcast](https://dfirinsights.com)

![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python) ![Volatility](https://img.shields.io/badge/Volatility-3-red) ![Status](https://img.shields.io/badge/Status-Active-brightgreen) ![License](https://img.shields.io/badge/License-GPLv3-blue)

---

## Why This Exists

If you've done memory forensics with Volatility, you know how it goes: run a plugin, wait for it to finish, run the next one. Some plugins are fast. Some take a while. Either way, you end up staring at a terminal — or worse, you walk away to do something else and come back ten minutes later to find it finished before you left the room. There's no notification or "hey, I'm done." That dead time between a plugin completing and you running the next plugin is pure waste. Multiply it across a dozen plugins, across three or four memory images from different hosts in the same incident, and you've burned hours you don't have while you're still getting questions from the CISO about "what do we know, could you tell me when you'll know more?". But really when you're running an investigation you want to kick off the tool, go away and do other work and come back to output that's already done.

This is why I built volsink. The name comes from the approach: throw the kitchen sink at it. Run everything (or a fast subset) in a scripted way and come back to a folder of structured, case-labelled output ready for analysis. 
## What volsink does

volsink wraps Volatility 3 inside a structured orchestration harness. Point it at a memory image (or a whole folder of images), choose Basic or Advanced scope, and it runs the appropriate plugin set automatically. It writes every result into a separate text file hashing each image for evidence integrity, and isolating failures so one broken plugin never stops a run (this was an issue experienced in early versions which is why the repo was dormant for 2 years)

### The outcome

An analyst who previously spent an hour manually running plugins across four memory images can kick off volsink, walk away, and return to a fully structured, case-labelled output folder ready for analysis. The investigation moves faster and you get output that's ready to review. This is the pattern behind most of what I build: identify where skilled people are spending time on mechanical work, remove that friction, and redirect that capacity toward work that actually requires expertise. volsink is that approach applied to memory forensics.

ote: 
### Core capabilities

| Capability | Detail |
|---|---|
| **OS auto-detection** | Runs `windows.info` / `banners.Banners` to identify Windows, Linux, or macOS — routes to the correct plugin set automatically |
| **Two-tier plugin profiles** | **Basic** covers network, persistence, and process visibility; **Advanced** adds credential extraction, rootkit detection, driver analysis, and memory anomalies |
| **Triple-format output** | Every plugin produces `.txt` (human-readable), `.json` (NDJSON, enriched for SIEM), and `.csv` (flat ingestion) |
| **Batch processing** | Process a single image or an entire evidence folder unattended — one output sub-folder per host |
| **Evidence integrity** | MD5 + SHA-256 streamed in a single pass using Python's `hashlib` — no external tool required. Hashes, Volatility version, and per-plugin results written to `manifest.json` |
| **Timeout capping** | 300s per plugin — a hung scanner is skipped and logged, never fatal |
| **Encoding insulation** | `errors='replace'` on all I/O, console reconfigured to UTF-8 — survives corrupt memory, piped output, and legacy code pages |
| **Environment-agnostic** | Uses `sys.executable` so Volatility runs under whichever interpreter/venv volsink was launched with |

---

## Plugin Coverage

### Windows
**Basic:** `Info`, `PsList`, `PsTree`, `PsScan`, `CmdLine`, `NetScan`, `NetStat`, `DllList`, `SvcScan`, `Sessions`, `Envars`, `HiveList`

**Advanced adds:** `Handles`, `LdrModules`, `Malfind`, `VadInfo`, `Modules`, `ModScan`, `DriverScan`, `DriverIrp`, `DriverModule`, `SSDT`, `Callbacks`, `Threads`, `MutantScan`, `FileScan`, `Privs`, `GetSIDs`, `GetServiceSIDs`, `Hashdump`, `Lsadump`, `Cachedump`, `UserAssist`, `Skeleton_Key_Check`, `HiveScan`, `SymlinkScan`, `MBRScan`, `VadWalk`, `DeviceTree`, `JobLinks`, `VirtMap`

### Linux
**Basic:** `PsList`, `PsTree`, `PsAux`, `Bash`, `Lsof`, `Sockstat`, `Lsmod`, `Envars`

**Advanced adds:** `Malfind`, `Check_syscall`, `Check_modules`, `Check_afinfo`, `Check_creds`, `Elfs`, `LibraryList`, `Maps`, `tty_check`, `Kmsg`, `IOMem`

### macOS
**Basic:** `PsList`, `PsTree`, `Psaux`, `Lsof`, `Netstat`, `Ifconfig`, `Bash`, `Lsmod`

**Advanced adds:** `Malfind`, `Check_syscall`, `Check_sysctl`, `Check_trap_table`, `Kauth_listeners`, `Kauth_scopes`, `Socket_filters`, `Maps`

---

## Quick Start

### Prerequisites
- Python 3.x
- A working Volatility 3 installation (you supply the path to `vol.py`)
- For Linux/macOS images: matching symbol tables pre-staged in Volatility's `symbols/` directory

### Run
```bash
python volsink.py
```

volsink prompts for:
1. Path to `vol.py`
2. Analyst name and case name
3. Evidence path — a single image file **or** a folder of images
4. Scope — Basic or Advanced

### Batch mode naming convention
In folder mode, the **filename (without extension) becomes the hostname**. Name your images after the host:

```
E:\Case-IR-2026-009\
    WS01.raw
    DC01.mem
    FILESRV02.vmem
```

Recognised extensions: `.raw .mem .vmem .dmp .lime .bin .img .dump .vmss .vmsn .core .crash`

---

## Output Structure

```
C:\temp\<YYYYMMDD_HHMMSS>_<CaseName>\
    WS01\
        windows.netscan.NetScan_output.txt
        windows.netscan.NetScan_output.json    ← NDJSON, enriched
        windows.netscan.NetScan_output.csv     ← enriched columns
        ... (one .txt/.json/.csv trio per plugin)
        manifest.json                          ← hashes, vol version, per-plugin status
        errors.txt                             ← only if a plugin failed
    DC01\
        ...
    case_summary.json                          ← all images: OS, hashes, pass/fail summary
```

Every JSON record is enriched with `volsink_case`, `volsink_host`, `volsink_os`, `volsink_plugin`, and `volsink_image_sha256` — making the whole engagement filterable in a single index.

---

## SOF-ELK / Elastic Ingestion

The `*_output.json` files are NDJSON (one record per line) — ready for a Logstash `file` input with a `json` codec, or direct bulk indexing. CSV output covers pipelines that prefer flat tabular ingest.

---

## Evidence Integrity

No external hashing utility required. volsink uses Python's standard-library `hashlib` to compute MD5 and SHA-256 in a single streamed pass. Each image's hashes, byte size, Volatility version, and per-plugin results are recorded in `manifest.json`, forming a reproducible processing record appropriate for evidentiary use.

---

## Symbols & Air-Gapped Environments

Volatility resolves OS structures using symbol tables (ISF files). Windows symbols are downloaded on demand; Linux/macOS symbols must be built or supplied for the specific kernel version.

On an **air-gapped analysis host** — common in IR engagements — this download fails silently, surfacing as failed plugins or an "unknown OS" during detection. If OS detection fails on an otherwise valid image, pre-stage the symbol packs into Volatility's `symbols/` directory before running volsink.

---

## Scope & Design Intent

> volsink **collects and formats** artifacts. It does **not** interpret, classify, or triage findings — that's the examiner's job.

The decision to keep AI/heuristic verdict logic out of the core tool was deliberate. In evidentiary contexts, automated interpretation introduces explainability risk. volsink's job is to get clean, structured, court-defensible output to the analyst as fast as possible. What happens next is a human call.

---

## The Build Journey

The idea is about three years old. It started as a frustration note after one too many engagements spent manually shepherding Volatility plugins in sequence — running one, waiting, running the next, occasionally coming back to a terminal that had quietly finished or quietly crashed ten minutes earlier.

The first real attempt at building it stalled about two years ago on Python environment issues — the same class of problem volsink now protects against. There's some irony in that being what stopped it.

Development resumed using early AI-assisted coding workflows before purpose-built tools like Claude Code existed. That meant a lot of prompt-engineering across tools that weren't designed for agentic build loops. When Claude Code and Gemini matured, the project accelerated — those final stretches compressed months of friction into focused sessions. Claude Code and Google Gemini were used to accelerate implementation throughout; the problem definition, architecture, and forensic methodology decisions are original work.

The hardest lessons along the way:

- **Defensive I/O is non-negotiable in forensics tooling.** Memory images are not clean data. Any assumption about encoding, structure, or completeness will eventually be violated — usually mid-engagement.
- **Timeout handling needs to be architectural, not bolted on.** A scanner that can hang indefinitely is a reliability failure in a time-sensitive discipline. It has to be a first-class design concern.
- **Clear problem definition matters more than ever when AI accelerates implementation.** The bottleneck shifts from writing code to knowing precisely what you're solving and why each design decision was made. That part is still entirely human.

---

## Development Tools

Python · Volatility 3 · Claude Code · Google Gemini

---

## Status

**Active** — in use across home lab. 

---

## Author

**Clint Marsden** — Senior SOC Analyst, DFIR practitioner, AI security researcher.
GCFE · GCFA · GEIR · GSTRT

[dfirinsights.com](https://dfirinsights.com) · [TLP Digital Forensics Podcast](https://dfirinsights.com) · [YouTube](https://youtube.com/@clintmarsden)

---

*volsink is an independent open-source project. It is not affiliated with the Volatility Foundation.*
