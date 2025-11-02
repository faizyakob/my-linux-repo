
=== WORK IN PROGRESS ===
## Table of contents

- [Introduction](#introduction)
- [View SSHD service status](#view-sshd-service-status)
- [View SSHD configuration files](#view-sshd-configuration-files)
- [Modify SSHD parameters](#modify-sshd-parameters)
- [Outro](#outro)

## Introduction

In this article, we walkthrough how to mount an ISO file as a repository. <br>
Normally, depending on the distros, we will register the VM with the respective subscription management service. When a VM is registered, a repository file is created automatically that will point to available online repositories, which then allow users to download and install package via the native package manager like `apt` or `dnf`.




> Note: We use Red Hat as example for this article.

## View SSHD service status


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
