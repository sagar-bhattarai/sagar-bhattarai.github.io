---
title: "Unprotected Admin Functionality - PortSwigger"
date: 2026-10-20 13:09:12 +0545

description: "Discovering an unprotected administrative endpoint through robots.txt and exploiting missing access controls using direct URL access."

categories: [PortSwigger, Access Control]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uaf.png
  alt: "Unprotected Admin Functionality - PortSwigger"

platform: PortSwigger

lab: "Unprotected admin functionality"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how inadequate access controls can expose administrative functionality to unauthorized users. By inspecting the application's `robots.txt` file, I discovered the `/administrator-panel` endpoint and accessed it directly without administrator authentication. I then used the exposed functionality to delete the user `carlos` and complete the lab.

# Question / Lab Objective

The objective was to discover the unprotected administrative interface, access it without authorization, and delete the user `carlos`.

![Lab question](/assets/images/writeups/portswigger/access_control/uaf/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed a list of products, individual product pages, and a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uaf/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

At this stage, I had not identified the administrative interface. Instead of limiting my investigation to the visible pages, I decided to check whether the application exposed any additional information about its available paths.

## 2. Inspecting the login page and robots.txt

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uaf/3.png)

Initially, I considered whether I needed to obtain administrator credentials. However, before attempting to guess a password, I decided to investigate whether the application revealed any information about its administrative functionality.

I accessed the following path on the lab's domain:

`https://<lab-id>.web-security-academy.net/robots.txt`

The `robots.txt` file contained a reference to the administrative endpoint: `/administrator-panel`

![robots.txt revealing the administrator-panel endpoint](/assets/images/writeups/portswigger/access_control/uaf/4.png)

### Why check robots.txt?

The `robots.txt` file provides instructions to web crawlers about which paths they should or should not crawl. Developers sometimes include administrative or internal paths in this file to discourage indexing.

However, **robots.txt is not an access-control mechanism**. It does not prevent users from visiting the listed paths directly.

Finding an administrative endpoint in `robots.txt` does not automatically mean that the endpoint is vulnerable. The next step is to check whether the application actually enforces authorization when someone attempts to access it.

This was the key clue that guided my investigation.

## 3. Accessing the administrator panel

After identifying the endpoint, I navigated directly to:

`https://<lab-id>.web-security-academy.net/administrator-panel`

The application displayed the administrative interface without requiring me to authenticate as an administrator.

![Admin dashboard and user-management functionality](/assets/images/writeups/portswigger/access_control/uaf/5.png)

This confirmed the core vulnerability: the application exposed privileged functionality without properly restricting access.

The issue was not simply that the endpoint's location was disclosed. The more serious problem was that the server failed to enforce authorization when the endpoint was accessed.

## 4. Deleting the user carlos

The administrative interface displayed user accounts and the corresponding delete functionality.

I located the user `carlos` and used the available administrative action to delete the account.

![Deleting the user and confirming lab completion](/assets/images/writeups/portswigger/access_control/uaf/6.png)

The lab displayed its completion confirmation, indicating that the required action had been performed successfully.

# Key Takeaways

- **Check common discovery files:** Resources such as `robots.txt` can reveal application paths that are not linked from the main interface.
- **Do not confuse obscurity with security:** Hiding an administrative URL does not protect it from unauthorized access.
- **Test authorization, not just authentication:** A sensitive endpoint must verify that the requester has the appropriate permissions, regardless of whether the URL is known.
- **Use direct URL testing:** Manually accessing discovered paths can reveal missing access controls.
- **Verify the impact:** Finding an administrative endpoint is only the first step. Demonstrating unauthorized access to a privileged function establishes the security significance of the issue.

# Security Impact

An administrative interface that lacks proper access controls may allow unauthorized users to perform privileged operations without administrator authentication.

Depending on the functionality exposed, the impact could include deleting user accounts, modifying application settings, accessing sensitive information, or performing other administrative actions.

In this lab, the exposed administrative panel allowed me to delete the user `carlos`. In a real-world application, the severity would depend on the privileges available and the potential consequences of unauthorized administrative actions.

# Remediation

To prevent unprotected administrative functionality:

1. **Enforce server-side authorization:** Every administrative endpoint must verify that the requester has the required permissions before performing an operation.
2. **Require appropriate authentication:** Administrative interfaces should be available only to authenticated users with authorized administrative privileges.
3. **Do not rely on robots.txt for security:** Treat this file as crawler guidance, not as a mechanism for restricting access.
4. **Apply least privilege:** Ensure users can perform only the operations required by their roles.
5. **Test direct access to sensitive endpoints:** Verify that unauthorized users cannot access administrative pages or execute privileged actions by entering URLs directly.
6. **Monitor administrative activity:** Log and investigate suspicious access attempts and unauthorized administrative operations.

# Conclusion

This lab demonstrated how a simple configuration mistake can expose privileged application functionality. By inspecting `robots.txt`, I discovered the `/administrator-panel` endpoint and accessed it directly. Because the application did not properly enforce authorization, I was able to reach the admin interface and delete the user `carlos`.

The main lesson is that administrative functionality must be protected by server-side authentication and authorization checks. Concealing a URL is never a substitute for enforcing access controls.

*Thanks for reading!*
