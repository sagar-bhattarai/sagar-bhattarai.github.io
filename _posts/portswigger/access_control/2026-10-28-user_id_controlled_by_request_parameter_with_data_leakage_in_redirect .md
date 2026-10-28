---
title: "User ID Controlled by Request Parameter with Data Leakage in Redirect - PortSwigger"
date: 2026-10-28 08:08:08 +0545

description: "Exploiting broken access control by modifying a user ID parameter and retrieving another user's API key from a redirect response using Burp Suite."

categories: [PortSwigger, Access Control]
tags: [burpsuite, idor]

image:
  path: /assets/img/posts_thumbnails/portswigger/access_control/uicbrpwdlir.png
  alt: "User ID Controlled by Request Parameter with Data Leakage in Redirect - PortSwigger"

platform: PortSwigger

lab: "User ID controlled by request parameter with data leakage in redirect"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how broken access control can expose another user's sensitive information when an application relies on a user-controlled request parameter to identify an account. Using Burp Suite, I modified the `id` parameter from `wiener` to `carlos` and discovered that the server returned Carlos's API key in the response body, despite issuing an HTTP 302 redirect to the login page. I submitted the disclosed API key to complete the lab.

# Question / Lab Objective

The objective was to retrieve Carlos's API key by exploiting an access control vulnerability in which the application used a user-controlled request parameter to identify the account whose information should be displayed.

![Lab question](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed several products and provided a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/2.png)

I browsed the application to understand its structure and identify potential entry points for further investigation.

Since the lab focused on access control and information leakage in a redirect response, I paid particular attention to account-related functionality and how the application identified individual users.

## 2. Accessing the login page

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/3.png)

The lab provided credentials for the user `wiener`. I used these credentials to authenticate and access the account page.

After logging in, I was redirected to Wiener's account dashboard.

![Wiener's user dashboard](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/4.png)

I now had a legitimate authenticated session that I could use to inspect the application's account-related requests and responses.

![Authenticated session for the legitimate user](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/5.png)

This step was important because it allowed me to investigate the vulnerability using a normal user's session rather than attempting to bypass the login process.

## 3. Inspecting and modifying the user ID parameter

I examined the account-related request and noticed that the application used an `id` parameter to identify the account whose information was being requested.

I decided to test whether changing this parameter would allow me to retrieve information belonging to another user.

The original and modified parameter values were:

- Original: `id=wiener`
- Modified: `id=carlos`

I changed the identifier from `wiener` to `carlos` while keeping the same authenticated session.

I then sent the modified request to the server and inspected the response in Burp Suite.

![Modified request returning Carlos's API key with an HTTP 302 redirect](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/6.png)

The server responded with an HTTP `302 Found` status and indicated a redirect to `/login`. However, the response body also contained account information belonging to Carlos, including his API key.

This was the critical observation: **the redirect did not prevent the sensitive information from being included in the HTTP response.**

## 4. Examining the redirect response

I inspected the response more closely to understand what the server had returned.

![Response body containing Carlos's API key despite the redirect](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/6_1.png)

Although the response instructed the browser to navigate to `/login`, the server had already sent Carlos's account information in the response body.

An HTTP redirect is not, by itself, an access control mechanism. A client that receives the response can inspect its contents before following the redirect. Tools such as Burp Suite make it possible to examine the raw HTTP response directly.

The issue was therefore not simply that the application redirected to the login page. The more serious problem was that sensitive information had been disclosed to a user who should not have been authorized to access it.

**Key finding:** The application returned Carlos's API key in response to a request made using Wiener's authenticated session, without adequately enforcing authorization for the requested account.

## 5. Understanding the vulnerability

This behavior demonstrates broken access control involving a user-controlled object identifier.

The application accepted the supplied `id` value to determine which user's information to retrieve but failed to enforce the necessary authorization checks before returning that information.

This resembles an **Insecure Direct Object Reference (IDOR)** vulnerability, a type of broken access control in which an application exposes an object through a reference that a user can manipulate without the server correctly verifying access permissions.

In this case, changing the identifier was enough to expose another user's API key.

The redirect response added an important detail: even though the application attempted to redirect the browser to the login page, the sensitive data was already present in the response body.

## 6. Submitting Carlos's API key

After retrieving Carlos's API key from the response, I returned to the lab page and submitted the value in the answer field.

![Submitting Carlos's API key](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/7.png)

The application accepted the API key, confirming that I had retrieved the information required to solve the challenge.

## 7. Confirming lab completion

After submitting the correct API key, the lab displayed the completion message.

![Lab solved confirmation](/assets/images/writeups/portswigger/access_control/uicbrpwdlir/8.png)

This confirmed that I had successfully completed the lab by exploiting the application's failure to properly protect another user's account information.

# Key Takeaways

- **Inspect user-controlled parameters:** Values such as `id`, `user`, or `accountId` may determine which account or object an application retrieves.
- **Test authorization, not just authentication:** Being logged in does not mean a user should be able to access every account.
- **Understand IDOR vulnerabilities:** Changing an object identifier can expose another user's data when the server fails to enforce object-level authorization.
- **Inspect the complete HTTP response:** Status codes and redirect locations do not tell the whole story. The response body may contain sensitive information even when the browser is redirected elsewhere.
- **Use Burp Suite to investigate requests and responses:** Intercepting or repeating requests allows you to compare how the application behaves when user-controlled parameters change.
- **Distinguish a redirect from access control:** An HTTP `302` response does not undo data that the server has already disclosed.
- **Treat API keys as sensitive credentials:** API keys should be protected from unauthorized disclosure and given only the permissions they require.

# Security Impact

The vulnerability allows an authenticated user to retrieve sensitive information belonging to another user by manipulating a request parameter.

In this lab, the exposed information was Carlos's API key. Depending on the permissions associated with an API key, unauthorized disclosure could allow an attacker to access protected API functionality or perform actions on behalf of the key's owner.

The HTTP redirect does not mitigate the disclosure because the sensitive data is already included in the response body. A client can read that response without following the redirect.

In a real-world application, the severity would depend on the sensitivity of the exposed data, the permissions associated with the affected account, and the potential actions available through the disclosed credential.

# Remediation

To prevent this type of vulnerability:

1. **Enforce server-side authorization:** Verify that the authenticated user is permitted to access the requested account before retrieving or returning its data.
2. **Validate object-level permissions:** Do not trust user-supplied identifiers as proof of authorization. Every request involving a protected object must be checked against the user's permissions.
3. **Avoid returning sensitive data before authorization:** Ensure that unauthorized requests never receive private account information in the response body, headers, or other response components.
4. **Handle redirects correctly:** Redirecting an unauthorized request to a login or error page must not replace proper access control checks.
5. **Protect API keys:** Store credentials securely, avoid exposing them in responses, and rotate or revoke compromised keys where necessary.
6. **Test access controls systematically:** Verify that users cannot access other users' data by changing identifiers in URLs, query parameters, request bodies, or headers.

# Conclusion

This lab demonstrated how a user-controlled request parameter can lead to broken access control and the disclosure of another user's sensitive information. By authenticating as Wiener, changing the `id` parameter to `carlos`, and inspecting the HTTP response in Burp Suite, I retrieved Carlos's API key despite the server returning an HTTP 302 redirect to `/login`.

The main lesson is that authorization must be enforced on the server before sensitive information is returned. Redirecting a browser does not protect data that has already been disclosed in the response.

*Thanks for reading!*
