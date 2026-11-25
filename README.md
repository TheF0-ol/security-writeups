# Security Writeups

Hands-on cybersecurity writeups documenting penetration testing,
vulnerable lab assessments, Active Directory work, and security research.

All testing was performed against intentionally vulnerable machines, CTFs,
or systems I own, for educational purposes.

---

## Writeups

### Kerberoasting Attack & Detection (Home Lab)

**Environment:** Self-built Windows Server 2022 DC + Windows 11 client
**Focus:** Full attack chain (ticket request → crack) plus the matching defender-side log evidence

**Skills demonstrated:**
- Kerberoasting (MITRE ATT&CK T1558.003)
- Using Rubeus and John the Ripper
- Correlating attack tool output against real Windows Security event logs

[Read Writeup](./kerberoasting-corp-lab.md)

---

### Active Directory Enumeration (Home Lab)

**Environment:** Self-built Windows Server 2022 DC + Windows 11 client
**Focus:** Domain enumeration as a standard, non-admin user

**Skills demonstrated:**
- Active Directory lab setup (forest promotion, domain join)
- Writing and testing custom enumeration tooling
- Identifying real misconfigurations (no account lockout policy)

[Read Writeup](./ad-enumeration-corp-lab.md)

---

### PortSwigger: IDOR in Chat Transcripts

**Platform:** PortSwigger Web Security Academy
**Difficulty:** Apprentice

**Skills demonstrated:**
- Broken access control / IDOR testing
- Burp Suite request tampering
- Writing up root cause and remediation

[Read Writeup](./portswigger-access-control-idor.md)

---

### Kioptrix Level 2

**Platform:** VulnHub
**Difficulty:** Beginner

Attack chain: Network Enumeration → SQL Injection → Command Injection →
Reverse Shell → Privilege Escalation → Root

**Skills demonstrated:**
- Nmap enumeration
- Web application testing
- SQL Injection and Command Injection
- Reverse shell
- Linux privilege escalation

[Read Writeup](./kioptrix-level-2.pdf)

---

### Kioptrix Level 1

**Platform:** VulnHub
**Difficulty:** Beginner

Reconnaissance, service enumeration, vulnerability identification,
exploitation and privilege verification.

[Read Writeup](./kioptrix-level-1.pdf)

---

### Basic Pentesting 1

**Platform:** VulnHub
**Difficulty:** Beginner

Network discovery, service enumeration, vulnerability research and
exploitation of an outdated service.

**Skills demonstrated:**
- Nmap
- Searchsploit
- Metasploit

[Read Writeup](./basic-pentesting-1.pdf)

---

## What I'm Practicing

- Active Directory attack paths and detection
- Web application and API security
- Authentication and authorization
- Vulnerability assessment
- Network penetration testing
- Linux privilege escalation
- Security reporting

---

## Disclaimer

For education only. Use these techniques only on systems you own or have
written permission to test.

---

## Author

**Shravan Kakkar** | Cybersecurity Student | Web & API Security | Active Directory | VAPT | Lab Practice
