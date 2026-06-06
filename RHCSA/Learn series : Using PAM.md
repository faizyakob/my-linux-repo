
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

### PAM Components

#### PAM Configuration Files

Located under `/etc/pam.d` directory.

Examples:

```
/etc/pam.d/sudo
/etc/pam.d/sshd
/etc/pam.d/login
/etc/pam.d/su
```

Each file defines the PAM stack for a specific service.

#### PAM Modules

Modules are shared libraries typically found in:

```
/usr/lib64/security/
```
or

Examples:
```
pam_unix.so
pam_limits.so
pam_env.so
pam_nologin.so
pam_exec.so
pam_warn.so
```

#### Understanding PAM Rule Syntax

A PAM rule consists of:
```
type    control    module-path    arguments
```

Example:
```
auth required pam_unix.so
```

#### PAM Types

