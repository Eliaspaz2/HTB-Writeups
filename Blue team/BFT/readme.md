# 🟢 Hack The Box — BFT

<p align="center">
  <img src="https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black">
  <img src="https://img.shields.io/badge/Difficulty-Very%20Easy-9FEF00?style=for-the-badge">
  <img src="https://img.shields.io/badge/Category-Windows%20Forensics-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/Focus-DFIR-red?style=for-the-badge">
</p>

---

## 📌 Machine Overview

**BFT** is a Very Easy Windows forensics challenge focused on **Master File Table (MFT) analysis**.

The investigation introduces a forensic methodology for analyzing NTFS MFT artifacts to identify malicious activity, reconstruct a timeline, identify suspicious files, recover file contents, and extract indicators of compromise.

The investigation is performed using:

- **MFTeCMD** to parse the MFT.
- **Timeline Explorer** to analyze and filter the parsed results.
- **HxD** to inspect the raw MFT data and recover the contents of an MFT-resident malicious file.

The investigation ultimately reveals a malicious batch file containing a PowerShell one-liner that connects to a remote Command and Control server.

---

# 🎯 Objectives

The main objectives of this investigation are:

1. Identify the ZIP file initially downloaded by the victim.
2. Identify the Host URL associated with the downloaded ZIP file.
3. Identify the full path of the malicious file responsible for executing malicious code and connecting to the C2 server.
4. Determine when the malicious file was created.
5. Calculate the hexadecimal MFT offset of the malicious file.
6. Recover the contents of the malicious file directly from the MFT.
7. Extract the C2 IP address and port used by the malicious PowerShell payload.

---

# 🗂️ Artifacts Provided

The investigation provides a single forensic artifact:

    2024-02-13T164623_MFT_TRIAGE.zip

### SHA1

    5A9BD3D02E0DC0732AAF882879FB6AF02795E1B9

The ZIP archive contains the MFT artifact required for the investigation.

---

# 🧠 1. Understanding the Master File Table

The **Master File Table (MFT)** is a critical component of the **NTFS (New Technology File System)** used by Windows.

The MFT can be considered a database containing metadata about files and directories stored on an NTFS volume.

Each file or directory is represented by an MFT record and assigned a unique **MFT record number**.

The MFT is extremely valuable during Windows forensic investigations because it can provide information about:

- File creation
- File modification
- File access
- File names
- File paths
- File metadata
- File sizes
- Deleted files
- File contents for certain resident files

This information can be used to reconstruct user activity, identify suspicious changes, recover deleted content, and establish forensic timelines.

---

# 🔬 2. Important MFT Attributes

Several MFT attributes are particularly relevant during forensic investigations.

## `$STANDARD_INFORMATION`

Contains timestamps associated with the file or directory:

- Creation Time
- Modification Time
- Access Time
- Entry Modified Time

These timestamps are useful for reconstructing the sequence of events.

---

## `$FILE_NAME`

Contains information such as:

- File name
- Parent directory
- Additional timestamps

This attribute can be used to validate file paths and identify suspicious changes.

---

## `$DATA`

Contains either the actual file data or a pointer to the data when the file is not resident.

This attribute is particularly important when attempting to recover file contents.

---

## `$LOGGED_UTILITY_STREAM`

Contains transactional NTFS information and can provide information about temporary file state changes.

---

## `$BITMAP`

Provides information about cluster allocation and can assist with recovering deleted files or reconstructing file data.

---

## `$SECURITY_DESCRIPTOR`

Contains information about:

- File owner
- Permissions
- Audit settings

This can be useful when investigating unauthorized permission changes.

---

## `$VOLUME_INFORMATION`

Contains information about the NTFS volume, including:

- Volume serial number
- Volume flags

---

## `$INDEX_ROOT` and `$INDEX_ALLOCATION`

These attributes contain directory indexing information and can assist in reconstructing directory structures.

---

# 🛠️ 3. Tools Used

The investigation uses the following tools:

### MFTeCMD

**MFTeCMD** is a forensic utility developed by Eric Zimmerman that parses MFT records and converts their contents into a more easily analyzed format.

### Timeline Explorer

