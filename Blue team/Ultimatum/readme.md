<div align="center">

# 🛡️ Hack The Box Sherlock: Ultimatum

**Linux Forensics · Web Forensics · Incident Response**

![Platform](https://img.shields.io/badge/Platform-Linux-informational?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-success?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-DFIR-blue?style=for-the-badge)

</div>

## Overview

A Forela WordPress server hosting an employee social-club blog was targeted through a vulnerable plugin. The investigation uses a CatScale incident-response collection, Apache access logs, a file timeline, and process/network artifacts to reconstruct the attack.

The evidence indicates that the attacker first fingerprinted the WordPress site, exploited **CVE-2023-3460** in Ultimate Member, created the unauthorized account `secragon`, then accessed the site from a second IP and modified a PHP theme file to establish a reverse web shell.

> **Evidence note:** The screenshots in `screenshots/` are reference screenshots extracted from the supplied Ultimatum walkthrough PDF. They illustrate the documented investigation steps; they are not screenshots captured from a separate live analysis session.

## Investigation Environment and Artifacts

- **Initial archive:** `Ultimatum.zip`
- **SHA-256/MD5 as stated by the source:** `Ultimatum.zip` — `3FD4E122BBD899F43FA7101918FE335D` (the source lists this value without explicitly identifying the algorithm)
- **Extracted archive:** `Ultimatum.tar.gz`
- **MD5:** `3B297966923AD265CDAEE943B406569F`
- **Collection:** CatScale output
- **Primary evidence:** Apache access logs, `-full-timeline.csv`, `-ss-anepo.txt`, `-processes-axwwSo.txt`, `-dev-dir-files-hashes.txt`, and a captured webshell file

### Relevant locations

```text
catscale_out/
├── Logs/
│   └── .../var/log/apache2/
│       ├── access.log
│       └── access.log.1
├── Misc/
│   ├── ...-pot-webshell-first-1000.txt
│   ├── ...-full-timeline.csv
│   └── ...-dev-dir-files-hashes.txt
└── Process_and_Network/
    ├── ...-ss-anepo.txt
    └── ...-processes-axwwSo.txt
```

## Investigation Walkthrough

### 1. Identify reconnaissance in Apache logs

Review the Apache access logs for automated WordPress enumeration. Requests from `23.106.60.163` identify **WPScan 3.8.24**, with numerous requests to target paths and many `404 Not Found` responses.

PowerShell search used in the walkthrough:

```powershell
Select-String 23.106.60.163 .\access.log.1 | Sort-Object
```

<img width="1456" height="501" alt="01-wpscan-reconnaissance" src="https://github.com/user-attachments/assets/0b63ab5c-b726-497d-9dcc-6854e1658bcc" />


The volume and pattern of requests are consistent with automated reconnaissance.

### 2. Fingerprint WordPress and its plugin

Search the logs for WordPress administration and plugin paths to identify version information:

```powershell
Select-String /wp-admin .\access.log | Sort-Object
Select-String /wp-content/plugins/ .\access.log | Sort-Object
```

The walkthrough identifies **WordPress 6.2.2** and **Ultimate Member 2.6.4**.

WordPress version evidence <img width="1828" height="443" alt="02-wordpress-version" src="https://github.com/user-attachments/assets/efaef151-f695-4296-a3c4-2331cf26fe7c" />


Plugin version evidence <img width="1896" height="318" alt="03-plugin-version" src="https://github.com/user-attachments/assets/de44773c-ff8b-4a34-9fb1-ed93c63b70a2" />


### 3. Correlate the registration request with CVE-2023-3460

Ultimate Member 2.6.4 is associated in the walkthrough with **CVE-2023-3460**, a vulnerability that can allow privilege escalation. Search for registration activity:

```powershell
Select-String /register .\access.log | Sort-Object
```

Then inspect the context surrounding the suspicious registration request:

```powershell
Select-String -Path .\access.log `
  -Pattern "POST //index.php/register/" `
  -Context 5,5
```

The logs show a registration POST from `23.106.60.163`, followed by creation of the unauthorized account `secragon` and a subsequent login. This supports the conclusion that the vulnerable registration flow was abused.

Registration request and surrounding log context 

<img width="1969" height="365" alt="04-registration-exploit-context" src="https://github.com/user-attachments/assets/7ebafc0e-b048-4d15-9278-da8ce447a7ca" />


### 4. Trace post-compromise activity from a second IP

After the account was created and used, the activity shifted to **`198.16.74.45`**. The walkthrough describes this source as manually exploring the WordPress site and interacting with PHP theme files, including `hidden-404.php` and `hidden-comments.php`.

The captured webshell artifact shows that `hidden-comments.php` was modified to contain a PHP reverse shell.

Post-exploitation activity in the logs

<img width="1981" height="597" alt="05-attacker-followup-activity" src="https://github.com/user-attachments/assets/a6c47198-8263-4694-8162-b15c06c34d9d" />


Evidence of the PHP reverse shell

<img width="1967" height="364" alt="06-reverse-shell-webshell-evidence" src="https://github.com/user-attachments/assets/c412361a-6d6d-45ed-85c5-794082f35e7a" />


### 5. Confirm the webshell path and file size

The file timeline can be filtered for the full path of the modified PHP file:

```powershell
Select-String `
  /var/www/html/wp-content/themes/twentytwentythree/patterns/hidden-comments.php `
  .\ip-172-31-11-131-20230808-full-timeline.csv
```

The timeline records the webshell's file metadata. The size reported by the walkthrough is **2,592 bytes**.

Webshell file metadata in the timeline

<img width="1326" height="679" alt="07-webshell-timeline-file-size" src="https://github.com/user-attachments/assets/2e2588c2-e888-4f8f-bdea-a836df142d35" />


### 6. Correlate the reverse-shell connection

Inspect `Process_and_Network/-ss-anepo.txt`, which contains output from Linux `ss -anepo`. The walkthrough correlates the reverse-shell activity with the command-and-control endpoint:

- **C2 IP:** `43.204.24.76`
- **C2 port:** `6969`

C2 endpoint evidence

<img width="1892" height="108" alt="08-c2-socket-evidence" src="https://github.com/user-attachments/assets/73b5d331-c416-43f4-bdc1-ae63cc7a32ca" />


Network socket output

<img width="916" height="167" alt="09-network-socket-output" src="https://github.com/user-attachments/assets/04e3ec85-b813-437b-bcd6-aa679706804c" />


### 7. Identify the interactive shell process

Review `-processes-axwwSo.txt`, generated from `ps -axwwSo`. The output identifies `/bin/bash -i` with PID **234521**, as a child of `sh -c` with PID **234517**. The interactive Bash process is evidence of hands-on shell access.

Interactive Bash process and PID

<img width="834" height="563" alt="10-interactive-bash-process" src="https://github.com/user-attachments/assets/7307b4fd-c43c-4b97-a049-fc8693254440" />


### 8. Identify the internal enumeration script

Inspect `Misc/-dev-dir-files-hashes.txt`. The walkthrough identifies **`LinEnum.sh`**, an open-source Linux enumeration script used to collect host information and investigate possible privilege-escalation paths. The source then checks its hash with VirusTotal.

LinEnum.sh entry in the hash listing

<img width="495" height="76" alt="11-linenum-hash-notepad" src="https://github.com/user-attachments/assets/9c87dafb-aadc-4a48-900a-1c4c2dbbaf95" />



## Attack Timeline

| Stage | Evidence | Finding |
|---|---|---|
| Reconnaissance | Apache access logs | WPScan 3.8.24 from `23.106.60.163` |
| Target identification | Requests to WordPress and plugin paths | WordPress 6.2.2; Ultimate Member 2.6.4 |
| Exploitation | Registration POST and surrounding log entries | CVE-2023-3460; account `secragon` created |
| Follow-on access | Apache access logs | Second IP `198.16.74.45` explored the site |
| Webshell installation | Captured file and timeline | `hidden-comments.php`, 2,592 bytes |
| Remote shell | Socket and process artifacts | C2 `43.204.24.76:6969`; interactive `/bin/bash -i`, PID 234521 |
| Internal enumeration | File hash listing | `LinEnum.sh` identified |

## Key Findings

| Question | Answer |
|---|---|
| Scanning tool and version | `wpscan/3.8.24` |
| Exploited CVE | `CVE-2023-3460` |
| IP used during exploitation | `23.106.60.163` |
| Unauthorized WordPress user | `secragon` |
| IP used for follow-on access | `198.16.74.45` |
| PHP webshell file | `hidden-comments.php` |
| Webshell size | `2592` bytes |
| C2 IP and port | `43.204.24.76:6969` |
| Interactive shell PID | `234521` |
| Enumeration script | `LinEnum.sh` |

## Recommended Response Actions

1. Isolate the affected WordPress server and preserve forensic evidence.
2. Remove unauthorized WordPress accounts and review privilege assignments and authentication logs.
3. Update WordPress and affected plugins to verified, secure versions; disable or remove vulnerable/unneeded plugins.
4. Restore the altered theme file from a trusted source and scan the full web root for additional implants.
5. Investigate outbound connections and processes associated with `43.204.24.76:6969`.
6. Review the host for artifacts related to `LinEnum.sh` and any subsequent privilege-escalation activity.
7. Rotate credentials and secrets that may have been exposed, and monitor for repeat access from the identified IPs.

## Skills Demonstrated

- Linux host forensics
- Apache access-log analysis
- WordPress and plugin version fingerprinting
- CVE research and exploit-log correlation
- Webshell identification
- File timeline analysis
- Process and network socket correlation
- Incident response and evidence-driven reporting

---

<div align="center">

**HTB Sherlock — Ultimatum**  
*Writeup based on the supplied investigation PDF.*

</div>
