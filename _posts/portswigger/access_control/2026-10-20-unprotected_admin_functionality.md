---
title: "Unprotected admin functionality - PortSwigger"
date: 2026-10-20 13:09:12 +0545

description: "Using Burp Suite ......"

categories: [PortSwigger, Access Control]
tags: [burpsuite]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uaf.png
  alt: "Unprotected admin functionality - PortSwigger"

platform: PortSwigger

lab: "Unprotected admin functionality"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how an exposed Git repository can disclose sensitive information from an application's version control history. Using Burp Suite's content discovery feature, I located a publicly accessible `.git` directory, downloaded the repository data, and examined its commit history. By comparing earlier and later versions of the source code, I recovered the administrator password and used it to access the administrative interface and complete the lab.

# Question / Lab Objective

The objective was to discover the administrator's password through the application's exposed version control history, log in as the administrator, and delete the user `carlos`.

![Lab question](/assets/images/writeups/portswigger/access_control/uaf/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed a list of products, individual product pages, and a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uaf/2.png)

I browsed the application to understand its structure and identify the available entry points for further investigation.

## 2. Accessing the login page

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uaf/3.png)

The lab required me to obtain the administrator's password. Instead of attempting to guess the password, I needed to investigate whether the application exposed information that could help me discover it.

This shifted my focus from the login form to the `application's files`, `directories`, and `deployment configuration`.





















![Admin dashboard and user-management functionality](/assets/images/writeups/portswigger/access_control/uaf/6_1.png)

The dashboard displayed user accounts and the corresponding delete functionality.

I located the user `carlos` and proceeded with the required action for the lab.

## 10. Deleting the user and confirming lab completion

I used the available administrative functionality to delete the user `carlos`.

![Deleting the user and confirming lab completion](/assets/images/writeups/portswigger/access_control/uaf/6_2.png)

The lab displayed its completion confirmation, indicating that I had successfully recovered the administrator password, authenticated as the administrator, and completed the required action.

# Key Takeaways



# Security Impact



# Remediation


# Conclusion



*Thanks for reading!*