**Timeline Explorer** is used to view, filter, and analyze timestamped forensic data generated from the MFT.

### HxD

**HxD** is a hex editor used to inspect the raw MFT and recover the contents of files stored directly within MFT records.

---

# ⚙️ 4. Parsing the MFT

The preferred methodology is to parse the MFT using **MFTeCMD** and export the results to CSV.

The command used is:

    MFTECMD.exe -f "C:\Users\burst\Desktop\BFT\$MFT" --csv "C:\Users\burst\Desktop\BFT_machine" --csvf mft.csv

This generates an `mft.csv` file containing the parsed MFT records.

The resulting CSV can then be imported into **Timeline Explorer** for filtering and analysis.

---

# 🔎 5. Identifying the Initial ZIP File

The scenario states that **Simon Stark** was targeted by attackers on **13 February 2024** and downloaded a ZIP file from a link received through email.

The first objective is therefore to identify which ZIP file was downloaded.

### Filtering by Date

In Timeline Explorer, the parsed MFT data can be filtered using the relevant timestamp column.

The investigation focuses on:

    13 February 2024

The date filter allows us to reduce the dataset to files created around the time of the suspected incident.

### Filtering by Extension

Since the scenario specifies that a ZIP file was downloaded, the `Extension` field can then be filtered for:

    zip

The resulting entries include:

    Stage-20240213T093324Z-001.zip
    invoices.zip
    KAPE.zip

`KAPE.zip` can be excluded because it is associated with artifact collection rather than the original download.

The remaining ZIP files are then compared using their paths and parent relationships.

The file identified as the initial download is:

    Stage-20240213T093324Z-001.zip

### Answer

    Stage-20240213T093324Z-001.zip

---

# 🌐 6. Identifying the Download URL

The next objective is to identify where the malicious ZIP file originated.

When a file is downloaded through a browser on Windows, an **Alternate Data Stream (ADS)** can be associated with the file.

One important ADS is the:

    Zone.Identifier

This can contain information about the source of the downloaded file, including the original Host URL.

To investigate this, the ZIP filter can be removed and the filename can be filtered using:

    stage

The investigation then focuses on entries associated with the downloaded file and specifically looks for the:

    Identifier

extension.

The Host URL associated with the initial ZIP file is:

    https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser

This indicates that **Google Drive** was used to host the downloaded ZIP file.

### Answer

    https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser

---

# 🦠 7. Identifying the Malicious File

The next step is to identify a suspicious file associated with the initially downloaded ZIP.

The investigation removes the previous filename filter and instead searches the **Parent Path** column for:

    stage

This reveals a suspicious batch file:

    invoice.bat

Batch files can execute commands directly on Windows and are therefore relevant during malware investigations.

The parent path allows the complete location of the suspicious file to be reconstructed.

### Full Path

    C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat

### Answer

    C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat

---

# 🕒 8. Determining the File Creation Time

The `Created0x30` timestamp can be used to determine when the malicious file was created on disk.

Analyzing this timestamp allows the event to be placed into the broader forensic timeline and correlated with the initial download and subsequent malicious activity.

The creation time of `invoice.bat` was:

    2024-02-13 16:38:39

### Answer

    2024-02-13 16:38:39

---

# 🔢 9. Calculating the MFT Offset

The next objective is to determine the raw MFT offset of the malicious `invoice.bat` file.

Each MFT record is:

    1024 bytes

The first step is to identify the **MFT Entry Number** associated with the file.

For `invoice.bat`, the MFT Entry Number is:

    23436

The offset can then be calculated using:

    MFT Entry Number × 1024

Therefore:

    23436 × 1024 = 23998464

The resulting decimal offset is:

    23998464

The decimal value must then be converted to hexadecimal.

The resulting hexadecimal offset is:

    16E3000

### Answer

    16E3000

---

# 💾 10. MFT Resident Files

An important forensic concept in this investigation is the concept of **MFT Resident Files**.

Each MFT record has a fixed size of:

    1024 bytes

When a file is sufficiently small, its contents may be stored directly inside its MFT record instead of being stored elsewhere on disk.

These are known as:

    MFT Resident Files

This is particularly useful during forensic investigations because the contents of a small malicious script may remain directly inside the MFT.

