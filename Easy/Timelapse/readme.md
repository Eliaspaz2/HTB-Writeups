# Hack The Box - Timelapse

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Timelapse-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Active Directory](https://img.shields.io/badge/Focus-Active%20Directory-red?style=for-the-badge)

> **Timelapse** is an Easy-difficulty Windows machine that involves enumerating a publicly accessible SMB share containing a password-protected ZIP archive. The archive contains a password-protected PFX certificate, which can be cracked and used to authenticate through WinRM. Further enumeration reveals credentials for the `svc_deploy` account in PowerShell command history. This account belongs to the `LAPS_Readers` group, allowing access to the LAPS password stored in the `ms-Mcs-AdmPwd` attribute. By directly enumerating computer objects in Active Directory, the local Administrator password can be recovered and used to obtain a privileged WinRM session.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | **Timelapse** |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Easy |
| **Target IP** | `10.129.227.113` |
| **Hostname** | `dc01` |
| **Domain** | `timelapse.htb` |
| **Primary Focus** | Active Directory |
| **Initial Access** | Anonymous SMB → ZIP → PFX → WinRM |
| **Privilege Escalation** | PowerShell History → `svc_deploy` → LAPS → Administrator |

---

# 1. Enumeration

## 1.1 Nmap

I started with a full TCP port scan to identify the services exposed by the target.

    nmap -sS -sV -sC -p- -Pn --min-rate 5000 10.129.227.113 -oN nmap

The scan revealed multiple Windows services and identified the target as:

    Hostname: dc01
    Domain: timelapse.htb

<img width="958" height="767" alt="image" src="https://github.com/user-attachments/assets/0fe5e37d-6392-42d7-957a-f903b81cd9bc" />


SMB was exposed, making anonymous share enumeration an important next step.

---

# 2. SMB Enumeration

## 2.1 Enumerating SMB Shares

I enumerated the available SMB shares:

    smbclient -L //10.129.227.113/

A share named:

    Shares

was accessible without authentication.

I connected to it using:

    smbclient //10.129.227.113/Shares

The share contained two interesting directories:

    Dev
    HelpDesk

Inside the `Dev` directory, I discovered:

    winrm_backup.zip

The ZIP archive was password protected.

<img width="956" height="196" alt="image" src="https://github.com/user-attachments/assets/0d162f18-765f-462a-b760-57ccf64d5b27" />


---

# 3. Cracking the ZIP Password

Since the password for `winrm_backup.zip` was unknown, I used `zip2john` to convert the ZIP password into a format that John the Ripper could crack.

    zip2john winrm_backup.zip > zip.john

<img width="957" height="81" alt="image" src="https://github.com/user-attachments/assets/52725e0a-c343-467f-a943-ff9277f163ab" />


Then I used the `rockyou.txt` wordlist:

    john zip.john --wordlist:/usr/share/wordlists/rockyou.txt

The ZIP password was successfully recovered:

    supremelegacy

<img width="954" height="223" alt="image" src="https://github.com/user-attachments/assets/74de1de7-077e-4610-9694-350b7a299c47" />


The archive could then be extracted.

---

# 4. PFX Certificate

Inside the extracted files was a PFX certificate:

    legacyy_dev_auth.pfx

The PFX file contained an SSL certificate and a private key.

However, the PFX itself was also password protected.

The password was different from the ZIP password, so I needed to crack it separately.

---

# 5. Cracking the PFX Password

I converted the PFX file into a hash format supported by John the Ripper using `pfx2john`.

   python3 pfx2john.py legacyy_dev_auth.pfx > pff.john

<img width="959" height="153" alt="image" src="https://github.com/user-attachments/assets/902e6c39-3145-4f45-9bcc-d5b555d28e47" />


Then I cracked the resulting hash:

    john pff.john --wordlist:/usr/share/wordlists/rockyou.txt

<img width="956" height="267" alt="image" src="https://github.com/user-attachments/assets/88bfbd89-3401-46f1-ba36-ed6453a887fc" />


The PFX password was successfully recovered.

After obtaining the password, I extracted the private key:

    openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes

And extracted the certificate:

    openssl pkcs12 -in legacyy_dev_auth.pfx -nokeys -out cert.pem

This resulted in:

    key.pem
    cert.pem

<img width="957" height="325" alt="image" src="https://github.com/user-attachments/assets/a9ecaaab-ced9-46a0-9fa6-c651e2bb72b1" />


These files could be used to authenticate to the target through WinRM.

---

# 6. Initial Access - WinRM

Nmap revealed that port `5986` was open.

Port `5986` is commonly used by WinRM over HTTPS.

Evil-WinRM supports certificate-based authentication using the `-c` and `-k` options.

I used:

    evil-winrm -i 10.129.227.113 -c cert.pem -k key.pem -S

<img width="961" height="293" alt="image" src="https://github.com/user-attachments/assets/fb22e04e-89fa-402e-9597-c117db9033f6" />


This successfully authenticated to the target.

The initial foothold was obtained through WinRM using the extracted certificate and private key.

---

# 7. PowerShell History

After obtaining the WinRM session, I performed manual enumeration.

One of the first things to check was the PowerShell command history.

The relevant file is:

    $env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

Reading the history revealed credentials for another user:

    svc_deploy

The recovered credentials were:

    Username: svc_deploy
    Password: E3R$Q62^12p7PLlC%KWaxuaV

These credentials could be used to authenticate through WinRM.

<img width="954" height="228" alt="image" src="https://github.com/user-attachments/assets/17addd11-8908-4afb-b513-6a4a9fef1376" />


---

# 8. svc_deploy Access

I authenticated using the recovered credentials:

    evil-winrm -i 10.129.227.113 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S

This provided a new WinRM session as:

    svc_deploy

I then enumerated the user's group membership:

    whoami /all

The output showed that `svc_deploy` was a member of:

    LAPS_Readers

<img width="958" height="742" alt="image" src="https://github.com/user-attachments/assets/67c80d89-27d4-4613-bce0-28295cdac7f0" />


---

# 9. LAPS Enumeration

LAPS stands for **Local Administrator Password Solution**.

It is used in Active Directory environments to manage unique local Administrator passwords on domain-joined computers.

The important point in this machine is that the `LAPS_Readers` group has permissions related to reading LAPS-managed passwords.

I used direct Active Directory enumeration to retrieve the Administrator password.

---

# 10. Administrator Password Discovery

I enumerated all computer objects in Active Directory and requested all available properties:

    Get-ADComputer -Filter 'ObjectClass -eq "computer"' -Property *

The output contained the LAPS-related attribute:

    ms-Mcs-AdmPwd

The value of this attribute contained the local Administrator password:

    7y!&Z}83&vAdo00t+3M02)J&

