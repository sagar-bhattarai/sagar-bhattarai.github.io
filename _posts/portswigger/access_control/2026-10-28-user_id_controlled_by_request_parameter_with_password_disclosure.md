---
title: "Insecure Direct Object References (IDOR) - PortSwigger"
date: 2026-10-28 13:02:20 +0545

description: "Exploiting an insecure direct object reference in a live chat transcript feature to access another user's conversation and retrieve credentials."

categories: [PortSwigger, Access Control]
tags: [burpsuite, idor]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/idor.png
  alt: "Insecure Direct Object References - PortSwigger"

platform: PortSwigger

lab: "Insecure direct object references"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how broken access control can expose sensitive information when an application uses predictable file identifiers without properly verifying authorization. By intercepting a live chat transcript request in Burp Suite and changing the requested filename from `2.txt` to `1.txt`, I accessed another user's conversation and discovered their password. I then used the credentials to log in as the target user and complete the lab.

# Question / Lab Objective

The objective was to exploit an insecure direct object reference (IDOR) in the live chat transcript functionality to obtain Carlos's password and log in to his account.

![Lab question](/assets/images/writeups/portswigger/access_control/idor/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed several products, an account section, and a **Live Chat** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/idor/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

## 2. Accessing the live chat page

I clicked **Live Chat** and started a conversation with Hal Pline. While exploring the chat interface, I noticed an option to view or download the conversation transcript.

![Live chat interface](/assets/images/writeups/portswigger/access_control/idor/3.png)

This feature became the starting point for my investigation because transcript downloads often involve a request parameter or filename that identifies the requested conversation.

## 3. Intercepting the transcript request in Burp Suite

I clicked the option to view the transcript. The application downloaded a text file named `4.txt`, containing the conversation.

I examined the corresponding HTTP request and response in Burp Suite. In the response, I could see the chat transcript returned by the server.

![Downloaded transcript and response in Burp Suite](/assets/images/writeups/portswigger/access_control/idor/4.png)

The filename `4.txt` caught my attention because it appeared to use a predictable numeric identifier. I decided to investigate whether changing that identifier would allow access to a different transcript.

## 4. Modifying the transcript filename

I sent the transcript request to Burp Suite's **Repeater** tool so that I could modify the request and inspect the server's response without repeatedly interacting with the browser.

I changed the requested filename from `4.txt` to `1.txt` and sent the modified request.

![Modified transcript request and response in Repeater](/assets/images/writeups/portswigger/access_control/idor/5.png)

The server returned a different chat transcript. While examining the response, I found a conversation containing Carlos's password.

This demonstrated the access control weakness: the application allowed me to request a different transcript by changing a predictable identifier, without properly checking whether I was authorized to access that conversation.

**Key finding:** Changing the filename from `4.txt` to `1.txt` exposed another user's chat transcript and sensitive credentials.

## 5. Logging in with Carlos's credentials

After retrieving Carlos's password, I returned to the application's login page and entered the disclosed credentials to authenticate as `carlos`.

![Logging in with Carlos's credentials](/assets/images/writeups/portswigger/access_control/idor/6.png)

The application accepted the credentials and granted access to Carlos's account, confirming that the disclosed information could be used to compromise another user's account in this lab.

## 6. Confirming lab completion

After logging in successfully, I was redirected to Carlos's account page, and the lab displayed the solved confirmation.

![Lab solved confirmation](/assets/images/writeups/portswigger/access_control/idor/7.png)

This confirmed that I had successfully exploited the insecure transcript access and completed the lab's objective.

# Key Takeaways

- **Understand IDOR:** An insecure direct object reference occurs when an application exposes a reference to an object, such as a filename or record ID, without adequately enforcing authorization.
- **Inspect file downloads:** Download and transcript features can reveal how applications identify and retrieve user-specific resources.
- **Look for predictable identifiers:** Sequential filenames such as `1.txt` and `2.txt` can make resources easy to enumerate, although predictability alone does not establish a vulnerability.
- **Use Burp Suite Repeater:** Repeater allows individual request parameters to be modified and server responses to be compared systematically.
- **Check authorization, not just authentication:** Being logged in should not automatically grant access to every user's files or records.
- **Recognize the impact of exposed credentials:** A transcript disclosure can become an account compromise when sensitive authentication information is included in the conversation.

# Security Impact

An IDOR vulnerability can allow an authenticated user to access another user's private resources by modifying an object identifier in an HTTP request.

In this lab, the transcript endpoint returned another user's conversation when the filename was changed. Because the conversation contained Carlos's password, the disclosure escalated from unauthorized access to a transcript to account compromise.

In a real-world application, similar weaknesses could expose private messages, personal information, account records, documents, or other sensitive data. The severity depends on the sensitivity of the exposed information, the number of affected users, and whether the disclosed data enables further unauthorized access.

# Remediation

To prevent IDOR vulnerabilities:

1. **Enforce server-side authorization:** Verify that the authenticated user has permission to access the specific transcript or resource requested.
2. **Validate every object request:** Do not assume that a user is authorized simply because they are logged in or know a valid filename.
3. **Avoid exposing credentials in chat transcripts:** Passwords and other authentication secrets should not be shared or stored in conversation histories.
4. **Use secure resource-handling practices:** Opaque identifiers can make enumeration more difficult, but they must never replace proper authorization checks.
5. **Apply least privilege:** Users should only be able to access resources they are explicitly authorized to view.
6. **Test for horizontal privilege escalation:** During authorized security testing, verify that changing an object identifier does not expose another user's data.

# Conclusion

This lab demonstrated how a seemingly simple live chat transcript feature can introduce a serious access control vulnerability. By intercepting the transcript request in Burp Suite, changing the filename from `2.txt` to `1.txt`, and inspecting the response, I retrieved Carlos's password and gained access to his account.

The key lesson is that applications must verify authorization for every requested resource. Predictable identifiers can make testing easier, but the underlying security failure is the absence of proper server-side access control.

*Thanks for reading!*
