
=== WORK IN PROGRESS ===
## Table of contents

- [Introduction](#introduction)
- [View SSHD service status](#view-sshd-service-status)
- [View SSHD configuration files](#view-sshd-configuration-files)
- [Modify SSHD parameters](#modify-sshd-parameters)
- [Outro](#outro)

## Introduction

In this article, we discuss on high level what SSH is. Objective is not to know the nitty-gritty details, but enough to sufficiently provide overwiew of SSH.<br>
SSH is not tested in details for both RHCSA and LFCS exams, but it is essential tools for lots of troubleshooting scenarios. An engineer have to at the very least, know how to operate SSH, and figure out what could be the issue if SSH is somehow not working. <br>

+ SSH is a protocol used to access a server from a client. It uses port `22` by default.

+ In Linux, SSH is implemented using the OpenSSH suite, which provides both the client (to connect to other machines) and the server (to accept SSH connections).

  - Client program: `ssh` — lets you connect to another system.

  - Server program: `sshd` — runs daemon on the system to accept incoming SSH connections.

+ OpenSSH is installed using the native package manager.

  - RHEL/Fedora/CentOS: ```sudo dnf install openssh-server openssh-clients```
  - Ubuntu/Debian: ```sudo apt install openssh-server```

+ To connect to another system using SSH, the most basic command is:
  ```
  ssh <username>@<hostname_or_ip>
  ```
  Note:
    - If ```<username>``` is not specified, the current user is assumed.
    - We can increase the verbosity when connecting by providing `-v`, so the process will output more logs. This is useful in troubleshooting connection. The `-vvv` is the most verbose option. 
  

## View SSHD service status

For some distros, SSH is already installed by default when the OS is installed. To check if SSH is isntalled and enabled, run following commands:

<details>

Insert details here.

</details>



```
dnf install -y nmap
```


## Modify SSHD parameters

## View SSHD configuration files

## Outro

As we are using local repositories, Red Hat will keep displaying the message to encourage registering the VM to entitlement server. 
This is expected and can safely be ignored. 

Note that the limitation of using local repositories is that it's limited to packages available in the ISO image file. If it does not contain packages that we want, we still need to use online repositories for that. 