This provided the credentials required to authenticate as the local Administrator.

<img width="957" height="674" alt="image" src="https://github.com/user-attachments/assets/fa62a2dd-e660-4c3e-9f4d-e37a4d7e4401" />

<img width="957" height="37" alt="image" src="https://github.com/user-attachments/assets/f8ca8845-9dbf-444a-ac56-112d02285c93" />


> **Note:** The `ms-Mcs-AdmPwd` attribute contains the plaintext LAPS password on systems using the legacy LAPS implementation. Access to this attribute is controlled by Active Directory permissions.

---

# 11. Administrator Access

With the recovered Administrator password, I authenticated through WinRM:

    evil-winrm -i 10.129.227.113 -u administrator -p '7y!&Z}83&vAdo00t+3M02)J&' -S

The authentication was successful.

I obtained a privileged shell as:

    administrator

At this point, the machine was fully compromised.

<img width="957" height="292" alt="image" src="https://github.com/user-attachments/assets/9652c29a-2b09-48bc-ad80-87dd517378db" />


---

# 12. Root Flag

With Administrator access, the root flag could be retrieved from the Administrator's Desktop.

    C:\Users\TRX\Desktop\root.txt

The machine was successfully compromised with administrative privileges.

---

# 13. Attack Chain

    Nmap
      |
      v
    SMB Enumeration
      |
      v
    Anonymous "Shares" Share
      |
      v
    /Dev/winrm_backup.zip
      |
      v
    zip2john
      |
      v
    ZIP Password Cracked
      |
      v
    legacyy_dev_auth.pfx
      |
      v
    pfx2john
      |
      v
    PFX Password Cracked
      |
      v
    SSL Certificate + Private Key
      |
      v
    Evil-WinRM
      |
      v
    Initial WinRM Access
      |
      v
    PowerShell History
      |
      v
    svc_deploy Credentials
      |
      v
    svc_deploy
      |
      v
    LAPS_Readers Group
      |
      v
    Get-ADComputer -Property *
      |
      v
    ms-Mcs-AdmPwd
      |
      v
    Administrator Password
      |
      v
    Evil-WinRM
      |
      v
    Administrator

---

# 14. Credentials

| Username | Password | Source |
|---|---|---|
| `svc_deploy` | `E3R$Q62^12p7PLlC%KWaxuaV` | PowerShell command history |
| `administrator` | `7y!&Z}83&vAdo00t+3M02)J&` | `ms-Mcs-AdmPwd` Active Directory attribute |

Additional passwords discovered during the attack:

| Item | Password | Source |
|---|---|---|
| `winrm_backup.zip` | `supremelegacy` | ZIP password cracking |
| `legacyy_dev_auth.pfx` | Cracked with John | PFX password cracking |

---

# 15. Key Findings

## Anonymous SMB Share

The `Shares` SMB share was accessible without credentials.

This exposed files that should not have been publicly accessible.

---

## Password-Protected Backup

The share contained:

    winrm_backup.zip

Although password protected, the password could be cracked offline using John the Ripper.

