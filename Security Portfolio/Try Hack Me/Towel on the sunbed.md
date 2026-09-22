# Penetration Testing & Vulnerability Assessment Report: Towel on the Sunbed (Hacker Holidays: Day 8)

**Target System:** Towel on the Sunbed / Hacker Holidays (Day 8)  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Vulnerability Focus:** Business Logic Flaw & Race Condition (Concurrency Exploit)  
**Prepared By:** Security Operations  

---

## Executive Summary

A penetration testing assessment was performed on the **Towel on the Sunbed (Hacker Holidays: Day 8)** room on TryHackMe. The primary objective was to audit the application's transaction logic, evaluate the daily reward claim mechanism, and identify pathways to bypass access controls to retrieve the root flag located inside the restricted vault.

The assessment identified a critical **Race Condition / Business Logic Vulnerability** in the coin reward claim mechanism. By sending concurrent, parallelized HTTP requests via Burp Suite Repeater, the server failed to validate session state quickly enough, allowing 5 identical requests to execute simultaneously. This multiplied the reward from 50 to 250 Ponzi Coins, enabling the instant unlocking of the vault and full system compromise.

---

## Vulnerability Summary Table

| Finding ID | Vulnerability Title | Severity | CVSS v3.1 Score | Primary Impact |
| :--- | :--- | :--- | :--- | :--- |
| **THM-01** | Business Logic Flaw via Concurrency / Race Condition | **High** | **8.1** | Unauthorized Currency Generation |
| **THM-02** | Broken Access Control / Vault Flag Disclosure | **High** | **7.5** | Privilege Escalation / Asset Access |

---

## Detailed Vulnerability Findings & Remediation

### Finding THM-01: Business Logic Flaw via Race Condition on Coin Reward Endpoint

* **Severity:** High  
* **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N` (**8.1**)  
* **Vulnerable Mechanism:** Daily Ponzi Coin Reward Request  

#### Description
The web application provides users with a daily allowance of 50 Ponzi Coins. However, the backend suffers from a **Time-of-Check to Time-of-Use (TOCTOU)** race condition. The server checks if the user has already claimed their reward before updating the database state, but fails to handle concurrent asynchronous threads safely. 

By cloning and transmitting multiple identical claim requests simultaneously, all incoming requests pass the initial verification check before any single thread marks the reward as "claimed," resulting in duplicate currency payouts.

#### Technical Proof of Concept
1. Intercept the daily coin claim HTTP request using **Burp Suite**.
2. Send the captured request to **Burp Suite Repeater**.
3. Create 5 duplicate tabs (clones) of the exact same request.
4. Create a request group in Repeater containing all 5 requests.
5. Execute the group simultaneously using the **"Send group in parallel (single-packet attack / parallel connection)"** feature.

```http
POST /api/v1/claim-reward HTTP/1.1
Host: <TARGET_IP>
Authorization: Bearer <SESSION_TOKEN>
Content-Type: application/json

{}

```

**Result:**

All 5 parallel requests returned successful `200 OK` responses, bypassing the intended single-use limit and granting **250 Ponzi Coins** ($5 \times 50$) instantaneously.

#### Remediation & Mitigation

* **Atomic Database Transactions:** Execute the check and state update within a single atomic transaction using database locks (`SELECT ... FOR UPDATE`).
* **Conditional Update Statements:** Perform the reward update conditionally at the database level:
```sql
UPDATE users 
SET balance = balance + 50, reward_claimed = TRUE 
WHERE user_id = 123 AND reward_claimed = FALSE;

```


* **Distributed Mutex Locking:** Implement server-side distributed locks (e.g., using Redis) indexed by `user_id` on sensitive transaction endpoints.

---

### Finding THM-02: Broken Access Control & Vault Flag Extraction

* **Severity:** High
* **CVSS v3.1 Vector:** `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` (**7.5**)
* **Vulnerable Endpoint:** Vault Unlock / In-Game Store

#### Description

Access to the target vault (containing the flag) is guarded strictly by an arbitrary balance threshold (200+ Ponzi Coins). Because the application lacks server-side verification of how the currency was accumulated, the exploited balance gained via **THM-01** was immediately accepted by the vault unlock logic.

#### Technical Proof of Concept

1. Submit a request to unlock or open the vault using the artificially inflated balance of 250 Ponzi Coins.
2. The server confirms the balance threshold (`250 >= required_cost`) and returns the flag.

```http
POST /api/v1/vault/open HTTP/1.1
Host: <TARGET_IP>
Authorization: Bearer <SESSION_TOKEN>
Content-Type: application/json

{"vault_id": "main_vault"}

```

#### Remediation & Mitigation

* **Transaction History Auditing:** Ensure sensitive balance-check operations validate the transaction history ledger rather than an isolated, mutable integer field.
* **Fix Root Business Logic Flaw:** Resolving **THM-01** natively prevents access to this vector.

---

## Strategic Recommendations

1. **Conduct Concurrency Audit:** Review all state-changing endpoints (purchases, transfers, reward claims) for race conditions using automated tools like Burp's Turbo Intruder.
2. **Implement Rate Limiting:** Enforce a strict rate limit per user session on API endpoints handling virtual items or currency.