The `invoice.bat` file is sufficiently small to be stored as an MFT-resident file.

This means that its contents can be recovered directly from the raw MFT artifact.

---

# 🧪 11. Recovering the Malicious File with HxD

The next step is to inspect the raw MFT using a hex editor.

**HxD** can be used to open the MFT artifact and navigate directly to the calculated hexadecimal offset.

The relevant offset is:

    16E3000

In HxD, the **Go To** functionality can be used to jump directly to this location.

After navigating to the offset, the data can be validated by looking for recognizable information such as:

    invoice.bat

The raw data contains the contents of the malicious batch file.

Because the file is MFT-resident, its contents are stored directly inside the MFT record.

---

# ⚡ 12. Analyzing the Malicious Stager

Inspection of the recovered `invoice.bat` contents reveals a PowerShell one-liner.

The PowerShell code contains an IP address and port used by the malware to communicate with its Command and Control infrastructure.

This establishes that the batch file is responsible for executing malicious code and attempting to communicate with a remote C2 server.

### C2 Server

    43.204.110.203:6666

### Answer

    43.204.110.203:6666

---

# 🧭 13. Attack Timeline

The investigation allows the following sequence of events to be reconstructed:

| Time / Stage | Event |
|---|---|
| 2024-02-13 | Simon Stark is targeted by attackers |
| 2024-02-13 | A ZIP file is downloaded from a link received by email |
| 2024-02-13 | `Stage-20240213T093324Z-001.zip` is identified as the initial download |
| 2024-02-13 | The download is associated with a Google Drive Host URL |
| 2024-02-13 16:38:39 | `invoice.bat` is created on disk |
| Post-download | `invoice.bat` is identified within the extracted Stage directory |
| Forensic analysis | MFT Entry Number `23436` is identified |
| Forensic analysis | MFT offset is calculated as `16E3000` |
| Forensic analysis | `invoice.bat` is recovered from the MFT |
| Post-analysis | PowerShell one-liner reveals C2 `43.204.110.203:6666` |

---

# 🚨 14. Indicators of Compromise

## Malicious ZIP

    Stage-20240213T093324Z-001.zip

## Malicious Batch File

    invoice.bat

## Full File Path

    C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat

## Download Host URL

    https://storage.googleapis.com/drive-bulk-export-anonymous/20240213T093324.039Z/4133399871716478688/a40aecd0-1cf3-4f88-b55a-e188d5c1c04f/1/c277a8b4-afa9-4d34-b8ca-e1eb5e5f983c?authuser

## C2 Server

    43.204.110.203:6666

## MFT Entry Number

    23436

## MFT Offset

    16E3000

---

# 🧩 15. Forensic Methodology

The investigation follows a structured Windows forensic workflow:

    1. Obtain the MFT artifact
           │
           ▼
    2. Parse the MFT with MFTeCMD
           │
           ▼
    3. Export the results to CSV
           │
           ▼
    4. Import the CSV into Timeline Explorer
           │
           ▼
    5. Filter by relevant date
           │
           ▼
    6. Filter by ZIP extension
           │
           ▼
    7. Identify the initial downloaded archive
           │
           ▼
    8. Analyze Zone.Identifier / ADS data
           │
           ▼
    9. Identify the download Host URL
           │
           ▼
    10. Search related Stage paths
           │
           ▼
    11. Identify invoice.bat
           │
           ▼
    12. Analyze Created0x30 timestamp
           │
           ▼
    13. Identify MFT Entry Number
           │
           ▼
    14. Calculate the raw MFT offset
           │
           ▼
    15. Open the MFT in HxD
           │
           ▼
    16. Recover the MFT-resident file
           │
           ▼
    17. Analyze the PowerShell payload
           │
           ▼
    18. Extract the C2 IP and port

---

# 🧠 16. Key Findings

The investigation identified several important artifacts that allowed the malicious activity to be reconstructed.

### Initial Delivery

The victim downloaded:

    Stage-20240213T093324Z-001.zip

The file was hosted using a Google Drive-related storage URL.

### Malicious Payload

Inside the extracted Stage directory, the investigation identified:

    invoice.bat

located at:

    C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat

### Timestamp