---

## PFX Certificate

The extracted archive contained:

    legacyy_dev_auth.pfx

After cracking its password, the certificate and private key could be extracted.

These were then used for WinRM authentication.

---

## PowerShell Command History

The initial WinRM user had a PowerShell history file containing credentials for:

    svc_deploy

Sensitive credentials should never be stored in shell history.

---

## LAPS Misconfiguration

The `svc_deploy` account belonged to:

    LAPS_Readers

This provided the necessary permissions to access LAPS-related information.

By querying Active Directory computer objects:

    Get-ADComputer -Filter 'ObjectClass -eq "computer"' -Property *

the `ms-Mcs-AdmPwd` attribute could be observed.

The attribute contained the Administrator password.

---

# 16. Tools Used

- Nmap
- SMBClient
- zip2john
- John the Ripper
- pfx2john
- OpenSSL
- Evil-WinRM
- PowerShell
- Active Directory PowerShell cmdlets
- LAPS

---

# 17. MITRE ATT&CK Techniques

| Technique | Description |
|---|---|
| **T1046** | Network Service Scanning |
| **T1135** | Network Share Discovery |
| **T1552.001** | Unsecured Credentials in Files |
| **T1110** | Brute Force / Password Cracking |
| **T1210** | Exploitation of Remote Services |
| **T1021.006** | Windows Remote Management |
| **T1552.004** | Private Keys |
| **T1087.002** | Domain Account Discovery |
| **T1069.002** | Permission Groups Discovery: Domain Groups |
| **T1078** | Valid Accounts |

---

# 18. Lessons Learned

1. Always enumerate SMB shares, including anonymous and guest-accessible shares.
2. Publicly accessible shares can expose sensitive backup files.
3. Password-protected archives should still be treated as potentially recoverable because their passwords can be cracked offline.
4. PFX files can contain private keys and certificates that may provide alternative authentication mechanisms.
5. Always inspect PowerShell command history after obtaining Windows access.
6. Credentials should never be stored in shell history.
7. Enumerate Active Directory group memberships to identify privilege escalation opportunities.
8. LAPS is designed to improve security by providing unique local Administrator passwords, but excessive permissions can expose those passwords.
9. Active Directory computer objects contain many useful attributes that can be enumerated with PowerShell.
10. The `ms-Mcs-AdmPwd` attribute should be properly protected because it can expose local Administrator passwords to unauthorized users.
11. Certificate-based WinRM authentication can provide a useful foothold when valid PFX credentials are discovered.
12. Always check for privilege escalation opportunities after obtaining access to a domain account.

---

# 19. Final Summary

Timelapse is an Easy Windows machine that demonstrates an Active Directory attack chain beginning with an anonymously accessible SMB share.

The initial enumeration reveals the `Shares` SMB share, which can be accessed without credentials. Inside the share, a password-protected `winrm_backup.zip` archive is discovered.

Using `zip2john` and John the Ripper, the ZIP password is cracked. The archive contains a password-protected PFX certificate. The PFX password is then converted into a crackable format using `pfx2john` and successfully recovered with John.

After decrypting the PFX file, the SSL certificate and private key are extracted using OpenSSL. These credentials allow authentication to the target through WinRM using Evil-WinRM.

Once inside the machine, PowerShell command history reveals credentials for the `svc_deploy` account. Authenticating as `svc_deploy` and enumerating its group memberships shows that the account belongs to `LAPS_Readers`.

Instead of using the `AdmPwd.PS` methodology from the original machine documentation, I directly enumerated Active Directory computer objects using:

    Get-ADComputer -Filter 'ObjectClass -eq "computer"' -Property *

The output exposed the `ms-Mcs-AdmPwd` attribute, which contained the local Administrator password:

    7y!&Z}83&vAdo00t+3M02)J&

Using these credentials with Evil-WinRM resulted in a privileged Administrator session and complete compromise of the machine.

The complete attack can therefore be summarized as:

    Anonymous SMB
          |
          v
    Shares / Dev
          |
          v
    winrm_backup.zip
          |
          v
    zip2john + John
          |
          v
    legacyy_dev_auth.pfx
          |
          v
    pfx2john + John
          |
          v
    Certificate + Private Key
          |
          v
    Evil-WinRM
          |
          v
    Initial Access
          |
          v
    PowerShell History
          |
          v
    svc_deploy Credentials
          |
          v
    LAPS_Readers
          |
          v
    Get-ADComputer -Property *
          |
          v
    ms-Mcs-AdmPwd
          |
          v
    Administrator Password
          |
          v
    Evil-WinRM
          |
          v
    Administrator

**Target:** `10.129.227.113`  
**Machine:** Timelapse  
**Difficulty:** Easy  
**OS:** Windows  
**Domain:** `timelapse.htb`  
**Focus:** SMB Enumeration · Password Cracking · WinRM · PowerShell · Active Directory · LAPS
