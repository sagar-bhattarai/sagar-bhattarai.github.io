---
title: "Unprotected Admin Functionality - PortSwigger"
date: 2026-10-20 13:09:12 +0545

description: "Discovering an unprotected administrative endpoint through robots.txt and exploiting missing access controls using direct URL access."

categories: [PortSwigger, Access Control]
tags: [burpsuite]

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

At this stage, I had not identified the administrative interface. Instead of limiting my investigation to the visible pages, I decided to check whether the application exposed additional information about its available paths.

## 2. Inspecting the login page and robots.txt

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uaf/3.png)

Initially, I considered whether I needed administrator credentials. However, before attempting to guess a password, I decided to investigate whether the application revealed any information about its administrative functionality.

I accessed the following path on the lab's domain:

`https://<lab-id>.web-security-academy.net/robots.txt`

The `robots.txt` file contained a reference to the administrative endpoint: `/administrator-panel`.

![robots.txt revealing the administrator-panel endpoint]
