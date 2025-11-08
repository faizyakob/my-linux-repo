
## Table of contents

- [Introduction](#introduction)
- [Steps to use ISO as repositories](#steps-to-use-iso-as-repositories)
- [Outro](#outro)

## Introduction

In this article, we walkthrough how to mount an ISO file as a repository. <br>
Normally, depending on the distros, we will register the VM with the respective subscription management service. When a VM is registered, a repository file is created automatically that will point to available online repositories, which then allow users to download and install package via the native package manager like `apt` or `dnf`.

For Red Hat Linux, we use `subscription-manager` tool to register the VM.

As root user, run `subscription-manager status` to check if VM is registered. <br>

<img width="5100" height="450" alt="image" src="https://github.com/user-attachments/assets/bd6c3cc2-4f85-4b29-a2b5-648ebfc3af36" />

However, there are cicumstances where we do not want to connect VM with online repositories, but still want to download and install packages. <br>

In this case, we can use the VM's ISO image and mount it to a directory. The ISO image file already contains a huge number of packages, which will serve as local repostiories nicely. <br>

(Ref: [VMware 00 Create VM from ISO image file ](<../VMware 00 Create VM from ISO image file.md>)) 

> Note: We use Red Hat as example for this article.

## Steps to use ISO as repositories

As root user, follow below steps.

<details>
  <summary>Step 1: Enable the ISO</summary><br>
   1. Use the virtualization software to "insert" the ISO image file.
   
   > Note: This depends on each virtualization software. Most software has a "CD" 📀 menu which represents optical drive for this purpose.
   
   2. Use `lsblk` to verify ISO image file has been inserted.

   Commonly either `sr0` or `sr1` will be used for representing ISO file (as block device) mounted using optical drive.
   Check the size. It should be same size as ISO image file.
   
   <img width="880" height="173" alt="image" src="https://github.com/user-attachments/assets/02ad17ce-fe34-4cf9-97a1-902309e2eff8" />

</details>

<details>
  <summary>Step 2: Create mount directory</summary><br>

  Create a directory to serve as mountpoint.
  ```
   mkdir -p /repo
  ```

</details>
<details>
  <summary>Step 3: Copy ISO block device to a file and mount it</summary><br>

  1. Use `dd` utility to copy the ISO block device as a file.
     
      ```
      dd if=/dev/sr0 of=/rhel9.iso bs=1M
      ```
      
  <img width="635" height="94" alt="image" src="https://github.com/user-attachments/assets/00147173-1b49-4324-a215-559a40731c60" />

  2. Edit _/etc/fstab_ and mount the file onto directory created in Step 2. Use `iso9660` as filesystem type.
     
     <img width="946" height="166" alt="image" src="https://github.com/user-attachments/assets/f89da8d1-998c-4b6a-8b63-bacad6fa19cf" />

  3. Reload _/etc/fstab_ file to ensure no mounting error.

     ```
     systemctl daemon-reload
     mount -a
     findmnt --verify
     ```

     Once mounted, there should be 2 folders when we view the mounted directory: **BaseOS** and **AppStream**.
     These folders contain the necessary packages to serve as local repositories. 

     <img width="702" height="264" alt="image" src="https://github.com/user-attachments/assets/04fb5b19-be48-4f82-a9fb-f97f0cd01c2b" />


</details>
<details>
  <summary>Step 4: Use package manager to create local repositories </summary><br>

  Create local repositories by running: 

  ```
  dnf config-manager --add-repo=file:///repo/BaseOS
  dnf config-manager --add-repo=file:///repo/AppStream
  ```

  <img width="1156" height="330" alt="image" src="https://github.com/user-attachments/assets/6c79e601-1bfa-47ce-8d92-5902db21a797" />

</details>
<details>
  <summary>Step 5: Disable GPG check on repositories </summary><br>

  The created local repositories are enabled by default. However they will fail to function as necessary certificates are not configured. We can ignore this requirement as the repositories are local.<br>
  
  Navigate to /etc/yum.repos.d/ to view the repo metadata file that are created for each.<br>
  
  Use `vim` to edit the files and add `gpgcheck=0` for each. <br>

  <img width="603" height="108" alt="image" src="https://github.com/user-attachments/assets/c256847a-fe0a-4396-8da8-80719ed91869" />


</details>

The repositories should be usable now. Try installing an application (example: Nmap), and `dnf` now will pull necessary packages from the local repositories and install them. 

```
dnf install -y nmap
```

<img width="1273" height="344" alt="image" src="https://github.com/user-attachments/assets/352c994f-73b5-46a3-8b26-4174faf84757" />


## Outro

As we are using local repositories, Red Hat will keep displaying the message to encourage registering the VM to entitlement server. 
This is expected and can safely be ignored. 

Note that the limitation of using local repositories is that it's limited to packages available in the ISO image file. If it does not contain packages that we want, we still need to use online repositories for that. 


