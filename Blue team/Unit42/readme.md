# 🟢 Hack The Box — Unit42

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black">
  <img src="https://img.shields.io/badge/Difficulty-Very%20Easy-9FEF00?style=for-the-badge">
  <img src="https://img.shields.io/badge/Category-Windows%20Forensics-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/Focus-Sysmon%20%7C%20DFIR-red?style=for-the-badge">
</p>

---

## 📌 Machine Overview

**Unit42** is a Windows forensics challenge focused on analyzing Sysmon logs to investigate an UltraVNC-related malware campaign.

The investigation uses Windows Event Viewer to examine process creation, file creation, timestamp manipulation, DNS queries, network connections, and process termination.

The objective is to reconstruct the initial infection activity, identify the malicious executable, determine how it was delivered, and extract relevant indicators of compromise.

### Scenario

The investigation is based on research by Palo Alto Networks Unit 42 into a campaign involving a backdoored UltraVNC variant.

### Artifact Provided

- **File:** `UltraVNC_UNIT42.zip`
- **SHA1:** `1D8AC45395551187EAF23793CE525056C4136D6E`
- **Log:** `Microsoft-Windows-Sysmon-Operational.evtx`
- **Total Sysmon events:** 169

### Tools Used

- Windows Event Viewer
- VirusTotal — optional hash reputation analysis

---

# 🔎 1. Initial Analysis

Extract the provided archive using the password:

    hacktheblue

The archive contains the Sysmon operational event log:

    Microsoft-Windows-Sysmon-Operational.evtx

Open the `.evtx` file in Windows Event Viewer.

The log contains 169 events. Rather than inspecting every event individually, filter the log by the Event IDs relevant to the investigation.

| Event ID | Purpose |
|---|---|
| 1 | Process creation |
| 2 | File creation time changed |
| 3 | Network connection |
| 5 | Process termination |
| 11 | File creation |
| 22 | DNS query |

**Important:** Event Viewer may display timestamps in the local configured timezone. When precise correlation matters, inspect the event details and the `UtcTime` field where available.

---

# 📂 2. Count File Creation Events

### Question

How many events have Event ID 11?

### Methodology

1. Open the Sysmon log in Event Viewer.
2. Select **Filter Current Log**.
3. Enter `11` in the Event IDs field.
4. Apply the filter and inspect the resulting event count.

Event ID 11 records file creation activity and helps identify files dropped by suspicious processes.

### Answer

    56

---

# 🦠 3. Identify the Initial Malicious Executable

### Question

What malicious process infected the victim's system?

### Methodology

Filter the log for:

    Event ID: 1

Event ID 1 records process creation and provides details such as the executable path, command line, parent process, user context, and file hashes.

The filtered log contains only six process creation events, making it practical to inspect each event.

Focus on these fields:

- `Image`
- `ParentImage`
- `CommandLine`
- `Description`
- `Product`
- `Hashes`

One executable stands out because it has a **double `.exe` extension** and runs directly from the user's Downloads directory.

The suspicious path is:

    C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe

The filename and execution location are inconsistent with the executable's apparent product description. Searching the file's hash on VirusTotal provides additional context and associates the binary with a malicious UltraVNC-related RAT.

### Answer

    C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe

---

# ☁️ 4. Identify the Malware Delivery Service

### Question

Which cloud drive was used to distribute the malware?

### Methodology

The goal is to correlate DNS activity with the creation of the malicious executable.

First, filter for:

    Event ID: 22

Event ID 22 records DNS queries and identifies the process performing the query and the domain being resolved.

Inspect the DNS events around the time of the suspected infection. One event reveals activity associated with Dropbox at approximately `03:41:26`.

Next, correlate the DNS activity with file creation events by inspecting Event ID 11 entries around the same time.

The nearby file creation activity and the DNS query support the conclusion that Dropbox was likely used to deliver the malicious executable.

### Answer

    Dropbox

---

# 🕒 5. Identify the Timestamp Used for Timestomping

### Question

What timestamp was applied to a PDF file by the malicious process?

### Methodology

Filter the log for:

    Event ID: 2

