
### Table of contents

- [Introduction](#introduction)
- [What is PAM?](#what-is-pam?)
- [How it works](#how-it-works)
- [Configuring demo service using systemd timer](#configuring-demo-service-using-systemd-timer)
- [Common mistakes and debugging tips](#common-mistakes-and-debugging-tips)
- [Cleanup and key takeaways](#cleanup-and-key-takeaways)

## Linux PAM Authentication: How Modules Are Invoked During sudo and SSH

### Introduction

Linux applications such as `sudo`, `sshd`, `login`, and `su` need a way to authenticate users. Rather than implementing authentication themselves, they rely on the **Pluggable Authentication Modules (PAM)** framework.

This article explains:

+ What PAM is
+ How PAM modules are invoked
+ How sudo authentication works
+ How SSH authentication works
+ A hands-on lab to observe PAM in action

### What is PAM?

PAM provides a standardized authentication framework for Linux applications.

Without PAM:

```
Application
 ├─ Password verification
 ├─ Account checks
 ├─ Session management
 └─ Authorization logic
```

With PAM:

```
Application
      │
      ▼
 PAM Library
      │
      ▼
 PAM Configuration
      │
      ▼
 PAM Modules
```

Applications simply ask PAM:

> "Please authenticate this user."

PAM then determines which modules to execute and in what order.
