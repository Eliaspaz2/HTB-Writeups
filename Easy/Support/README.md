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

_Work in progress..._

---

## ⬆️ Privilege Escalation

_Work in progress..._

---

## 📚 Lessons Learned

_Work in progress..._
