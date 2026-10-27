---
title: "User ID Controlled by Request Parameter - PortSwigger"
date: 2026-10-27 14:30:12 +0545

description: "Testing for broken access control by modifying a user ID parameter and retrieving another user's API key."

categories: [PortSwigger, Access Control]
tags: [burpsuite, idor]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uicbrp.png
  alt: "User ID Controlled by Request Parameter - PortSwigger"

platform: PortSwigger

lab: "User ID controlled by request parameter"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how improper access control can allow one authenticated user to access another user's account information. By inspecting a request in Burp Suite and changing the `id` parameter from `wiener` to `carlos`, I was able to retrieve Carlos's API key and submit it to complete the lab.

# Question / Lab Objective

The objective was to access Carlos's account information and obtain his API key by exploiting the application's user-controlled request parameter.

![Lab question](/assets/images/writeups/portswigger/access_control/uicbrp/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed a list of products, individual product pages, and a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uicbrp/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

## 2. Accessing the login page

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uicbrp/3.png)

The lab instructions provided credentials for the user `wiener`. I used these credentials to authenticate and access the account page.

This gave me a legitimate authenticated session to use while investigating how the application handled requests for account information.

## 3. Inspecting the request and response in Burp Suite

After logging in, I opened Burp Suite and examined the HTTP traffic generated when accessing the account page.

The request included the following URL parameter: `GET /my-account?id=wiener`

![Wiener's account request and response in Burp Suite](/assets/images/writeups/portswigger/access_control/uicbrp/4.png)

In the response, I found account information belonging to `wiener`, including an API key.

This observation was important because the application appeared to use the `id` parameter to determine which user's account information should be returned.

I decided to investigate whether the application properly verified that the authenticated user was authorized to access the account identified by this parameter.

## 4. Modifying the user ID parameter

I considered whether the application associated account access exclusively with the authenticated session or whether it trusted the user identifier supplied in the request.

To test this, I changed the parameter from: `id=wiener` to: `id=carlos`

I then submitted the modified request using the same authenticated session.

![Modified request returning Carlos's API key](/assets/images/writeups/portswigger/access_control/uicbrp/5.png)

The application returned account information for `carlos`, including his API key, even though I had authenticated as `wiener`.

This confirmed that the application did not adequately enforce authorization when processing the supplied user ID.

**Key finding:** The server trusted a user-controlled account identifier without properly checking whether the authenticated user was permitted to access the requested account.

## 5. Submitting Carlos's API key

After retrieving Carlos's API key, I returned to the lab page and submitted the key in the answer field.

![Submitting Carlos's API key](/assets/images/writeups/portswigger/access_control/uicbrp/6.png)

The lab accepted the API key and displayed the completion message, confirming that I had successfully completed the challenge.

# Key Takeaways

- **Inspect account-related parameters:** Parameters such as `id`, `user`, and `accountId` can determine which user's information an application retrieves.
- **Understand the difference between authentication and authorization:** Authentication establishes who the user is; authorization determines what that user is allowed to access.
- **Test for IDOR:** Insecure Direct Object Reference (IDOR) vulnerabilities can occur when an application exposes an object identifier and fails to verify access permissions.
- **Use Burp Suite to compare requests:** Inspecting HTTP requests and responses makes it easier to identify whether changing a parameter affects the data returned by the server.
- **Do not assume a session provides sufficient protection:** An authenticated session does not automatically authorize access to every user's account.
- **Verify the impact:** Receiving another user's sensitive information provides evidence of unauthorized access, even when no password or session token has been stolen.

# Security Impact

This vulnerability is an example of broken access control and can be classified as an IDOR-style vulnerability when the application exposes a direct object reference without enforcing appropriate authorization.

An attacker with a valid account could potentially modify the user ID parameter to access another user's private information. Depending on the application's functionality and the data exposed, the impact could include unauthorized access to API keys, personal information, account details, or other sensitive resources.

In this lab, changing the `id` parameter allowed the authenticated `wiener` user to retrieve Carlos's API key. In a real-world application, the severity would depend on the sensitivity of the exposed data, the privileges associated with it, and whether the issue affected additional accounts or operations.

# Remediation

To prevent this type of vulnerability:

1. **Enforce server-side authorization:** Verify that the authenticated user is permitted to access the requested account or resource on every relevant request.
2. **Do not trust user-supplied identifiers:** Treat parameters such as `id` and `accountId` as untrusted input. A valid identifier does not prove that the requester has permission to access the corresponding resource.
3. **Apply least privilege:** Ensure users can access only the information and functionality required for their roles.
4. **Protect sensitive account data:** Restrict access to API keys and other credentials, and avoid exposing them unnecessarily in account pages or responses.
5. **Test access control systematically:** Include tests that attempt to access another user's resources using different identifiers and authenticated sessions.
6. **Monitor suspicious access patterns:** Log and investigate repeated attempts to access accounts or resources belonging to other users.

# Conclusion

This lab demonstrated how a simple user ID parameter can become an access control weakness when the server fails to verify resource ownership. By logging in as `wiener`, inspecting the account request in Burp Suite, and changing `id=wiener` to `id=carlos`, I retrieved Carlos's API key and successfully completed the lab.

The key lesson is that applications must enforce authorization on the server for every protected resource. User-controlled identifiers should never be treated as proof that a requester is entitled to access the associated account.

*Thanks for reading!*
