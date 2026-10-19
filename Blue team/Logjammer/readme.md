
<div align="center">

# 🕵️ HTB Sherlock: Logjammer

![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Category](https://img.shields.io/badge/Category-Windows%20DFIR-blue)
![Focus](https://img.shields.io/badge/Focus-Event%20Log%20Analysis-red)

**Windows Event Log Analysis | Splunk | Digital Forensics & Incident Response**

</div>

---

## 📌 Overview

Logjammer is an Easy-difficulty HTB Sherlock focused on Windows Event Log analysis using Splunk. The investigation examines suspicious activity associated with the user `CyberJunkie`, including firewall rule modifications, audit policy changes, scheduled task creation, antivirus detections, PowerShell activity, and event log clearing.

The objective is to reconstruct the user's actions and identify evidence of potentially malicious activity from Windows event logs.

## 🧰 Tools Used

- **Splunk** — Event indexing, searching, filtering, and correlation.
- **EvtxECmd** — Windows Event Log conversion to JSON.
- **Windows Event Viewer** — Manual inspection of audit policy details.
- **CyberChef** — Decoding HTML-encoded scheduled task XML.

## 📂 Evidence

| Artifact | Details |
|---|---|
| Archive | `logjammer.zip` |
| MD5 | `2ba794c406aee485186a959c840125af` |
| Primary evidence | Windows EVTX event logs |
| Analysis platform | Splunk |

## 🔎 Investigation

### 1. Event Log Preparation

The Windows Event Logs were converted into JSON using Eric Zimmerman's EvtxECmd:

    EvtxECmd.exe -d ./SP/ --sync -json events.json

The resulting JSON data was imported into a new Splunk index named `startingpoint`.

This allowed searches across the collected events using fields such as `Provider`, `EventId`, `UserName`, and `TimeCreated`.

### 2. User Login

The investigation began by identifying the user's successful logon event.

**Splunk SPL query:**

    index="startingpoint" CyberJunkie Provider="Microsoft-Windows-Security-Auditing" EventId=4624

The write-up reports a successful login at:

    27/03/2023 14:37:09

This timestamp was used as a reference point for investigating subsequent activity.

### 3. Firewall Rule Modification

Windows Firewall events were searched to identify newly created rules.

**Splunk SPL query:**

    index="startingpoint" Provider="Microsoft-Windows-Windows Firewall With Advanced Security" EventId="2004"

Three results were identified:

| Rule | Direction | Action | Protocol |
|---|---|---|---|
| Metasploit C2 Bypass | Outbound | Pass | TCP:4444 |
| Microsoft Edge | Outbound | Block | All protocols |
| Microsoft Edge | Inbound | Block | All protocols |

The suspicious rule was **Metasploit C2 Bypass**, allowing outbound TCP traffic on port `4444`. This is consistent with a potential command-and-control communication path.

To display the firewall event details in a table:

    index="startingpoint" Provider="Microsoft-Windows-Windows Firewall With Advanced Security" EventId="2004"
    | table TimeCreated, PayloadData1, PayloadData3, PayloadData4, PayloadData5
    | rename TimeCreated as Time, PayloadData1 as Name, PayloadData2 as Direction, PayloadData4 as Action, PayloadData5 as Protocol

### 4. Audit Policy Modification

The next step was to investigate changes to the Windows audit policy.

**Splunk SPL query:**

    index="startingpoint" audit policy

The search identified Security Event ID `4719`, indicating that the system audit policy had changed.

The event was inspected manually in Windows Event Viewer because EvtxECmd did not map the relevant subcategory value into the JSON output.

**Recovered details:**

- Category: `Object Access`
- Subcategory: `Other Object Access Events`
- Change: `Success Added`
- Subject: `SYSTEM`
- Account: `DESKTOP-887GK2L$`

The audit policy modification is significant because changes to auditing can affect which activities are recorded by the system.

### 5. Scheduled Task Creation

A search for scheduled-task activity associated with the user identified a task named `HTB-AUTOMATION`.

**Splunk SPL query:**

    index="startingpoint" scheduled CyberJunkie

The task's XML was HTML-encoded, so it was decoded using CyberChef.

**Recovered task configuration:**

| Field | Value |
|---|---|
| Task name | `HTB-AUTOMATION` |
| Author | `DESKTOP-887GK2L\CyberJunkie` |
| Description | `practice` |
| Schedule | Daily |
| Start boundary | `2023-03-27T09:00:00` |
| Run level | `LeastPrivilege` |
| Executable/script | `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1` |
| Arguments | `-A cyberjunkie@hackthebox.eu` |

The scheduled task establishes that a PowerShell script was configured to run through Windows Task Scheduler. The recovered path and arguments provide useful evidence for further investigation.

### 6. Windows Defender Detection

Windows Defender events were searched to identify detected threats and the actions taken.

**Splunk SPL query:**

    index="startingpoint" Provider="Microsoft-Windows-Windows Defender"

The investigation identified Event ID `1117`, reporting a high-severity detection.

**Detection details:**

- Detected file: `C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip`
- Threat name: `HackTool:MSIL/SharpHound!MSR`
- Action: `Quarantine`
- Severity: `HIGH`

SharpHound is associated with Active Directory data collection. Its presence is relevant in a security investigation, although the detection alone does not establish the full purpose of its use on the host.

### 7. PowerShell Activity

PowerShell operational events were searched to recover commands associated with the user.

**Splunk SPL query:**

    index="startingpoint" Provider="Microsoft-Windows-PowerShell" UserName="DESKTOP-887GK2L\\CyberJunkie"

The search returned 11 events. One relevant record, Event ID `4103`, occurred at `2023-03-27 14:58:33`.

**Recovered command:**

    Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1

The command calculated an MD5 hash of the scheduled PowerShell script. This links the script identified in the scheduled task configuration to subsequent PowerShell activity.

### 8. Event Log Clearing

The final stage investigated whether event logs had been cleared.

**Splunk SPL query:**

    index="startingpoint" EventId="104"

The result indicated that the following log had been cleared:

    Microsoft-Windows-Windows Firewall With Advanced Security/Firewall

Clearing a log can remove useful forensic evidence and should be correlated with surrounding events to establish intent and timing.

## 🚨 Key Findings

| Finding | Result |
|---|---|
| User login | `27/03/2023 14:37:09` |
| Suspicious firewall rule | `Metasploit C2 Bypass` |
| Firewall direction | Outbound |
| Allowed traffic | TCP port `4444` |
| Audit policy subcategory | `Other Object Access Events` |
| Scheduled task | `HTB-AUTOMATION` |
| Scheduled script | `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1` |
| Task arguments | `-A cyberjunkie@hackthebox.eu` |
| Defender detection | `SharpHound` |
| Detected archive | `C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip` |
| Defender action | Quarantine |
| PowerShell command | `Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1` |
| Cleared log | Windows Firewall With Advanced Security/Firewall |

## 🧭 Attack Timeline

| Stage | Activity |
|---|---|
| 1 | `CyberJunkie` successfully logs in at `14:37:09` on 27 March 2023. |
| 2 | A suspicious outbound firewall rule named `Metasploit C2 Bypass` is identified, allowing TCP port `4444`. |
| 3 | The system audit policy is modified, adding success auditing for `Other Object Access Events`. |
| 4 | A scheduled task named `HTB-AUTOMATION` is configured to execute `Automation-HTB.ps1`. |
| 5 | Windows Defender detects `SharpHound-v1.1.0.zip` and quarantines it. |
| 6 | PowerShell logs record a `Get-FileHash` command targeting the scheduled script. |
| 7 | Event ID `104` indicates that the Windows Firewall log was cleared. |

*The source write-up does not provide exact timestamps for every activity above, so the sequence is an investigative summary rather than a fully timestamped chronology.*

## 🎯 MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|
| Scheduled Task/Job: Scheduled Task | T1053.005 | `HTB-AUTOMATION` was created to execute a PowerShell script. |
| Impair Defenses: Disable or Modify System Firewall | T1562.004 | A suspicious firewall rule allowed outbound TCP traffic on port `4444`. |
| Impair Defenses: Indicator Blocking | T1562.006 | The audit policy and firewall logs were modified or cleared; this mapping is contextual and not a definitive classification of every action. |
| Indicator Removal: Clear Windows Event Logs | T1070.001 | Event ID `104` recorded the clearing of the Windows Firewall log. |
| Security Software Discovery | T1518.001 | Windows Defender detected a SharpHound archive; this detection alone does not prove the user executed it. |

*These ATT&CK mappings are analytical classifications based on the reported evidence. They are not additional findings explicitly listed in the source write-up.*

## 🛡️ Defensive Recommendations

- Forward Windows Event Logs to a centralized, access-controlled SIEM.
- Alert on creation of firewall rules allowing unexpected outbound ports.
- Monitor Windows audit policy changes, particularly Event ID `4719`.
- Audit scheduled task creation and the scripts they execute.
- Investigate SharpHound detections and correlate them with process and network telemetry.
- Monitor PowerShell script access and execution.
- Alert when Windows Firewall logs are cleared, including Event ID `104`.
- Preserve original EVTX files and maintain hashes to support evidence integrity.

## 📚 Key Takeaways

- Splunk enables efficient searching and correlation of Windows event logs.
- Firewall Event ID `2004` can reveal newly created firewall rules.
- Security Event ID `4719` identifies audit policy modifications.
- Scheduled task XML can expose execution paths, arguments, and scheduling details.
- Windows Defender events provide evidence of threat detection and remediation.
- Event ID `104` can indicate that a specific Windows event log was cleared.
- Correlating these artifacts provides a stronger incident narrative than analyzing any single event in isolation.

## 🧠 Skills Demonstrated

- Windows Event Log forensics
- Splunk SPL queries
- EVTX-to-JSON conversion
- Firewall event analysis
- Audit policy investigation
- Scheduled task analysis
- Windows Defender event analysis
- PowerShell logging
- Event log clearing detection
- Incident timeline reconstruction

---

<div align="center">

**HTB Sherlock: Logjammer**

*Windows DFIR • Splunk • Event Log Analysis*

</div>
