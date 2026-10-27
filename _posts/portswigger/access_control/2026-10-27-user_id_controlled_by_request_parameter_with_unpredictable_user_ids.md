---
title: "User ID Controlled by Request Parameter, with Unpredictable User IDs - PortSwigger"
date: 2026-10-27 17:05:12 +0545

description: "Exploiting broken access control by modifying a user ID parameter in an authenticated request to retrieve another user's API key."

categories: [PortSwigger, Access Control]
tags: [burpsuite, idor]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uicbrpwuui.png
  alt: "User ID Controlled by Request Parameter, with Unpredictable User IDs - PortSwigger"

platform: PortSwigger

lab: "User ID controlled by request parameter, with unpredictable user IDs"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how broken access control can allow an authenticated user to retrieve another user's private account information. Using Burp Suite, I inspected the account request, discovered Carlos's GUID through the application's blog author links, and modified the `id` parameter to access his account. The server returned Carlos's API key, which I submitted to solve the lab.

# Question / Lab Objective

The objective was to retrieve Carlos's API key by exploiting an access control vulnerability in which the application used a user-controlled request parameter to identify the account whose information should be displayed.

![Lab question](/assets/images/writeups/portswigger/access_control/uicbrpwuui/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed several blog posts and provided a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uicbrpwuui/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

Since the lab focused on access control, I paid particular attention to account-related functionality and how the application identified individual users.

## 2. Accessing the login page

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uicbrpwuui/3.png)

The lab provided credentials for the user `wiener`. I used these credentials to authenticate and access the account page.

Logging in as a legitimate user gave me an authenticated session that I could use to examine how the application handled account information requests.

## 3. Inspecting the account request in Burp Suite

After logging in, I opened Burp Suite and examined the HTTP traffic generated when accessing the account page.

The request contained an `id` parameter:

```http
GET /my-account?id=771a90e7-d52a-4ca2-8fe2-6d391096c1d2 HTTP/2
Host: <lab-id>.web-security-academy.net
```

![Wiener's account request and response in Burp Suite](/assets/images/writeups/portswigger/access_control/uicbrpwuui/4.png)

In the response, I found account information belonging to `wiener`, including his API key.

This observation suggested that the application used the value supplied in the `id` parameter to determine which user's account information to return.

I wanted to determine whether the server verified that the authenticated user was authorized to access the account identified by this parameter.

## 4. Investigating the blog posts to discover Carlos's GUID

Before modifying the request, I explored the application's blog section to determine whether it exposed any information that could help identify other users.

![Carlos's blog post](/assets/images/writeups/portswigger/access_control/uicbrpwuui/5.png)

While browsing the blog posts, I noticed that articles were associated with different authors, including `wiener` and `carlos`.

I opened a blog post written by Carlos and inspected the author link.

![GUID discovered through the blog author link](/assets/images/writeups/portswigger/access_control/uicbrpwuui/6.png)

The author link exposed Carlos's GUID, which appeared to be the same type of identifier used by the `id` parameter in the account request.

This was an important discovery because the GUIDs were not necessarily easy to guess, but the application itself exposed them through publicly accessible blog content.

**Key observation:** An unpredictable identifier can still be discovered if the application reveals it through other accessible resources.

## 5. Modifying the user ID parameter

With Carlos's GUID identified, I returned to the account request captured in Burp Suite.

I changed the `id` parameter from Wiener's GUID to Carlos's GUID while keeping the same authenticated session.

Original request parameter:

```text
id=771a90e7-d52a-4ca2-8fe2-6d391096c1d2
```

Modified request parameter:

```text
id=<carlos-guid>
```

I then sent the modified request to the server.

![Modified request returning Carlos's API key](/assets/images/writeups/portswigger/access_control/uicbrpwuui/7.png)

The application returned account information belonging to Carlos, including his API key, even though I was still authenticated as Wiener.

This confirmed that the server did not adequately enforce authorization when processing the supplied user identifier.

**Key finding:** The application trusted a user-controlled account identifier without correctly verifying whether the authenticated user was permitted to access the requested account.

## 6. Submitting Carlos's API key

After retrieving Carlos's API key, I returned to the lab page and submitted the value in the answer field.

![Submitting Carlos's API key](/assets/images/writeups/portswigger/access_control/uicbrpwuui/8.png)

The API key was the required evidence for completing the challenge.

## 7. Confirming lab completion

After submitting the correct API key, the lab displayed the completion message.

![Lab solved confirmation](/assets/images/writeups/portswigger/access_control/uicbrpwuui/9.png)

This confirmed that I had successfully completed the lab by accessing another user's account information through the vulnerable request parameter.

# Key Takeaways

- **Inspect user-controlled parameters:** Account-related parameters such as `id`, `user`, or `accountId` can reveal how an application selects the resource to return.
- **Understand GUIDs:** Globally unique identifiers are harder to guess than simple sequential IDs, but they are not authorization controls.
- **Look for indirect information disclosure:** Public pages, author links, and other application resources may reveal identifiers used by sensitive endpoints.
- **Test authorization, not just authentication:** Successfully logging in does not mean a user should be allowed to access every account.
- **Use Burp Suite to compare requests:** Changing one parameter while keeping the authenticated session unchanged helps isolate authorization weaknesses.
- **Distinguish discovery from exploitation:** Finding another user's GUID identifies a potential target; receiving that user's private account information confirms the access control failure.

# Security Impact

This vulnerability is an example of **broken access control**, commonly associated with insecure direct object reference (IDOR) vulnerabilities.

An attacker with a valid account could modify the `id` parameter to retrieve information belonging to another user without authorization.

Depending on the information exposed and the application's functionality, the consequences may include:

- Unauthorized access to other users' account information.
- Exposure of API keys or other sensitive account data.
- Potential misuse of exposed credentials if they grant access to additional functionality.
- Privacy violations and possible further compromise, depending on the privileges associated with the disclosed information.

In this lab, the vulnerability allowed me to retrieve Carlos's API key while authenticated as Wiener. The demonstrated impact was unauthorized access to another user's account information.

# Remediation

To prevent this type of vulnerability:

1. **Enforce server-side authorization:** Every account request must verify that the authenticated user is permitted to access the requested account.
2. **Do not rely on unpredictable identifiers:** GUIDs and UUIDs can help make identifiers difficult to guess, but they must never replace authorization checks.
3. **Derive identity from the authenticated session where appropriate:** For a personal account page, use the identity associated with the session rather than blindly trusting a client-supplied user ID.
4. **Apply least privilege:** Users should only be able to access the resources and operations required for their roles.
5. **Review all sensitive endpoints:** Test account pages, API endpoints, and other resources for unauthorized cross-user access.
6. **Protect sensitive responses:** Return only the information the current user is authorized to view, including API keys and other secrets.

# Conclusion

This lab demonstrated that unpredictable user IDs do not prevent unauthorized access when an application fails to enforce proper access controls. By inspecting the account request in Burp Suite, discovering Carlos's GUID through a blog author link, and replacing Wiener's GUID with Carlos's, I retrieved information that should have remained private.

The main lesson is that **identification and authorization are two different things**. A GUID identifies a resource, but the server must still verify that the requesting user has permission to access it.

*Thanks for reading!*
