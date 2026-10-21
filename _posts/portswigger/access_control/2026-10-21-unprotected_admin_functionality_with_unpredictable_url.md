---
title: "Unprotected Admin Functionality with Unpredictable URL - PortSwigger"
date: 2026-10-21 17:03:12 +0545

description: "Discovering an unprotected administrative endpoint in JavaScript source code and accessing privileged functionality without proper authorization."

categories: [PortSwigger, Access Control]
tags: [burpsuite, javascript, access-control]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uafwuu.png
  alt: "Unprotected admin functionality with unpredictable URL - PortSwigger"

platform: PortSwigger

lab: "Unprotected admin functionality with unpredictable URL"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how an application can expose administrative functionality through an unpredictable URL without properly enforcing access controls. By examining the page source, I discovered the hidden administrative endpoint and accessed it directly without administrator authorization. I then used the admin panel to delete the user `carlos` and complete the lab.

# Question / Lab Objective

The objective was to discover the administrative interface, access it without authorization, and delete the user `carlos`.

![Lab question](/assets/images/writeups/portswigger/access_control/uafwuu/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed a list of products, individual product pages, and a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uafwuu/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

At this stage, I had not identified the administrative interface. Rather than limiting my investigation to the visible pages, I considered whether the application might disclose additional information through its HTML source or client-side scripts.

## 2. Inspecting the page source

The lab provided a hint that encouraged me to investigate the application's source code. I initially suspected that JavaScript might reveal information about hidden functionality, so I opened the page source and examined the available code.

![Inspecting the page source](/assets/images/writeups/portswigger/access_control/uafwuu/3.png)

While inspecting the source, I discovered a reference to the administrative endpoint: `/admin-5<REDACTED>a`

The URL was not displayed as a normal navigation option on the page. However, its presence in the source code revealed where the administrative interface was located.

### Why was inspecting the source useful?

Web applications often contain HTML, JavaScript, and other client-side resources that reveal how the application is structured. Developers may unintentionally leave references to internal pages or administrative endpoints in these resources.

As a penetration tester, this suggests several useful investigation steps:

- Inspect the HTML source for hidden links and endpoint references.
- Examine JavaScript files for routes and API endpoints.
- Review browser network requests to identify additional resources.
- Investigate discovered endpoints only within the authorized testing scope.

However, finding a hidden URL is only the discovery stage. It does not automatically mean the endpoint is vulnerable. The next step is to determine whether the server actually restricts access to the resource.

## 3. Accessing the administrative panel

After identifying the endpoint, I navigated directly to:

`https://<lab-id>.web-security-academy.net/admin-5<REDACTED>a`

The application displayed the administrative interface without requiring me to authenticate as an administrator.

![Admin dashboard and user-management functionality](/assets/images/writeups/portswigger/access_control/uafwuu/4.png)

This confirmed the core vulnerability: the server did not properly restrict access to the administrative functionality.

The endpoint's unpredictable name made it less obvious to discover, but it did not provide reliable access control. Once the URL was identified, the application allowed direct access to the privileged interface.

### Why is an unpredictable URL not enough?

An unpredictable URL may make a resource harder to find, but it does not establish who is allowed to access it.

For example, consider an administrative page at: `/admin-5<REDACTED>a`

If the server assumes that users cannot discover this path, anyone who obtains the URL may be able to access the functionality.

A secure implementation must verify authorization on the server for every protected resource and action. It must not rely on the secrecy of a URL as its primary security mechanism.

This weakness is an example of **security through obscurity**: relying on an obscure location to conceal functionality instead of enforcing proper access controls.

## 4. Deleting the user carlos

After accessing the administrative interface, I examined the available user-management functionality.

The page displayed user accounts and their corresponding delete actions. I located the user `carlos` and used the available functionality to delete the account.

![Deleting the user and confirming lab completion](/assets/images/writeups/portswigger/access_control/uafwuu/5.png)

The lab displayed its completion confirmation, indicating that the required action had been performed successfully.

This demonstrated the impact of the vulnerability: the exposed endpoint allowed an unauthorized user to perform an administrative action.

# Key Takeaways

- **Inspect client-side source code:** HTML and JavaScript can reveal hidden paths, endpoint references, and other information about an application's structure.
- **Use hints to guide investigation:** A lab hint can help identify the next area to examine, but understanding why that step matters is more valuable than memorizing the solution.
- **Distinguish discovery from authorization:** Finding an endpoint does not prove that it is vulnerable. The important question is whether the server properly restricts access.
- **Understand security through obscurity:** An unpredictable URL may reduce casual discovery, but it cannot replace proper authorization.
- **Test direct access:** In an authorized assessment, directly requesting a discovered endpoint can reveal whether the server enforces access restrictions.
- **Verify security impact:** The ability to delete a user demonstrates a more significant issue than merely discovering an administrative URL.

# Security Impact

An administrative interface that lacks proper access controls can allow unauthorized users to perform privileged operations.

Depending on the functionality exposed, the impact could include deleting accounts, modifying user data, changing application settings, or performing other administrative actions.

In this lab, direct access to the hidden administrative endpoint allowed me to delete the user `carlos` without administrator authorization.

In a real-world assessment, the severity would depend on the administrative capabilities exposed, the sensitivity of the affected data, and the practical consequences of unauthorized actions.

# Remediation

To prevent this type of vulnerability:

1. **Enforce server-side authorization:** Verify that each request to a protected endpoint is made by a user with the required permissions.
2. **Do not rely on unpredictable URLs:** Treat endpoint secrecy as an additional layer at most, never as the primary security control.
3. **Protect administrative functionality:** Require appropriate authentication and administrator-level authorization before exposing privileged operations.
4. **Check individual actions:** Authorization must be enforced for sensitive operations, including account deletion, rather than only when displaying an administrative page.
5. **Review client-side resources:** Avoid unnecessarily exposing internal endpoint references, while recognizing that hiding them does not secure the server.
6. **Test direct requests:** Include authorization tests for administrative pages and sensitive API endpoints, including requests made without the required permissions.

# Conclusion

This lab demonstrated how an administrative endpoint can remain accessible to unauthorized users even when its URL is unpredictable. By inspecting the page source, I discovered the hidden endpoint, accessed the administrative interface directly, and deleted the user `carlos`.

The key lesson is that hiding a URL does not secure the functionality behind it. Proper access control must be enforced on the server for every protected resource and administrative action.

*Thanks for reading!*
