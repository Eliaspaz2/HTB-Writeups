
<div align="center">

# 🕵️ Hack The Box Sherlock: RogueOne

**Memory Forensics · Incident Response · Threat Hunting**

![Platform](https://img.shields.io/badge/Platform-Windows-blue?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-success?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-Memory%20Forensics-purple?style=for-the-badge)
![Tool](https://img.shields.io/badge/Tool-Volatility%203-orange?style=for-the-badge)

</div>

## Overview

In this Sherlock, the SOC team investigates suspicious command-and-control (C2) alerts originating from Simon Stark's workstation. Although the Task Manager screenshots did not reveal any suspicious processes, the alerts continued, prompting the team to acquire a memory dump for forensic analysis.

Using **Volatility 3 on Linux**, the investigation identifies a malicious process masquerading as `svchost.exe`, traces its parent and child processes, extracts the suspicious executable, and correlates its network activity with the incident timeline.

## Objectives

- Identify the malicious process and its PID.
- Determine which child process enabled command execution.
- Extract the malicious executable and calculate its MD5 hash.
- Identify the C2 IP address and port.
- Establish the execution and connection timestamp.
- Retrieve the process memory offset.
- Determine when the sample was first submitted to VirusTotal.

## Tools and Artifacts

| Item | Details |
|---|---|
| Memory image | `20230810.mem` |
| Forensics framework | Volatility 3 |
| Shell | Bash |
| Analysis environment | Linux |
| Threat intelligence | VirusTotal |
| Initial archive | `MemDump.zip` |
| ZIP password | `hacktheblue` |
| Archive hash provided by the guide | `5810AA928E827D5AB8373AA4E1107572` |

## Investigation

### 1. Extract and inspect the memory dump

The provided ZIP archive is password-protected. After extracting it with the supplied password, the memory image `20230810.mem` is available for analysis.

Verify the archive's MD5 hash against the value provided by the Sherlock, then inspect the system information using Volatility 3.

```bash
md5sum MemDump.zip
unzip MemDump.zip
```

Start by identifying the operating system and other available system information:

```bash
python3 vol.py -f 20230810.mem windows.info
```

The following sections use this command as the base for all memory analysis.

### 2. Identify the malicious process

List the processes present in the memory image:

```bash
python3 vol.py -f 20230810.mem windows.pslist
```

At first glance, none of the processes immediately appears malicious. However, several `svchost.exe` instances share the same parent PID: **788**.

Investigate that parent process:

```bash
python3 vol.py -f 20230810.mem windows.pslist | grep -w 788
```

The expected parent is `services.exe`. A closer review reveals an anomalous `svchost.exe` instance with PID **6812**, whose parent PID is **7436** rather than 788.

Inspect the process tree:

```bash
python3 vol.py -f 20230810.mem windows.pstree | grep -E '6812|7436'
```

The parent process is identified as `explorer.exe`. This relationship is suspicious because the process does not follow the expected service-hosting pattern.

Inspect the command line and executable path:

```bash
python3 vol.py -f 20230810.mem windows.cmdline --pid 6812
```

The suspicious executable is located in the Downloads directory and masquerades as `svchost.exe`. The unusual parent-child relationship and execution path provide evidence that this is the malicious process.

**Finding:** PID `6812`.

### 3. Identify the child process

The next step is to determine whether the malicious process spawned another process that could have allowed the attacker to execute commands.

Filter the process listing for children whose parent PID is 6812:

```bash
python3 vol.py -f 20230810.mem windows.pslist | grep -w 6812
```

Review the process tree and identify the child process associated with the malicious `svchost.exe`.

The investigation identifies a `cmd.exe` process spawned by the malicious process.

**Finding:** Child process PID `4364`.

### 4. Extract the malicious executable and calculate its MD5

Use Volatility's `dumpfiles` plugin to extract files associated with the suspicious process:

```bash
python3 vol.py -f 20230810.mem windows.dumpfiles --pid 6812
```

Review the generated output and locate the dumped executable whose original filename is `svchost.exe`. Rename the extracted file for easier identification.

Calculate its MD5 hash:

```bash
md5sum <extracted_svchost_file>
```

The resulting hash can be provided to the reverse engineering team for further analysis and threat-intelligence correlation.

**Finding:** `5bd547c6f5bfc4858fe62c8867acfbb5`.

### 5. Identify the command-and-control endpoint

The `netscan` plugin identifies network connections represented in the captured memory:

```bash
python3 vol.py -f 20230810.mem windows.netscan
```

Filter the output for the malicious process PID:

```bash
python3 vol.py -f 20230810.mem windows.netscan | grep -w 6812
```

The results show a connection associated with the malicious `svchost.exe` process to a remote IP address on port `8888`.

**Finding:** `13.127.155.166:8888`.

### 6. Establish the execution and C2 timeline

The relevant timestamp is available in the `netscan` output identified in the previous step. Correlating the network connection with the malicious process helps establish when the activity occurred.

**Finding:** `10/08/2023 11:30:03`.

This timestamp is reported as the process execution and C2 establishment time in the supplied walkthrough.

### 7. Retrieve the malicious process memory offset

The `pslist` output contains the process offset alongside its process metadata.

Run the plugin again:

```bash
python3 vol.py -f 20230810.mem windows.pslist
```

Locate the row corresponding to PID `6812` and inspect its offset value.

**Finding:** `0x9e8b87762080`.

### 8. Determine the first VirusTotal submission time

The malicious executable's MD5 hash can be used to search for the sample in VirusTotal.

1. Open VirusTotal.
2. Search for the hash `5bd547c6f5bfc4858fe62c8867acfbb5`.
3. Open the sample's report.
4. Navigate to the **Details** tab.
5. Locate the first submission timestamp.

The walkthrough reports that the sample had already been uploaded, potentially by the reverse engineering team.

**Finding:** `10/08/2023 11:58:10`.

## Key Findings

| Question | Answer |
|---|---|
| Malicious process PID | `6812` |
| Child process PID | `4364` |
| Malicious executable MD5 | `5bd547c6f5bfc4858fe62c8867acfbb5` |
| C2 IP and port | `13.127.155.166:8888` |
| Execution/C2 timestamp | `10/08/2023 11:30:03` |
| Process memory offset | `0x9e8b87762080` |
| First VirusTotal submission | `10/08/2023 11:58:10` |

## Indicators of Compromise

| Indicator | Value |
|---|---|
| Suspicious process | `svchost.exe` |
| Malicious PID | `6812` |
| Parent PID | `7436` |
| Parent process | `explorer.exe` |
| Child process | `cmd.exe` |
| Child PID | `4364` |
| Executable MD5 | `5bd547c6f5bfc4858fe62c8867acfbb5` |
| C2 endpoint | `13.127.155.166:8888` |

## Incident Response Recommendations

1. Isolate the affected workstation and preserve the original memory image and extracted artifacts.
2. Investigate the suspicious executable and determine how it reached the Downloads directory.
3. Hunt across the environment for the executable's MD5 hash, suspicious process relationships, and the C2 endpoint.
4. Review endpoint and network logs around the identified timestamp.
5. Investigate the spawned `cmd.exe` process and determine which commands were executed.
6. Use the extracted sample for controlled malware analysis and enrich the investigation with threat intelligence.
7. Remove malicious artifacts and restore the endpoint only after the incident's scope has been established.

## Skills Demonstrated

- Windows memory forensics with Volatility 3
- Process and parent-child relationship analysis
- Suspicious executable identification
- File extraction and hash verification
- Network connection analysis
- C2 indicator identification
- Timeline reconstruction
- Threat intelligence and incident response
- Bash-based forensic workflows

---

<div align="center">

**HTB Sherlock — RogueOne**

*Memory forensics and threat hunting with Volatility 3 on Linux.*

</div>
