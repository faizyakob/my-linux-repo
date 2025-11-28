

## Table of contents

- [Introduction](#introduction)
- [Steps to reset root password](#steps-to-reset-root-password)
- [Tips](#tips)
- [Outro](#outro)

## Introduction

In this article, we walkthrough how to reset a root password. For obvious reason, resetting a forgotten root password is required when the access to root is lost.

Note that a user needs to have access to the VM, either physically, via console, or terminal to perform a root password reset. For security purposes, it's very important to only allow VM access to relevant personnel for performing this operation. 

> No users can reset a root password, except the root user itself. Resetting a root password needs to be done before the SystemD and GNOME initialized.

Next section describes the steps in details. 

## Steps to reset root password


<details>
  <summary>Step 1: Reboot VM</summary><br>

  This can either be through GUI menu, or command line. 

  ```
  sudo reboot
  ```
  ```
  sudo systemctl reboot
  ```

</details>

<details>
  <summary>Step 2: Enter GRUB menu during rebooting</summary><br>

  When a reboot is triggered, VM will reinitialize. 
  
  Press "e" during the brief GRUB menu display.

  <img width="1015" height="633" alt="image" src="https://github.com/user-attachments/assets/2f57a4a2-d0d2-4520-badc-8a06f0201081" />

  On some Linux distros, we need to use 'Esc' instead. 

</details>
<details>
  <summary>Step 3: Include required directive for GRUB</summary><br>

  On the now editable GRUB menu, find the line starting with "linux". At the end, include `init=/bin/bash` directive. 
     
  <img width="1019" height="601" alt="image" src="https://github.com/user-attachments/assets/3e14ee34-ff00-4d46-b567-1e79d6fbfb10" />

  This tells the system to run `bash` as the first process on next reboot. 

  Hit CTRL+X to reboot again from GRUB menu. 


</details>
<details>
  <summary>Step 4: Reset root password </summary><br>

  We will be dropped into a `bash` terminal. 
  However, we first need to remount the `/` file system so it is writable. 

  ```
  mount -o remount,rw /
  ```

  At this point we can reset the root password.
  
  ```
  passwd
  ```

  We need to take care of SELinux context as well. Creating `/.autorelabel` file will trigger proper SELinux context relabelling for the whole file system. Bigger file system requires more time. This will be apparent on next reboot. 
  
  ```
  touch /.autorelabel
  ```

  Initiate SystemD as the first process again (PID =1). System will automatically reboot. 
  
  ```
  exec /sbin/init
  ```
  <img width="1278" height="253" alt="image" src="https://github.com/user-attachments/assets/df308a53-9a5d-4abf-99ae-6929a4abab4f" /><br>

  > You can also use `reboot -f`, but this does not guarantee to work on some Linux system

</details>
<details>
  <summary>Step 5: Log in and test new root password </summary><br>

  System should reboot passed the GRUB menu and into GNOME. 

  Log in as a normal user. 
  
  <img width="1276" height="829" alt="image" src="https://github.com/user-attachments/assets/ac8d86f4-51ca-441d-8cd6-8e7269922f00" />

  Once inside, switch to root password. The new password should be enforced now. 

  ```
  su -
  ```

  <img width="1167" height="195" alt="image" src="https://github.com/user-attachments/assets/1b0814d1-1669-45b8-88b4-49514a9b6f86" />

## Tips

We might encounter few challenges when performing root password reset. What I have personnaly encountered were the followings: 

:trollface: GRUB menu does not appear at all

I did some googling and this seems to be a problem when running VMwareFusion and ARM64 architecture. Pressing 'e' won't have any effect and VM will continually boot into GNOME. 


To fix this edit the `/etc/default/grub.cfg` and include the lines:

```
sudo vim /etc/default/grub.cfg
```

+ GRUB_TIMEOUT_STYLE="menu"
+ GRUB_FORCE_HIDDEN_MENU="false"
+ GRUB_TIMEOUT="10"          # This is just to make GRUB menu appears longer

<img width="1434" height="312" alt="image" src="https://github.com/user-attachments/assets/e86fc9e1-0838-46b7-b534-cc4bc590dee9" />


Recreate the `grub.cfg` file. 

```
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```
<img width="1279" height="311" alt="image" src="https://github.com/user-attachments/assets/b9c0072f-c912-489a-8ff5-cc999b2ded06" />

:trollface: GRUB menu still does not appear

If it still does not appear, then we can force the GRUB menu to appear. 

```
sudo grub2-editenv - unset menu_auto_hide
sudo grub2-editenv list
```
Ensure the parameter "_menu_auto_hide_" no longer appears in the result. On next reboot GRUB menu should appear.

<img width="1434" height="117" alt="image" src="https://github.com/user-attachments/assets/391e5f90-4c38-475a-8b4e-d6b6f217e414" />

There could be caused by other cases as well like fastboot. This needs to be checked manually per application basis (VMware, Hyper-V, etc..). However above 2 should fix it in most cases. 


## Outro

Resetting a root password should not be used to hack into the system. 
The procedure assumed individual performing this is authorized to do so, as there is no filtering done by the Linux VM itself. 
Therefore, it is crucial access to the VM is restricted using VPN or firewall. 


