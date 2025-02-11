### Table of contents

- [Introduction](#introduction)
- [Download VMware Fusion Pro](#download-vmware-fusion-pro)
- [Download Ubuntu distro image](#download-ubuntu-distro-image)

### Introduction

In this article, I documented the processes of spinning up a Linux VM using VMware Fusion Pro. 
We can use alternative tools like Hyper-V manager (Windows), UTM or Parallels, but VMware Fusion is what I am familiar with. 

We can use any supported Linux distros for the guest OS, but you must be aware of the CPU architecture that your host OS is using. 
+ Windows uses x64-based CPU type.
+ MacOS uses ARM64-based CPU type.

The guest OS must be using similar CPU architecture as the host OS. 

Since I am using MacOS running M3 chip, I searched for distros that supported ARM64 architecture. 
I came across Oracular Ubuntu at [Ubuntu 24.10 (Oracular Oriole)](https://cdimage.ubuntu.com/releases/oracular/release/) which supports it.
"Oracular" is actually a codename for Ubuntu version 24.10.

> Today Canonical announced the release of Ubuntu 24.10, codenamed “Oracular Oriole”

This Ubuntu VM can then be used for many purposes, for example in creating a local Kubernetes cluster. 
Following this point, I'll be referring Ubuntu as distro of choice. 

## Steps

### Download VMware Fusion Pro

Since May 2024, VMware has released VMware Fusion Pro free for personal use : [VMware Fusion Pro: Now Available Free for Personal Use](https://blogs.vmware.com/teamfusion/2024/05/fusion-pro-now-available-free-for-personal-use.html). 
> We now provide a Free Personal Use or a Paid Commercial Use subscription for our Pro apps.

Follow the embedded link inside the blog to download a copy and install VMware Fusion Pro. You still need to register though before downloading.

### Download Ubuntu distro image

Download the ISO image from link in [Introduction](#introduction).
The downloader should automatically detects your host OS including its CPU architecture, and downloads the correct ISO image. 
If it's not, manually search the correct ISO image. 
At time of writing, the file name are <ins>ubuntu-24.10-desktop-arm64.iso</ins> and <ins>ubuntu-24.10-desktop-amd64.iso</ins> for ARM64 and x64 architecture respectively. 

### Load VM onto VMware Fusion

Run VMware Fusion Pro application. 


