### Table of contents

- [Introduction](#introduction)
- [Download VMware Fusion Pro](#download-vmware-fusion-pro)
- [Download Ubuntu distro image](#download-ubuntu-distro-image)
- [Load VM onto VMware Fusion](#load-vm-onto-vmware-fusion)
- [Start the VM](#start-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

In this article, I documented the processes of spinning up a Linux VM using VMware Fusion Pro. 
VMware Fusion Pro currently only supports MacOS, but if you are on Windows machine, alternative tools are available like Hyper-V manager (Windows), UTM or Parallels. 
The way to configure a VM on these tools might slightly differ from VMware Fusion. 

We can use any supported Linux distros for the guest OS, but you must be aware of the CPU architecture that your host OS is using. 
+ Windows uses x64-based CPU type.
+ MacOS with Intel chip uses x64-based CPU type.
+ MacOS with M1,M2 or M3 chip uses ARM64-based CPU type.

The guest OS must be using similar CPU architecture as the host OS. 

Since I am using MacOS running M3 chip, I searched for distros that supported ARM64 architecture. 
I came across Oracular Ubuntu at [Ubuntu 24.10 (Oracular Oriole)](https://cdimage.ubuntu.com/releases/oracular/release/) which supports it.
"Oracular" is actually a codename for Ubuntu version 24.10.

> Today Canonical announced the release of Ubuntu 24.10, codenamed “Oracular Oriole”

This Ubuntu VM can then be used for many purposes, for example in creating a local Kubernetes cluster. 
Following this point, I'll be referring Ubuntu as distro of choice. 

## Steps

### Download VMware Fusion Pro

In May 2024, VMware has released VMware Fusion Pro free for personal use : [VMware Fusion Pro: Now Available Free for Personal Use](https://blogs.vmware.com/teamfusion/2024/05/fusion-pro-now-available-free-for-personal-use.html). 
> We now provide a Free Personal Use or a Paid Commercial Use subscription for our Pro apps.

Follow the embedded link inside the blog to download a copy and install VMware Fusion Pro. You still need to register before downloading.

### Download Ubuntu distro image

Download the ISO image from link in [Introduction](#introduction).
The downloader should automatically detects your host OS including its CPU architecture, and downloads the correct ISO image. 
If it's not, manually search the correct ISO image. 
At time of writing, the file name are <ins>ubuntu-24.10-desktop-arm64.iso</ins> and <ins>ubuntu-24.10-desktop-amd64.iso</ins> for ARM64 and x64 architecture respectively. 

### Load VM onto VMware Fusion

Steps:
1. Run VMware Fusion Pro application.
2. Click "New" from the drop down.
   <img width="1136" alt="Screenshot 2025-02-11 at 15 43 49" src="https://github.com/user-attachments/assets/45257a89-9537-455b-9b93-38defa163329" />
3. On the _Create a New Virtual Machine_ window, choose the ISO image downloaded earlier.
   <img width="752" alt="Screenshot 2025-02-11 at 15 44 29" src="https://github.com/user-attachments/assets/d2e8136c-2889-46ec-94d3-03559dae3970" />
4. Click "Customize Settings" if you want to use non-default configurations. Here, you can modify the settings related to the VM like disk capacity, CPU and RAM.
   
   Please note if you intend to create the VM as part of a Kubernetes cluster, recommended to give each VM as much CPU and RAM as possible. Recommended is 2 CPUs and 4GB RAM.
   For worker nodes, accomodate extra disk spaces for pods' applications.
   
6. Click "Finish" to proceed.
   ![image](https://github.com/user-attachments/assets/263f730f-54ee-4314-a06a-9ec92062c770)

### Start the VM
1. The new VM should be displayed in the VMWware Fusion main window.
   ![image](https://github.com/user-attachments/assets/42b2a7d4-bd27-4219-ae12-73854f86336f)
2. Double-click it to start the VM.
3. After booting up for the first time, you will need to run the standard OS installation processes, including creating a new root and user account. Follow the on-screen instructions to finish it.
   ![Screenshot 2025-02-11 at 15 46 19](https://github.com/user-attachments/assets/472db194-4a59-4f4f-b153-b936fd0785cf)
   ![image](https://github.com/user-attachments/assets/744f37b0-df23-4379-b554-af784951e5a6)
4. Once configurations are completed, the VM will reboot, and should provide the login screen. Log in using the account created previously.
   ![image](https://github.com/user-attachments/assets/d56fb563-96aa-4117-bf50-b894c26a2a18)
5. If everything is fine, you will see the Desktop. Ensure you can open the Terminal application. 
   ![image](https://github.com/user-attachments/assets/b19202c0-4924-4a57-8312-b26ec469113a)
6. VM setup is now completed. You now have a running Ubuntu VM.

### Outro
* Steps in this article is not limited to ISO image for Linux distros that provides a GUI. You can also use the steps for CLI-only ISO image. 
* If you use default configurations of the VM to deploy the VM, you can always modify its properties later, even with the VM's OS already been installed. You might need to shutdown the VM first though.
* Be very careful when expanding RAM or disk capacity of the VM, as this capacity is taken from the host OS, which might become under-resourced due to this expansion operation.

### Tips

* VMware Fusion will automatically assign an IP adress using DHCP for each VM. The IP address pool depends on the ISP. Most of the time, IP adress will stick. If it does not, we will need further configuration to make the assinged IP static. TBD.
* For copy and paste from host machine to guess VM, install Open VM tools. Using root user:
  
  `apt-get -y upgrade && apt-get -y update`
  
  `apt-get install -y open-vm-tools-desktop`
  
  `reboot`

* Install additional helpful tools like `curl` and text editor:

   `apt-get install -y curl vim`
  
  
   



