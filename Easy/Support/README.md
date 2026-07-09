# Support

**Platform:** Hack The Box

**Difficulty:** Easy

**Operating System:** Windows

---

## 📖 Overview

Support is an Active Directory-focused Windows machine that introduces LDAP enumeration, BloodHound analysis, and Resource-Based Constrained Delegation (RBCD). The exploitation path demonstrates how excessive Active Directory permissions can be abused to impersonate privileged users through Kerberos.

This machine provides excellent practice with Impacket tooling and reinforces the importance of understanding AD object permissions and delegation attacks.

---

## 🎯 Skills Learned

- LDAP enumeration and information gathering.
- Active Directory enumeration using BloodHound.
- Analysis of Active Directory object permissions.
- Abuse of GenericAll permissions.
- Resource-Based Constrained Delegation (RBCD).
- Kerberos ticket manipulation using Impacket.
- Lateral movement within an Active Directory environment.

---

## 🛠️ Tools Used

- Nmap
- SMBClient
- LDAPSearch
- BloodHound
- bloodhound-python
- Impacket
- Evil-WinRM

---

## 🔍 Enumeration

The initial Nmap scan revealed several services commonly found in an Active Directory environment, including DNS, Kerberos, LDAP, SMB, and WinRM, confirming that the target was a Domain Controller.

SMB enumeration did not expose any writable shares, but it provided valuable information about the domain configuration. The next step focused on LDAP enumeration, where an anonymous query revealed user accounts and organizational information.

Using the recovered credentials, I collected Active Directory data with `bloodhound-python` and analyzed the environment in BloodHound. This identified an attack path based on excessive permissions, ultimately revealing that the `support` account had `GenericAll` privileges over another domain object, opening the door to privilege escalation.
---

## 🚪 Initial Access

After obtaining the credentials stored within the `UserInfo.exe` application, I authenticated against the domain services and gathered additional Active Directory information.

BloodHound analysis revealed that the `support` account had `GenericAll` permissions over another domain object. By abusing these excessive privileges, I configured Resource-Based Constrained Delegation (RBCD), allowing me to request a Kerberos service ticket on behalf of a privileged account.

Using Impacket, I converted the obtained Kerberos ticket into a usable format and authenticated to the target system, successfully gaining access as a domain administrator.

---

## ⬆️ Privilege Escalation

Privilege escalation relied on abusing Resource-Based Constrained Delegation (RBCD). After identifying that the `support` account possessed `GenericAll` permissions over a computer object, I created a controlled machine account and modified the target's delegation settings.

This configuration allowed me to impersonate a privileged domain user by requesting a Kerberos service ticket through Impacket. After exporting the ticket into a compatible format, I authenticated using Kerberos and obtained a shell with Domain Administrator privileges, completing the compromise of the Active Directory environment.

---

## 📚 Lessons Learned

- Active Directory permissions can be just as critical as software vulnerabilities and often provide alternative attack paths.
- BloodHound is an invaluable tool for identifying privilege escalation opportunities that are difficult to detect manually.
- Resource-Based Constrained Delegation (RBCD) is a powerful Kerberos attack that can lead to full domain compromise when object permissions are misconfigured.
- Proper credential management is essential, as hardcoded credentials within applications can expose sensitive domain accounts.
- Understanding the logic behind Kerberos authentication is just as important as knowing how to use offensive tooling such as Impacket.