Event ID 2 records changes to file creation timestamps. This can reveal **timestomping**, a defense-evasion technique used to make files appear older than they actually are.

Inspect the events and identify the PDF file whose creation timestamp was modified by the malicious executable.

The timestamp applied to the PDF file was:

### Answer

    2024-01-14 08:10:06

---

# 📄 6. Locate `once.cmd`

### Question

Where was `once.cmd` created on disk?

### Methodology

Filter the log for:

    Event ID: 11

Use Event Viewer's **Find** function to search for:

    once.cmd

The search returns two relevant results: one associated with `msiexec` and another associated with the suspicious executable.

Focus on the event where the `Image` field identifies:

    Preventivo24.02.14.exe.exe

The event reveals the path where the malicious process created `once.cmd`.

### Answer

    C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd

---

# 🌐 7. Identify the DNS Domain Queried by the Malware

### Question

Which dummy domain did the malicious process attempt to reach?

### Methodology

Return to:

    Event ID: 22

Inspect the `Image` field to identify DNS queries initiated by the malicious executable.

The relevant event shows that the process attempted to resolve a dummy domain, likely as part of checking internet connectivity.

### Answer

    www.example.com

---

# 🔌 8. Identify the Destination IP Address

### Question

Which IP address did the malicious process attempt to contact?

### Methodology

Filter the log for:

    Event ID: 3

Event ID 3 records network connections and includes fields such as:

- `Image`
- `UtcTime`
- `Protocol`
- `DestinationIp`
- `DestinationPort`
- `Initiated`

Use Event Viewer's **Find** function to search for:

    Preventivo

Inspect the matching event and verify that the process image corresponds to the malicious executable.

The destination IP recorded in the event is:

### Answer

    93.184.216.34

---

# 🛑 9. Determine When the Malicious Process Terminated

### Question

When did the malicious process terminate itself?

### Methodology

Filter the log for:

    Event ID: 5

Event ID 5 records process termination.

Inspect the `Image` field and locate the event associated with the malicious executable:

    Preventivo24.02.14.exe.exe

The event timestamp indicates when the process terminated.

### Answer

    2024-02-14 03:41:58

---

# 🧭 10. Investigation Timeline

The relevant events can be organized into the following sequence.

| Time / Stage | Event |
|---|---|
| Initial access | Victim obtains a malicious executable through a cloud-hosted delivery mechanism |
| Around 03:41:26 | DNS activity associated with Dropbox is observed |
| Infection | `Preventivo24.02.14.exe.exe` executes from the user's Downloads directory |
| Post-execution | The malicious process creates additional files, including `once.cmd` |
| Post-execution | File creation timestamps are modified, including a PDF timestamp set to `2024-01-14 08:10:06` |
| Network activity | The malicious process queries `www.example.com` |
| Network activity | A connection attempt to `93.184.216.34` is recorded |
| `2024-02-14 03:41:58` | The malicious process terminates |

**Timeline note:** The source identifies the Dropbox DNS event as occurring around `03:41:26`, but does not provide a complete UTC-normalized timestamp for every event. Correlate the original event details before treating all times as belonging to the same timezone.

---

# 🚨 11. Indicators of Compromise

## Malicious Executable

    Preventivo24.02.14.exe.exe

## Executable Path

    C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe

## Dropped File

    once.cmd

## Dropped File Path

    C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd

## DNS Domain

    www.example.com

## Destination IP

    93.184.216.34

## Timestamp Applied to PDF

    2024-01-14 08:10:06

## Suspected Delivery Service

    Dropbox

## Process Termination

    2024-02-14 03:41:58

---

# 🧩 12. Sysmon Investigation Workflow

The investigation follows this practical workflow:

    Open Sysmon EVTX
           │
           ▼
    Filter Event ID 1
           │
           ▼
    Identify suspicious executable
           │
           ▼
    Check file path and metadata
           │
           ▼
    Search hash reputation if needed
           │
           ▼
    Filter Event ID 22
           │
           ▼
    Correlate DNS activity with file creation
           │
           ▼
    Filter Event ID 11
           │
           ▼
    Identify files dropped by the malware
           │
           ▼
    Filter Event ID 2
           │
           ▼
    Identify timestomping activity
           │
           ▼
    Filter Event ID 3
           │
           ▼
    Extract destination IP
           │
           ▼
    Filter Event ID 5
           │
           ▼
    Determine process termination time
           │
           ▼
    Build the incident timeline and IOC list

