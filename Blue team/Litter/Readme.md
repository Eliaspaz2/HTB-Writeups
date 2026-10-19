
<div align="center">

# 🕵️ HTB Sherlock: Litter

![Category](https://img.shields.io/badge/Category-Network%20Forensics-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Platform](https://img.shields.io/badge/Platform-HTB-black)
![Focus](https://img.shields.io/badge/Focus-DNS%20Tunneling%20%7C%20Data%20Exfiltration-red)

**Network Forensics | DNS Tunneling | PII Exfiltration**

</div>

---

## 📌 Overview

Litter is an Easy-difficulty HTB Sherlock focused on network traffic analysis and data exfiltration detection. The investigation centers on a PCAP containing suspicious DNS traffic between two internal hosts.

By analyzing DNS queries and decoding hexadecimal subdomains, the attacker’s activity was reconstructed, the stolen file was identified, and the total number of exfiltrated customer records was determined.

## 🧰 Tools Used

- **Brim / Zui** — Initial PCAP overview and traffic exploration.
- **Wireshark** — Protocol statistics, DNS query inspection, and traffic filtering.
- **tcpdump** — DNS traffic extraction from the original PCAP.
- **Python 3 + Scapy** — Packet parsing and hexadecimal payload decoding.
- **CyberChef** — Hexadecimal-to-text conversion.

## 📂 Evidence

| Artifact | Details |
|---|---|
| `suspicious_traffic.pcap` | Primary network capture |
| SHA-256 | Not provided in the source write-up |
| MD5 | `98c26da32ebc0518f327636f5bad777d` |

## 🔎 Investigation

### 1. Initial Traffic Analysis

The initial PCAP review identified an unusually high volume of DNS traffic. DNS represented approximately 20% of the capture, which was notable given the PCAP's reported size of around 40 MB.

The investigation identified substantial DNS communication between:

- `192.168.157.144`
- `192.168.157.145`

The suspicious host identified in the Sherlock questions was `192.168.157.145`.

### 2. Detecting DNS Tunneling

Inspection of DNS query names revealed long hexadecimal strings embedded in subdomains associated with `microsofto365.com`.

This pattern, combined with the volume of DNS traffic, indicated probable DNS tunneling. The decoded values contained comma-separated data consistent with personally identifiable information (PII).

The DNS tunneling tool identified in the investigation was **dnscat**, version `0.07`.

### 3. Extracting DNS Traffic

The DNS packets were extracted into a separate PCAP to simplify the analysis:

    tcpdump -nr suspicious_traffic.pcap port 53 -w dns.pcap

A Python script using Scapy then iterated through DNS question records, extracted query names, removed the expected prefix and domain suffix, and converted hexadecimal content into text.

    from scapy.all import *

    r = rdpcap("dns.pcap")

    with open("output.txt", "w") as myfile:
        c = b""

        for packet in r:
            if packet.haslayer(DNSQR):
                a = packet[DNSQR].qname
                no9 = a[18:]
                b = no9.replace(b"microsofto365.com.", b"")

                if not b or b == c:
                    continue

                c = b

                try:
                    hex_chars = "".join(
                        ch for ch in b.decode(errors="ignore")
                        if ch in "0123456789abcdefABCDEF"
                    )

                    ascii_str = bytes.fromhex(hex_chars).decode(
                        "utf-8", errors="ignore"
                    )

                    myfile.write(ascii_str + "\n")

                except ValueError as e:
                    print(f"Error converting hex to ASCII: {e}")

The resulting output contained the commands executed by the attacker and the data associated with the exfiltration.

**Note:** The script above preserves the source write-up's extraction logic. Its prefix/suffix assumptions and handling of hexadecimal fragments may need adjustment when applied to other captures.

### 4. Reconstructing Attacker Activity

The recovered commands showed the following sequence:

1. Executed `whoami` to identify the current user.
2. Listed the contents of the Downloads directory.
3. Renamed the `dnscat` binary to `win_installer.exe`.
4. Navigated to the user's home directory.
5. Entered the OneDrive directory.
6. Navigated to Documents.
7. Opened the `client data optimisation` directory.
8. Read the contents of `user details.csv`.

This sequence established that the attacker accessed a file containing customer information and transferred its contents through the suspicious DNS channel.

### 5. Identifying the Stolen Data

The exfiltrated file was located at:

    C:\Users\test\Documents\client data optimisation\user details.csv

The CSV contained fields including:

- Job and company
- Social Security Number (SSN)
- Residence and current location
- Blood group
- Website and username
- Name and sex
- Address and email
- Birthdate

The investigation confirmed that **721 customer PII records** had been exfiltrated.

## 🚨 Key Findings

| Finding | Result |
|---|---|
| Suspicious protocol | DNS |
| Suspect host | `192.168.157.145` |
| Other communicating host | `192.168.157.144` |
| First attacker command | `whoami` |
| DNS tunneling tool | `dnscat` |
| Tool version | `0.07` |
| Renamed binary | `win_installer.exe` |
| Exfiltrated file | `user details.csv` |
| Confirmed records stolen | `721` |

## 🧭 Attack Chain

    Suspicious DNS Traffic
             |
             v
    Hexadecimal Subdomains
             |
             v
    DNS Tunneling via dnscat
             |
             v
    Host and Directory Enumeration
             |
             v
    Access to Customer CSV
             |
             v
    PII Exfiltration over DNS
             |
             v
    721 Records Compromised

## 🎯 MITRE ATT&CK Mapping

| Technique | ID | Evidence |
|---|---|---|
| Command and Scripting Interpreter | T1059 | Attacker commands were recovered from the DNS traffic. |
| Application Layer Protocol: DNS | T1071.004 | DNS was used as the communication channel. |
| Data from Local System | T1005 | The attacker accessed a local customer data CSV. |
| Exfiltration Over Web Service / Alternative Protocol | — | Data was exfiltrated through DNS tunneling; no specific exfiltration technique ID is asserted here. |

*Mappings are analytical classifications based on the observed behavior, rather than additional findings explicitly reported in the source write-up.*

## 🛡️ Defensive Recommendations

- Monitor DNS query volume and unusually long or high-entropy subdomains.
- Investigate repeated hexadecimal-looking DNS labels and anomalous domain patterns.
- Restrict unauthorized DNS traffic and enforce approved recursive resolvers.
- Alert on suspicious execution or renaming of tunneling utilities.
- Apply least-privilege access to sensitive customer files.
- Monitor unusual access to cloud-synchronization directories and customer datasets.
- Encrypt sensitive information at rest and minimize unnecessary PII exposure on testing hosts.

## 📚 Key Takeaways

- DNS traffic can conceal command-and-control activity and data exfiltration.
- Long hexadecimal subdomains are a useful indicator of potential DNS tunneling.
- Extracting DNS packets into a separate PCAP makes targeted analysis easier.
- Scapy can automate the reconstruction of data embedded in DNS queries.
- Correlating reconstructed commands with decoded payloads can establish the scope of a data breach.
- In this case, the investigation confirmed the theft of **721 customer records**.

## 🧠 Skills Demonstrated

- PCAP analysis
- DNS tunneling detection
- Wireshark display filtering
- tcpdump packet extraction
- Scapy-based packet parsing
- Hexadecimal payload decoding
- Command reconstruction
- PII exfiltration investigation
- Incident impact assessment

---

<div align="center">

**HTB Sherlock: Litter**  
*Network Forensics • DNS Tunneling • Data Exfiltration*

</div>
