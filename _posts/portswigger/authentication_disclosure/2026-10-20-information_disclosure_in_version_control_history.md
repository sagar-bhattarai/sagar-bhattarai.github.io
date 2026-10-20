---
title: "Information Disclosure in Version Control History - PortSwigger"
date: 2026-10-20 09:09:12 +0545

description: "Using Burp Suite and Git history analysis to discover an exposed .git directory and recover an administrator password from an earlier commit."

categories: [PortSwigger, Information Disclosure]
tags: [burpsuite, git]

image:
  path: /assets/img/posts_thumbnails/portswigger/idivch.png
  alt: "Information Disclosure in Version Control History - PortSwigger"

platform: PortSwigger

lab: "Information disclosure in version control history"
type: "Lab Write-up"
status: complete
---

# Overview

This lab demonstrates how an exposed Git repository can disclose sensitive information from an application's version control history. Using Burp Suite's content discovery feature, I located a publicly accessible `.git` directory, downloaded the repository data, and examined its commit history. By comparing earlier and later versions of the source code, I recovered the administrator password and used it to access the administrative interface and complete the lab.

# Question / Lab Objective

The objective was to discover the administrator's password through the application's exposed version control history, log in as the administrator, and delete the user `carlos`.

![Lab question](/assets/images/writeups/portswigger/authentication_disclosure/idivch/1.png)

# Walkthrough

## 1. Exploring the application

After launching the lab, I accessed the target application and explored its available functionality. The website displayed a list of products, individual product pages, and a **My Account** option.

![Initial application page](/assets/images/writeups/portswigger/authentication_disclosure/idivch/2.png)

I browsed the application to understand its structure and identify the available entry points for further investigation.

## 2. Accessing the login page

I clicked **My Account**, which redirected me to the login page.

![Login form](/assets/images/writeups/portswigger/authentication_disclosure/idivch/3.png)

The lab required me to obtain the administrator's password. Instead of attempting to guess the password, I needed to investigate whether the application exposed information that could help me discover it.

This shifted my focus from the login form to the `application's files`, `directories`, and `deployment configuration`.

## 3. Discovering the exposed `.git` directory

