
<div align="center">

# Meerkat

![Category](https://img.shields.io/badge/Category-Digital%20Forensics-blue)
![Platform](https://img.shields.io/badge/Platform-Network%20Analysis-0078D6)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Focus](https://img.shields.io/badge/Focus-Incident%20Response-red)

**HTB Sherlock — Network Forensics & Incident Response**

</div>

---

## Machine Overview

| Field | Details |
|---|---|
| Name | Meerkat |
| Difficulty | Easy |
| Platform | Network Forensics |
| Primary Application | BonitaSoft |
| Vulnerability | CVE-2022-25237 |
| Main Artifacts | PCAP and Zeek alerts |
| Objective | Reconstruct the attack, identify compromised credentials, and determine the persistence mechanism |

## Artifacts

| Artifact | SHA-256 |
|---|---|
| `meerkat.zip` | `a76867ade304c8081e2023cbf2977c65e8c146180b2e0ff760e4059d042c2a5a` |
| `meerkat.pcap` | `aa3838dbd634f9798d1e9505d243a4fee1d340d6e25e2f0c9648dd64e2178dbf` |
| `meerkat-alerts.json` | `012aa4e8aae5d500c001510d6e65567eb0cdbfffe2dab9a119b66f7770c222be` |

Archive password: `hacktheblue`

## 1. Initial Alert Analysis

The Zeek alerts were parsed to identify the application and suspicious activity:

    jq '.[].alert.signature' meerkat-alerts.json

The alert signatures indicated exploitation attempts against **BonitaSoft**, a business process management platform.

To isolate alerts mentioning CVEs:

    jq '.[].alert.signature' meerkat-alerts.json | grep CVE

The vulnerability associated with the attack was **CVE-2022-25237**, an authorization bypass affecting BonitaSoft.

## 2. Credential Stuffing Investigation

### Identify the attack traffic

The PCAP was opened in Wireshark and the HTTP conversations were examined. Most relevant traffic originated from `138.199.59.221` and targeted `172.31.6.44:8080`.

The login requests could be isolated with:

    http.request.uri == "/bonita/loginservice" &&
    http.user_agent == "python-requests/2.28.1"

The user-agent was:

    python-requests/2.28.1

### Extract attempted credentials

`tshark` and `jq` were used to extract and deduplicate the submitted HTTP data:

    tshark -Y '(http.request.uri == "/bonita/loginservice") &&
    (http.user_agent == "python-requests/2.28.1")' \
    -r meerkat.pcap -T json |
    jq '.[]._source.layers.http."http.file_data"' |
    sort | uniq

The output contained **57 unique username/password combinations**. However, the exploit repeatedly attempted the application's default credentials, `install:install`. Excluding that repeated default attempt leaves **56 credential combinations** attributable to the credential-stuffing activity.

### Determine the successful credentials

Comparing HTTP conversations showed that some requests received `401 Unauthorized`, while other conversations continued into exploitation activity.

The successful credentials were identified as:

    Username: seb.broom%40forela.co.uk
    Password: g0vernm3nt

The username was URL-encoded. Decoding `%40` yields the actual account:

    seb.broom@forela.co.uk

**Confirmed credentials:**

    seb.broom@forela.co.uk:g0vernm3nt

The evidence supports **Credential Stuffing**, rather than password guessing, password cracking, or password spraying.

## 3. Authorization Bypass

The vulnerable API request contained the additional path string:

    ;i18ntranslation

The affected endpoint was:

    /bonita/API/pageUpload;i18ntranslation

This string was used to bypass the authorization filter in the exploitation of CVE-2022-25237.

## 4. Post-Exploitation Activity

Reviewing the HTTP requests and following the relevant streams revealed commands executed through the exploit. The activity included an initial `whoami` command, followed by a request to retrieve a payload from:

    pastes.io

The text-sharing service was used to host the next stage of the attack.

## 5. Persistence via SSH Authorized Keys

The payload retrieved from `pastes.io` instructed the system to copy the contents of a file named:

    hffgra4unv

into the SSH authorized-keys file:

    /home/ubuntu/.ssh/authorized_keys

The referenced file contained the attacker's SSH public key. Adding it to `authorized_keys` enabled SSH authentication using the corresponding private key, establishing a persistence mechanism.

The source writeup used the Wayback Machine to recover the hosted payload when investigating its historical contents.

**MITRE ATT&CK mapping:**

| Technique | ID | Evidence |
|---|---|---|
| Account Manipulation: SSH Authorized Keys | T1098.004 | Attacker-controlled public key added to `authorized_keys` |

## 6. Indicators of Compromise

### Network

| Indicator | Value |
|---|---|
| Source IP observed in the credential attack | `138.199.59.221` |
| Target server | `172.31.6.44` |
| Target port | `8080` |
| Text-sharing service | `pastes.io` |
| HTTP user-agent | `python-requests/2.28.1` |

### Application and Exploitation

| Indicator | Value |
|---|---|
| Application | BonitaSoft |
| CVE | `CVE-2022-25237` |
| Login endpoint | `/bonita/loginservice` |
| Authorization bypass string | `i18ntranslation` |
| Default credentials repeatedly attempted | `install:install` |
| Successful account | `seb.broom@forela.co.uk` |

### Persistence

| Indicator | Value |
|---|---|
| Downloaded key file | `hffgra4unv` |
| Modified file | `/home/ubuntu/.ssh/authorized_keys` |
| MITRE ATT&CK | `T1098.004` |

## 7. Detection and Hunting Recommendations

- Alert on repeated authentication attempts against `/bonita/loginservice`, especially when associated with scripted HTTP clients.
- Detect repeated username/password combinations across login requests and distinguish credential stuffing from password spraying.
- Investigate suspicious requests to `/bonita/API/` containing `;i18ntranslation`.
- Review successful logins followed by unexpected API activity or command execution.
- Monitor outbound requests to public text-sharing services when initiated by application servers.
- Audit changes to `/home/*/.ssh/authorized_keys`, particularly when a new public key is added by an unexpected process or user.
- Correlate network telemetry with host-level authentication and file-integrity events to confirm post-exploitation behavior.

## 8. Attack Chain

    Credential Stuffing
            |
            v
    BonitaSoft Authentication
            |
            v
    CVE-2022-25237
    Authorization Bypass
            |
            v
    Exploitation and Command Execution
            |
            v
    Payload Retrieved from pastes.io
            |
            v
    Public Key Added to authorized_keys
            |
            v
    SSH Persistence

## Summary

The investigation identified a credential-stuffing attack against BonitaSoft followed by exploitation of CVE-2022-25237. The attacker used valid credentials to progress beyond failed authentication attempts, exploited an authorization bypass, and executed commands through the vulnerable application.

The post-exploitation activity involved retrieving a payload from `pastes.io` and adding an attacker-controlled SSH public key to `/home/ubuntu/.ssh/authorized_keys`. This provided a persistence mechanism mapped to MITRE ATT&CK `T1098.004`.

## Key Takeaways

- Zeek alerts provide an efficient starting point for identifying suspicious application activity.
- PCAP analysis can expose authentication attempts and application exploit requests when traffic is unencrypted.
- Deduplicating extracted request data helps quantify credential-stuffing attempts while accounting for repeated default credentials.
- Historical web archives can help recover deleted or unavailable payloads.
- SSH authorized-key changes are important persistence indicators that should be monitored on Linux systems.

## Skills Demonstrated

- PCAP analysis with Wireshark and `tshark`
- Zeek alert analysis using `jq`
- HTTP authentication traffic inspection
- Credential-stuffing identification
- CVE-based application exploitation analysis
- Payload investigation using the Wayback Machine
- Linux SSH persistence analysis
- MITRE ATT&CK mapping
- Incident chain reconstruction