The malicious batch file was created at:

    2024-02-13 16:38:39

### MFT Record

The file was associated with MFT Entry:

    23436

### Raw Offset

The calculated hexadecimal MFT offset was:

    16E3000

### Payload

The file was small enough to be stored directly within the MFT as a resident file.

### C2 Infrastructure

The embedded PowerShell payload revealed:

    43.204.110.203:6666

---

# 📝 17. Investigation Summary

The investigation began with a single NTFS Master File Table artifact.

The MFT was parsed using **MFTeCMD** and the resulting CSV was imported into **Timeline Explorer**, allowing the forensic data to be filtered according to the date and file characteristics described in the scenario.

Filtering for ZIP files created on **13 February 2024** revealed several candidates. After excluding the KAPE artifact and correlating the remaining entries using their parent paths, the initial malicious download was identified as:

    Stage-20240213T093324Z-001.zip

The associated Zone Identifier information was then analyzed to identify the Host URL responsible for the download. The URL pointed to Google Cloud Storage infrastructure associated with Google Drive.

Further investigation of the Stage-related parent paths revealed a suspicious batch file:

    invoice.bat

The complete path was:

    C:\Users\simon.stark\Downloads\Stage-20240213T093324Z-001\Stage\invoice\invoices\invoice.bat

Its `Created0x30` timestamp established that the file was created at:

    2024-02-13 16:38:39

The MFT Entry Number for the file was `23436`. Since each MFT record is 1024 bytes, the raw offset was calculated as:

    23436 × 1024 = 23998464

The decimal value was converted to hexadecimal:

    16E3000

Using HxD, the investigation navigated directly to this offset within the raw MFT. The contents of `invoice.bat` were recovered because the file was small enough to be stored as an MFT-resident file.

The recovered contents contained a PowerShell one-liner that revealed the Command and Control server:

    43.204.110.203:6666

This demonstrates the value of MFT analysis in Windows forensics. Even when a suspicious file is no longer available through a conventional filesystem view, its metadata and, in some cases, its complete contents may remain recoverable from the MFT.

---

# 🎯 18. Key Takeaways

- The MFT is one of the most valuable artifacts for Windows forensic investigations.
- MFT records contain metadata that can be used to reconstruct file activity.
- MFT timestamps can help establish an incident timeline.
- Timeline Explorer makes it easier to filter and correlate large amounts of parsed MFT data.
- Zone Identifier information can reveal the source URL of downloaded files.
- Alternate Data Streams can provide valuable forensic evidence about file provenance.
- Small files may be stored directly inside their MFT records as MFT-resident files.
- MFT-resident files can sometimes be recovered directly from the raw MFT.
- Calculating the MFT offset allows forensic analysts to navigate directly to the relevant record in a hex editor.
- Recovering the contents of a malicious script can reveal critical indicators such as C2 infrastructure.
- Combining metadata analysis with raw artifact analysis provides a stronger forensic investigation methodology.

---

# 🛠️ Skills Demonstrated

- NTFS Forensics
- Windows Forensics
- MFT Analysis
- Timeline Creation
- File Recovery
- Disk Forensics
- Alternate Data Stream Analysis
- Contextual Analysis
- Malware Artifact Analysis
- Hexadecimal Analysis
- MFT Resident File Recovery
- IOC Identification
- C2 Infrastructure Identification
- DFIR Methodology

---

# 🏁 Final Attack Chain

    Malicious Email / Link
             │
             ▼
    ZIP File Downloaded
             │
             ▼
    Stage-20240213T093324Z-001.zip
             │
             ▼
    Stage Directory
             │
             ▼
    invoice.bat
             │
             ▼
    PowerShell One-Liner
             │
             ▼
    C2 Connection
             │
             ▼
    43.204.110.203:6666

---

# 📚 Tools

- **MFTeCMD** — MFT parsing
- **Timeline Explorer** — Timeline analysis and filtering
- **HxD** — Raw hexadecimal analysis and file recovery

---

**Machine:** BFT  
**Platform:** Hack The Box  
**Difficulty:** Very Easy  
**Category:** Windows Forensics / DFIR  
**Primary Artifact:** NTFS Master File Table (MFT)
