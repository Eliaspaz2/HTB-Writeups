
<div align="center">

# HTB Sherlock: Jingle Bell

![Category](https://img.shields.io/badge/Category-Windows%20Forensics-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Focus](https://img.shields.io/badge/Focus-Incident%20Response-red)

**Windows Notifications | SQLite Forensics | Insider Threat Investigation**

</div>

## Overview

Jingle Bell investigates a suspected insider threat at Forela. An employee, Torrin, was suspected of leaking confidential company information to a competitor and removing applications from their workstation.

The investigation focuses on Windows notification artifacts to recover Slack-related activity, identify the parties involved, and reconstruct the details of the suspected data leak.

## Tools & Artifacts

| Item | Details |
|---|---|
| Evidence archive | `jinglebell.zip` |
| SHA-256 | `4d857dd670e8f91479e201cd564dacc6480384c323735309f4ad8c0ab2a97ad9` |
| Primary artifact | Windows Notifications SQLite database |
| Database components | `.db`, `.db-shm`, `.db-wal` |
| Analysis tools | DB Browser for SQLite, SQL, `sed` |
| Investigation focus | Notification records and Slack messages |

## Investigation

### 1. Identify and Examine the Database

The provided evidence contained three components of a SQLite database:

- `.db` — the main database file.
- `.db-shm` — the shared-memory file used for database coordination.
- `.db-wal` — the write-ahead log containing changes that may not yet have been checkpointed into the main database.

All three files were kept together during analysis to ensure that SQLite could access the associated database state.

The database was opened using **DB Browser for SQLite**. Initial examination identified the `Notification` table as the primary source of relevant evidence.

<img width="956" height="69" alt="image" src="https://github.com/user-attachments/assets/78b88aaa-ce76-4cd7-98ba-b9253fbd5462" />


### 2. Extract Notification Records

The raw notification entries were difficult to interpret directly. SQL queries were used to filter relevant records, which were then exported to CSV for further examination.

The exported data was reformatted by inserting line breaks between entries, making individual notifications easier to read and correlate.

This process exposed Slack-related notification content that provided evidence of communication between Torrin and an individual associated with a competing organization.

<img width="954" height="844" alt="image" src="https://github.com/user-attachments/assets/1b8d4e5d-0d84-4943-9141-1ec4bcbb1ff2" />


<img width="955" height="617" alt="image" src="https://github.com/user-attachments/assets/a2ae0652-91b9-426c-89fd-9ace6d0a136a" />


### 3. Identify the Application Used

The recovered notifications contained references to Slack-specific activity, including channel launches and invitations to join Slack.

**Finding:** Slack was the application used to communicate about the suspected leak.

### 4. Identify the Recipient Organization

The notification content identified the competitor as **PrimeTech Innovations**.

The same notification evidence linked Torrin's communications to the username `Cyberjunkie-PrimeTechDev`, identifying the person with whom the information was shared.

### 5. Recover the Communication Channel

Further examination of the notification records identified the channel used for the conversation:

    forela-secrets-leak

The channel name strongly suggests that the communications concerned the disclosure of Forela's confidential information.

### 6. Recover the Archive Server Password

A subsequent notification disclosed the password associated with the archive server:

    Tobdaf8Qip$re@1

This credential was part of the recovered conversation and represents a sensitive indicator relevant to the investigation.

### 7. Identify the Data Upload Destination

The conversation included a Google Drive folder URL provided to Torrin for uploading the data:

    https://drive.google.com/drive/folders/1vW97VBmxDZUIEuEUG64g5DLZvFP-Pdll?usp=sharing

The recovered URL established the intended destination for the suspected data transfer. The notification evidence supports identifying this link as the destination shared with Torrin; it does not independently establish the complete contents of any uploaded files.

### 8. Establish the Time of the Shared Link

The timestamp associated with the notification was converted to UTC to establish when the Google Drive link was shared.

**Finding:** `2023-04-20 10:34:49 UTC`

This timestamp provides a reference point for correlating the recovered communication with other potential forensic evidence.

### 9. Determine the Reported Payment

The recovered messages indicated that Torrin was offered **£10,000** in exchange for leaking Forela's secrets.

This financial detail supports the suspected insider-threat motive described in the scenario.

## Reconstructed Incident

The notification evidence supports the following sequence:

1. Torrin communicated through Slack.
2. The communications involved `Cyberjunkie-PrimeTechDev`, associated with PrimeTech Innovations.
3. The parties used the `forela-secrets-leak` channel.
4. An archive server password was shared.
5. A Google Drive folder was provided as the intended upload destination on April 20, 2023, at 10:34:49 UTC.
6. The reported payment for the information was £10,000.

The recovered notifications provide evidence of the suspected data-leak arrangement. They should be correlated with additional endpoint, file-system, and network artifacts to independently establish whether the transfer was completed and precisely which files were disclosed.

## Key Findings

| Finding | Result |
|---|---|
| Application used | Slack |
| Competitor organization | PrimeTech Innovations |
| Recipient username | `Cyberjunkie-PrimeTechDev` |
| Communication channel | `forela-secrets-leak` |
| Archive server password | `Tobdaf8Qip$re@1` |
| Intended upload destination | Google Drive folder |
| Link shared (UTC) | `2023-04-20 10:34:49` |
| Reported payment | £10,000 |

## Defensive Recommendations

- **Monitor data exfiltration:** Detect unusual uploads to external file-sharing services and investigate transfers involving sensitive corporate data.
- **Audit collaboration platforms:** Review Slack messages, invitations, channel membership, and suspicious external communications in accordance with organizational policy.
- **Protect sensitive archives:** Enforce strong authentication, rotate exposed passwords, and restrict access to archive servers.
- **Preserve endpoint artifacts:** Collect Windows notification databases and their associated WAL/SHM files during incident response.
- **Correlate forensic evidence:** Compare notification timestamps with file access records, browser history, network logs, and cloud audit trails.
- **Strengthen insider-threat controls:** Apply least privilege, monitor unusual data access, and establish clear controls for sharing confidential information externally.

## Key Takeaways

- Windows notification databases can preserve valuable evidence of application activity and communications.
- SQLite WAL and SHM files may be necessary to interpret a database accurately.
- Notification content can help reconstruct suspected insider-threat activity even when direct evidence of file exfiltration is not immediately available.
- Recovered messages can establish relationships, intended transfer destinations, timestamps, and possible financial motives.
- Findings from notification artifacts should be correlated with independent evidence before concluding that data exfiltration was successfully completed.

## Skills Demonstrated

`Windows Forensics` · `SQLite Analysis` · `Notification Forensics` · `Incident Response` · `Insider Threat Investigation` · `Evidence Correlation` · `Data Exfiltration Analysis`