---

# 🛡️ 13. Security Findings

### Suspicious Execution

The executable `Preventivo24.02.14.exe.exe` ran from the user's Downloads directory and used a double `.exe` extension, making it suspicious.

### Malware Delivery

DNS and file creation events support the conclusion that Dropbox was likely involved in delivering the malicious executable.

### File Dropping

The malicious process created `once.cmd` in a directory under the user's AppData roaming profile.

### Defense Evasion

Event ID 2 revealed timestamp manipulation affecting a PDF file. This is consistent with timestomping behavior intended to obscure the apparent age of a file.

### Network Activity

The process queried `www.example.com` and attempted to connect to `93.184.216.34`.

### Process Lifecycle

Event ID 5 established that the malicious process terminated at `2024-02-14 03:41:58`.

---

# 📝 14. Investigation Summary

The investigation began with the provided Sysmon operational log, which contained 169 events.

Filtering Event ID 11 established that the log contained 56 file creation events. Filtering Event ID 1 then revealed the suspicious executable:

    C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe

Its double `.exe` extension and execution from the Downloads directory warranted further investigation. The source writeup reports that hash reputation checks supported classifying the executable as a malicious UltraVNC-related RAT.

DNS and file creation events were correlated to identify Dropbox as the likely delivery service.

Further analysis identified additional malicious activity:

- Creation of `once.cmd` under the user's AppData roaming profile.
- Timestomping of a PDF file to `2024-01-14 08:10:06`.
- A DNS query for `www.example.com`.
- A network connection attempt to `93.184.216.34`.
- Process termination at `2024-02-14 03:41:58`.

By correlating Sysmon Event IDs 1, 2, 3, 5, 11, and 22, the investigation reconstructed the key stages of the infection and identified several artifacts useful for detection and incident response.

---

# 🎯 15. Key Takeaways

- **Event ID 1:** Identify suspicious process execution and inspect parent process, command line, path, and hashes.
- **Event ID 2:** Detect file creation timestamp manipulation.
- **Event ID 3:** Identify destination IP addresses and ports associated with process network activity.
- **Event ID 5:** Determine when suspicious processes terminate.
- **Event ID 11:** Identify files created by suspicious processes.
- **Event ID 22:** Identify DNS queries and correlate them with other activity.
- Correlating multiple event types provides stronger evidence than investigating isolated events.
- Suspicious filenames and execution paths are useful leads, but should be corroborated with process metadata and threat intelligence.
- Cloud storage services can be abused to distribute malware.
- Timeline analysis helps connect initial delivery, execution, file dropping, network activity, and termination.

---

# 🛠️ Skills Demonstrated

- Windows Event Log Analysis
- Sysmon Log Analysis
- Process Creation Analysis
- File Creation Analysis
- DNS Query Analysis
- Network Connection Analysis
- Timestomping Detection
- Malware Investigation
- IOC Identification
- Timeline Reconstruction
- Contextual Analysis
- DFIR Methodology

---

# 🏁 Final Infection Chain

    Cloud-Based Delivery
           │
           ▼
    Dropbox Identified as Likely Delivery Service
           │
           ▼
    Malicious Executable Downloaded
           │
           ▼
    Preventivo24.02.14.exe.exe
           │
           ▼
    Malicious Process Execution
           │
           ├──────────────► File Creation: once.cmd
           │
           ├──────────────► Timestamp Manipulation
           │
           ├──────────────► DNS Query: www.example.com
           │
           └──────────────► Network Activity: 93.184.216.34
                                    │
                                    ▼
                           Process Termination
                           2024-02-14 03:41:58

---

**Machine:** Unit42  
**Platform:** Hack The Box  
**Difficulty:** Very Easy  
**Category:** Windows Forensics / DFIR  
**Primary Artifact:** `Microsoft-Windows-Sysmon-Operational.evtx`