To identify additional resources, I reused the content-discovery technique from my previous write-up, [Source Code Disclosure via Backup Files](https://sagar-bhattarai.github.io/portswigger/information%20disclosure/2026/10/19/source_code_disclosure_via_backup_files.html).

Using Burp Suite's **Discover content** feature, I checked the application for directories and files that were not immediately visible through the website's interface.

During this process, I discovered several endpoints. One particularly interesting result was: `/.git/`

![Discovered endpoints, including .git](/assets/images/writeups/portswigger/authentication_disclosure/idivch/endpoints.png)

### Why was `.git` interesting?

Git is a version control system used to track changes to source code. A repository normally contains a hidden `.git` directory that stores information such as commit objects, references, and repository metadata.

If this directory is accessible through a web server, an attacker may be able to reconstruct the repository and inspect its history.

This is important because **removing a password from the latest version of a file does not necessarily remove it from earlier Git commits**.

I decided to investigate the exposed directory further.

## 4. Exploring the `.git` directory and commit history

I opened the discovered URL in the browser: `https://<lab-id>.web-security-academy.net/.git/`

![Accessing the exposed .git directory](/assets/images/writeups/portswigger/authentication_disclosure/idivch/4.png)

I then explored the available repository resources, including the Git log information at: `/.git/logs`

![Viewing .git/logs/HEAD](/assets/images/writeups/portswigger/authentication_disclosure/idivch/4_1.png)

While examining the log, I noticed commit information and messages indicating that changes had been made to the project.

![Commit history information](/assets/images/writeups/portswigger/authentication_disclosure/idivch/4_2.png)

This gave me a useful clue: the repository had a history of changes that might reveal information no longer present in the current version of the application.

Rather than examining every file manually through the browser, I decided to download the exposed repository data and investigate it using Git's command-line tools.

## 5. Downloading the repository data

I used `wget` to download the publicly accessible Git directory within the lab environment.

```bash
wget -r https://<lab-id>.web-security-academy.net/.git/
```

![Downloading the exposed Git repository](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5.png)

The command uses recursive downloading to retrieve accessible resources beneath the specified URL. The exact files retrieved depend on the server's configuration and how it exposes the directory.

After the download, I inspected the resulting directory structure.

### Understanding the hidden `.git` directory

The `.git` directory is hidden by default on Linux because its name begins with a dot. To display hidden files and directories, I used:

![Viewing the downloaded hidden .git directory](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_1.png)

```bash
ls -la
```

I then examined the downloaded repository data and its internal files and directories.

![Inspecting the downloaded Git repository data](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_2.png)

**Important:** Git commands should normally be run from the working-tree directory that contains `.git`, rather than from inside `.git` itself. If only the repository metadata was downloaded, the commands can instead be run with an explicit Git directory, provided the required repository objects were retrieved successfully.

## 6. Inspecting branches and commit history

Once I had access to the repository data, I wanted to understand which branches and commits were available.

I started with:

```bash
git branch
```

![Inspecting Git branches](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_3.png)

The output showed the available branch information. In this case, I observed a branch named `master`.

I then examined the commit history:

```bash
git log
```

![Viewing the Git commit history](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_4.png)

The log showed the recorded commits, including their identifiers, authors, dates, and messages.

I had already seen a clue in the exposed log information, so I wanted to investigate how the source code had changed between commits. My next step was to inspect the actual changes rather than relying on the commit messages alone.

## 7. Recovering the administrator password from earlier commits

Git provides several ways to inspect changes between versions. I used two approaches to investigate the repository history.

### Method 1: Comparing two commits with `git diff`

The general syntax is:

```bash
git diff <older-commit-id> <newer-commit-id>
```

![Comparing changes between Git commits](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_5.png)

This command compares the contents represented by two commits and displays their differences.

For example, a line prefixed with `-` was present in the older version but is absent from the newer version. A line prefixed with `+` was added in the newer version.

By comparing the relevant commits, I could inspect whether sensitive information had been removed or modified.

### Method 2: Examining changes with `git log -p`

I also used:

```bash
git log -p
```

![Inspecting commit history and patches with git log -p](/assets/images/writeups/portswigger/authentication_disclosure/idivch/5_6.png)

Unlike `git log` alone, `git log -p` displays the patches associated with commits. These patches show the changes introduced by each commit.

This was particularly useful because a developer might remove a hard-coded password from the latest version of a file while leaving the original value recoverable from an earlier commit.

By reviewing the historical changes, I located the administrator credential required by the lab.

### Why this technique works

Consider a simplified example:

- **Earlier commit:** A configuration file contains a hard-coded administrator password.
- **Later commit:** The developer removes the password from that file.
- **Current application:** The password no longer appears in the latest version.
- **Git history:** The earlier commit still records the original change.

Unless the sensitive data has been properly removed from the repository's history, someone who obtains the repository may still be able to recover it.

This explains why inspecting the current source code alone may not reveal every exposed secret.

## 8. Logging in with the recovered credentials

After locating the administrator password in the repository history, I returned to the application's login page and used the recovered credentials.

![Logging in with the recovered administrator credentials](/assets/images/writeups/portswigger/authentication_disclosure/idivch/6.png)

The credentials allowed me to log in as the administrator.

This confirmed that the information disclosed through the Git history was useful beyond simply revealing internal source code: it exposed a credential that could be used to access a privileged account.

## 9. Accessing the admin dashboard

After logging in, I accessed the administrative interface and observed the available user-management functionality.

![Admin dashboard and user-management functionality](/assets/images/writeups/portswigger/authentication_disclosure/idivch/6_1.png)

The dashboard displayed user accounts and the corresponding delete functionality.

I located the user `carlos` and proceeded with the required action for the lab.

## 10. Deleting the user and confirming lab completion

I used the available administrative functionality to delete the user `carlos`.

![Deleting the user and confirming lab completion](/assets/images/writeups/portswigger/authentication_disclosure/idivch/6_2.png)

The lab displayed its completion confirmation, indicating that I had successfully recovered the administrator password, authenticated as the administrator, and completed the required action.

# Key Takeaways

- **Inspect hidden resources:** Application functionality does not always reveal every file or directory exposed by the web server.
- **Use content discovery:** Burp Suite's Discover content feature can identify resources that are not directly linked from the application's pages.
- **Recognize exposed Git repositories:** A publicly accessible `.git` directory can disclose repository metadata and source code history.
- **Understand Git commits:** Each commit records a snapshot of the project, allowing earlier versions of tracked files to be examined.
- **Use `git log` and `git log -p`:** The first displays commit history, while the second also shows the patches associated with commits.
- **Use `git diff` to compare versions:** Comparing commits can reveal code or credentials that were removed in a later revision.
- **Remember that deleted secrets may remain recoverable:** Removing a credential from the latest file does not automatically remove it from the repository's history.
- **Verify the impact:** Recovering a credential is a significant finding, but its real-world impact depends on whether it remains valid and what permissions it provides.

# Security Impact

An exposed version control repository can disclose source code, internal implementation details, configuration data, and historical changes. If developers have committed passwords, API keys, or other secrets, those values may remain recoverable even after being removed from the current code.

In this lab, the exposed Git history revealed the administrator password. The credential enabled administrative login, which allowed the user `carlos` to be deleted.

In a real application, similar exposure could lead to unauthorized account access, data disclosure, data modification, or other privileged actions, depending on the validity and permissions associated with the exposed credentials.

# Remediation

To prevent information disclosure through version control history:

1. **Prevent public access to `.git`:** Configure the web server to deny requests for Git metadata and other development artifacts.
2. **Exclude repository metadata from deployments:** Deploy only the application files required at runtime rather than the entire development repository.
3. **Never commit secrets:** Store credentials in an appropriate secrets manager or securely configured environment instead of source code.
4. **Remove exposed credentials from active use:** If a secret has been disclosed, revoke or rotate it promptly.
5. **Clean repository history when necessary:** If sensitive information has been committed, follow a suitable history-rewriting procedure and coordinate the change with repository users. Rewriting history alone does not invalidate a credential, so rotation is essential.
6. **Audit deployment artifacts:** Check production deployments for exposed `.git` directories, backup files, configuration files, and other unnecessary resources.
7. **Include exposure checks in deployment pipelines:** Automate checks to detect accidentally published repository metadata and secrets before deployment.

# Conclusion

This lab demonstrated how an exposed Git repository can reveal credentials that are no longer present in the application's latest source code. Using Burp Suite's content discovery feature, I located the `/.git/` directory, examined its log information, downloaded the repository data, and inspected the commit history with Git.

By using `git diff` and `git log -p`, I recovered the administrator password and used it to access the admin dashboard and delete the user `carlos`.

The main lesson is that version control history can retain sensitive information long after it has been removed from the current source code. Preventing public access to repository metadata and avoiding the storage of secrets in source code are essential practices for securing production applications.

*Thanks for reading!*
