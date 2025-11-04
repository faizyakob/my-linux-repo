
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

+ SSH is a protocol used to access a server from a client. It uses TCP and port `22` by default.

  <img width="723" height="251" alt="image" src="https://github.com/user-attachments/assets/42ee3430-b4bc-476e-8562-cf218e842668" />


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

For some distros, SSH is already installed by default when the OS is installed. To check if SSH is installed and enabled, run following commands:

<details>
<summary>Check if SSHD is running</summary><br>
  
The SSHD daemon must be running to provide SSH service.<br>
  
```
sudo systemctl status sshd
```
<img width="842" height="210" alt="image" src="https://github.com/user-attachments/assets/67d12953-b565-4c7b-8973-120fc7c52d67" />

If the status is not **active (running)**, enable & start the SSHD daemon.

If the output is "_Unit sshd.service could not be found._", then SSH package is not installed. Refer how to install in "Introduction" section above, then enable & start the SSH daemon.

</details>

<details>
<summary>Enable & start SSHD daemon</summary><br>
  
```
sudo systemctl enable sshd
sudo systemctl start sshd
```

Check again the status to ensure it is now running.

</details>




## View SSHD configuration files

SSH makes use of many configuration files, depending on its usage context: <br>

  🖥️ Server-side (SSH Daemon) files <br>
  💻 Client-side (User SSH configuration and cache) <br>
  🔑 Authentication-related files (on both ends) <br>
  🧰 Supporting files <br>

  ### Server-side (SSH Daemon) files

  | File | Usage | 
  |------|-------|
  | `/etc/ssh/sshd_config`| Global config for the SSH **server daemon**. Controls authentication methods, allowed users, port, etc. | 
  |`/etc/ssh/ssh_host_*_key` and `/etc/ssh/ssh_host_*_key.pub`|Server’s **private and public host keys**. Used to identify the server to clients (e.g. ssh_host_rsa_key, ssh_host_ecdsa_key).|
  | `/etc/nologin` (optional)| If present, prevents non-root logins (useful for maintenance). | 
  |`/var/log/auth.log` or `/var/log/secure`|Where SSH authentication events are logged (depends on distro).|

  `/etc/ssh/sshd_config` is the most common file that is modified, as it affects behavior when client is connecting to this host. Some of important parameters are _PermitRootLogin_, _Port_, _PasswordAuthentication_ and _PubkeyAuthentication_. In [Modify SSHD parameters](#modify-sshd-parameters), we demonstrate on how to modify some of these parameters.

## Modify SSHD parameters
## Outro

As we are using local repositories, Red Hat will keep displaying the message to encourage registering the VM to entitlement server. 
This is expected and can safely be ignored. 

Note that the limitation of using local repositories is that it's limited to packages available in the ISO image file. If it does not contain packages that we want, we still need to use online repositories for that. 
