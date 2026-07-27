# Arctic — Hack The Box

## Machine Information

| Field                    | Value                    |
| ------------------------ | ------------------------ |
| **Platform**             | Hack The Box             |
| **Machine**              | Arctic                   |
| **Difficulty**           | Easy                     |
| **Operating System**     | Windows                  |
| **Initial User**         | `tolis`                  |
| **Privilege Escalation** | Unpatched Windows system |
| **Final User**           | `NT AUTHORITY\SYSTEM`    |

---

# Initial Enumeration

The first step was to scan the target for open ports using Nmap.

Rather than immediately performing service enumeration and script scanning against all 65,535 ports, I first performed a full TCP port scan to identify the open ports:

```bash
sudo nmap -p- --min-rate=10000 -oA allports.txt -v <TARGET_IP>
```

Once the open ports were identified, I performed a more detailed scan against the discovered services:

```bash
sudo nmap -sC -sV -Pn -T4 -p 135,8500,49154 <TARGET_IP>
```

The scan revealed the following open ports:

```text
135/tcp     open  msrpc
8500/tcp    open  http
49154/tcp   open  msrpc
```

Port `8500` was running an HTTP server, so I focused my enumeration on the web application.

---

# Web Exploitation

## Adobe ColdFusion 8

I accessed the HTTP service running on port `8500`:

```text
http://<TARGET_IP>:8500
```

The web server exposed two different directories.

After enumerating the available content, I discovered a login page for:

```text
Adobe ColdFusion 8
```

I first attempted to use default credentials, but they were unsuccessful.

Since the application was running an outdated version of Adobe ColdFusion, I searched for known exploits affecting the service.

---

## Directory Traversal

I found a directory traversal vulnerability affecting the ColdFusion installation.

I tested the exploit against the target, confirming that the web application was vulnerable.

By exploiting the directory traversal vulnerability, I was able to access sensitive files outside the intended web directory.

Inspecting the returned content and its source code revealed a password hash.

The hash was then submitted to CrackStation:

```text
https://crackstation.net/
```

The hash was successfully cracked, revealing the password required to access the ColdFusion administrator dashboard.

---

## ColdFusion Administrator Access

Using the recovered credentials, I logged into the Adobe ColdFusion administrator panel.

After gaining access to the dashboard, I enumerated the available functionality but did not initially find anything particularly interesting.

However, during the vulnerability research phase, I had also identified an exploit involving the upload of a `.jsp` file.

---

## JSP File Upload

The ColdFusion application allowed a JSP file to be uploaded and executed by the server.

I modified the JSP file to execute a reverse shell payload.

After uploading the modified JSP file and executing it through the web application, I obtained a foothold on the target machine.

The initial access was obtained as:

```text
tolis
```

I verified the current user:

```cmd
whoami
```

I then retrieved the user flag:

```text
C:\Users\tolis\Desktop\user.txt
```

---

# Privilege Escalation

## Post-Foothold Enumeration

After obtaining initial access as `tolis`, I began performing standard post-exploitation enumeration.

I first gathered information about the Windows operating system:

```cmd
systeminfo
```

The system information revealed that the Windows installation had no relevant updates or hotfixes installed.

This indicated that the system could potentially be vulnerable to known local privilege escalation exploits.

---

## Creating a More Stable Reverse Shell

To create a better and more reliable connection, I generated a reverse shell payload and transferred it to the Windows machine.

I created a `tools` directory on the target to store the required files.

On my Kali machine, I hosted the payload using a Python HTTP server:

```bash
python3 -m http.server 8000
```

The payload was then transferred to the Windows machine and executed.

I configured a Netcat listener on my Kali machine:

```bash
rlwrap -cAr nc -lvnp 4444
```

This provided a more stable reverse shell and also allowed me to quickly regain access if the original shell was dropped.

---

## Local Privilege Escalation

After creating a more reliable connection, I continued with the post-exploitation enumeration.

The output of:

```cmd
systeminfo
```

showed that the Windows operating system did not have the necessary updates or hotfixes installed.

Based on this information, I identified a local privilege escalation exploit affecting the system.

I transferred the exploit binary to the target machine and executed it.

After executing the binary, the exploit successfully escalated my privileges to:

```text
NT AUTHORITY\SYSTEM
```

I verified the elevated privileges:

```cmd
whoami
```

The final flag was located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

---

# Attack Path Summary

```text
Full TCP Port Scan
        ↓
Ports 135, 8500 and 49154 discovered
        ↓
Port 8500 — HTTP
        ↓
Adobe ColdFusion 8 identified
        ↓
ColdFusion directory traversal
        ↓
Sensitive file accessed
        ↓
Password hash discovered
        ↓
Hash cracked with CrackStation
        ↓
ColdFusion Administrator Access
        ↓
JSP file upload
        ↓
Code execution
        ↓
Initial foothold as tolis
        ↓
User flag
        ↓
Post-foothold enumeration
        ↓
systeminfo
        ↓
No Windows updates or hotfixes
        ↓
Reverse shell payload transferred
        ↓
Local privilege escalation exploit
        ↓
NT AUTHORITY\SYSTEM
        ↓
Root flag
```

---

# Key Takeaways

## Enumeration

* Start by identifying open ports before performing deeper service enumeration.
* Once the open ports are known, perform targeted service and version detection.
* Outdated software versions can provide valuable attack vectors.

## Web Exploitation

* Adobe ColdFusion 8 was running an outdated web application.
* Directory traversal vulnerabilities can expose sensitive files outside the intended web directory.
* Password hashes obtained during exploitation can potentially be cracked offline.
* File upload functionality should always be examined for possible code execution.

## Post-Exploitation

* `systeminfo` is an important command for Windows post-exploitation enumeration.
* Missing Windows updates and hotfixes can indicate potential local privilege escalation vulnerabilities.
* A more stable reverse shell can make further enumeration and exploitation easier.

## Privilege Escalation

* Transferring and executing a local exploit can result in a SYSTEM-level shell when the target is vulnerable.
* The final privilege escalation resulted in:

```text
NT AUTHORITY\SYSTEM
```

---

# Tools Used

* Nmap
* CrackStation
* Python HTTP Server
* Netcat
* rlwrap
* Windows Command Prompt
* PowerShell
* Local Windows Privilege Escalation Exploit

---

# Conclusion

Arctic was a great machine for practicing a complete Windows penetration testing workflow.

The initial attack path consisted of:

1. Performing a full TCP port scan.
2. Identifying the HTTP service on port `8500`.
3. Discovering Adobe ColdFusion 8.
4. Exploiting a directory traversal vulnerability.
5. Obtaining a password hash.
6. Cracking the hash with CrackStation.
7. Accessing the ColdFusion administrator dashboard.
8. Uploading a malicious `.jsp` file.
9. Obtaining initial access as `tolis`.
10. Retrieving the user flag.

For privilege escalation:

1. I performed post-foothold enumeration.
2. Used `systeminfo` to identify the unpatched Windows system.
3. Created a more stable reverse shell connection.
4. Transferred a local privilege escalation exploit to the target.
5. Executed the exploit.
6. Successfully escalated to:

```text
NT AUTHORITY\SYSTEM
```

This allowed me to retrieve the final root flag and complete the machine successfully.
