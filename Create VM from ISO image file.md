
In this article, I documented the processes of spinning up a Linux VM using VMware Fusion Pro. 

We can use any supported Linux distros, but you must be aware of the CPU architecture that your OS is using.
+ Windows uses x64-based CPU type.
+ MacOS uses ARM64-based CPU type.

Since I am using MacOS running M3 chip, I searched for distros that supported ARM64 architecture. 
I came across Oracular Ubuntu at [Ubuntu 24.10 (Oracular Oriole)](https://cdimage.ubuntu.com/releases/oracular/release/) which supports it.
"Oracular" is actually a codename for Ubuntu version 24.10.

> Today Canonical announced the release of Ubuntu 24.10, codenamed “Oracular Oriole”

This Ubuntu VM can then be used for many purposes, for example in creating a local Kubernetes cluster. 

## Table of contents

- [Download Ubuntu distro image](#download-ubuntu-distro-image)

## Download Ubuntu distro image

Go ahead and download its ISO image. The downloader should automatically detects your local OS including its CPU architecture, and downloads the correct ISO image. 
From this point, I'll be referring Ubuntu as distro of choice.
