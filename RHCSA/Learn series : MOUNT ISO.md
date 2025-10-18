
## Table of contents

- [Introduction](#introduction)
- [Steps to use ISO as repositories](#steps-to-use-iso-as-repositories)

## Introduction

In this article, we walkthrough how to mount an ISO file as a repository. <br>
Normally, depending on the distros, we will register the VM with the respective subscription management service. When a VM is registered, a repository file is created automatically that will point to available online repositories, which then allow users to download and install package via the native package manager like `apt` or `dnf`.

For Red Hat Linux, we use `subscription-manager` tool to register the VM.

As root user, run `subscription-manager status` to check if VM is registered. <br>

<img width="5100" height="450" alt="image" src="https://github.com/user-attachments/assets/bd6c3cc2-4f85-4b29-a2b5-648ebfc3af36" />


However, there are cicumstances where we do not want to connect VM with online repositories, but still want to download and install packages. <br>

In this case, we can use the VM's ISO image and mount it to a directory. The ISO image file already contains a huge number of packages, which will serve as local repostiories nicely. <br>

(Ref: [VMware 00 Create VM from ISO image file](<VMware 00 Create VM from ISO image file.md>) 

## Steps to use ISO as repositories

As root user, follow below steps.

<details>
  <summary>Step 1: Enable the ISO</summary><br>
   1. Use the virtualization software to "insert" the ISO image file.
   
   > Note: This depends on each virtualization software. Most software has a "CD" 📀 menu which represents optical drive for this purpose.
   
   2. Use `lsblk` to verify ISO image file has been inserted.

   Commonly either `sr0` or `sr1` will be used.
   Check the size. It should be same size as ISO image file.
   
   <img width="880" height="173" alt="image" src="https://github.com/user-attachments/assets/02ad17ce-fe34-4cf9-97a1-902309e2eff8" />

</details>

<details>
  <summary>Step 2: Create mount directory</summary><br>

1. Create a directory to serve as mountpoint.
  ```
   mkdir -p /repo
  ```

</details>
