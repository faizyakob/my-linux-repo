

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

## Outro

As we are using local repositories, Red Hat will keep displaying the message to encourage registering the VM to entitlement server. 
This is expected and can safely be ignored. 

Note that the limitation of using local repositories is that it's limited to packages available in the ISO image file. If it does not contain packages that we want, we still need to use online repositories for that. 


