# Hack The Box - Escape

![Windows](https://img.shields.io/badge/OS-Windows-0078D6?style=for-the-badge&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-Hack%20The%20Box-9FEF00?style=for-the-badge&logo=hackthebox)
![Category](https://img.shields.io/badge/Category-Active%20Directory-red?style=for-the-badge)

## 📌 Machine Information

| Property | Value |
|---|---|
| **Machine** | Escape |
| **Platform** | Hack The Box |
| **Operating System** | Windows |
| **Difficulty** | Medium |
| **Domain** | `sequel.htb` |
| **Hostname** | `dc.sequel.htb` |
| **Focus** | SMB, MSSQL, NTLM, Kerberos and Active Directory Certificate Services |
| **Techniques** | SMB Enumeration, MSSQL Authentication, NTLM Hash Capture, Password Cracking, Credential Reuse, ESC1, Certificate Authentication and Pass-the-Hash |

> ⚠️ **Disclaimer:** This writeup was created for educational purposes and documents the exploitation of a retired Hack The Box machine. All techniques were performed in an authorized lab environment.

---

# 🧭 Attack Path

```text
SMB Anonymous Access
        │
        ▼
Download "SQL Server Procedures.pdf"
        │
        ▼
Obtain MSSQL Credentials
        │
        ▼
Connect to MSSQL
        │
        ▼
Force NTLM Authentication with xp_dirtree
        │
        ▼
Capture sql_svc NTLMv2 Hash
        │
        ▼
Crack Password with John the Ripper
        │
        ▼
WinRM Access as sql_svc
        │
        ▼
Find MSSQL Backup Log
        │
        ▼
Obtain ryan.cooper Credentials
        │
        ▼
WinRM Access as ryan.cooper
        │
        ▼
Enumerate Active Directory Certificate Services
        │
        ▼
Identify ESC1-Vulnerable Certificate Template
        │
        ▼
Request Administrator Certificate
        │
        ▼
Authenticate with Certipy
        │
        ▼
Obtain Administrator NT Hash
        │
        ▼
Pass-the-Hash via WinRM
        │
        ▼
Administrator Access
