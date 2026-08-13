# TryHackMe - Fools-mate Room Writeup

## Overview

This is a detailed walkthrough of how I captured the flag of **Fools-mate** room on TryHackMe. The room involved Enumeration, Understanding how the application works, Bypassing application logic, Manual Web Exploitation and Critical Thinking.

---

## Enumeration

I started the engagement with an `nmap` scan:

```bash
nmap -sC -sV -oN initial_scan.txt <target-ip>
```

The scan revealed two open ports:

- **22** (SSH)
- **80** (HTTP)

---

## Web Application Discovery

I started with basic enumeration. A quick Nmap scan did not reveal anything useful beyond the web service, so I moved on to manual web inspection.

When opening the page in the browser, the application displayed a chess board game.

The position was a simple rook endgame puzzle. White had a rook on a1, and the black king was trapped on g8 by its own pawns.

The intended winning move was:
from: a1
to: a8

I inspected the page source and JavaScript files. The main frontend logic was loaded from `/js/app.js`

Then i search for the text `shut`, i find the function `preMoveCheck` validating at the frontend.
---

## Exploitation
Continuing through the JavaScript file, I found that moves were submitted to the  backend with the api POST `api/move`.

I open the burpsuite to bypass the javascript validation on the frontend, then i captured the request and modified the json body to `from: a1` , `to: a8` and forwarded the request, the server responded with the flag without any validation and check.

---

🖼️ **Find all screenshots here:** [`screenshots/fools_mate/`](../screenshots/fools_mate/)


### Tools used:
--------------
- Gobuster 
- Firefox Developer Tools
- Burp Suite


## Summary & Learning Objectives

This room focuses on a typical and basic web application assessment workflow:

## Core Topics

1.Enumeration

 - Explore the website.
 - Identify pages, forms, and hidden functionality.
 - Inspect requests and responses.
 - Use browser Developer Tools and/or Burp Suite.

2.Understanding how the application works

 - Observe how user input is processed.
 - Look for filters or restrictions.
 - Identify where security checks are performed.

3.Bypassing application logic

 - The title "Can you bypass the engine?" is the biggest clue.
 - Rather than exploiting an outdated vulnerability, you'll likely need to bypass a validation or filtering mechanism.
 - This teaches logic flaws rather than just CVE exploitation.

4.Manual Web Exploitation

 - Modifying HTTP requests
 - Changing parameters
 - Trying alternative encodings
 - Testing edge cases
 - Looking for hidden endpoints
 - Understanding server responses

5.Critical Thinking

 - Observe
 - Hypothesize
 - Test
 - Repeat



*Thanks for reading!*
