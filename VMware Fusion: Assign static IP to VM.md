### Table of contents

- [Introduction](#introduction)
- [Pre-requisites](#pre-requisites)
- [Method 1: Edit VMware DHCP configuration](#method-1-edit-vmware-dhcp-configuration)
- [Method 2: Configure static IP directly in VM](#method-2-configure-static-ip-directly-in-vm)
- [Restart VMware Fusion Networking](#restart-vmware-fusion-networking)
- [Restart the VM](#restart-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

Some applications running on your VM will require the VM to maintain using same IPs, otherwise it can messed up its configurations, causing them to cease working.
This is true fif you run Kubernetes cluster on your VM, especially. 
VMware uses DHCP by default, and it does not expose interface to modify its configuration. 

Using static IP for a VM is desirable, as it prevents various kind of issues. For Kubernetes especially, the kube-apiserver IP address is selected using the current IP address of the main NIC interface (normally ens160 or eth0). During kubeadm initilization, this particular IP address is used in generating the certificates for the Kubernetes core components. 

If this IP changes, Kubernetes will stop working as the Kubernetes components will unable to talk to kube-apiserver, who is still listening on the previous IP address which is no longer exists. 
This is true for other applications that rely on having consistent IP address for listening.  

In this article, we explore how to make an IP address static for a VM. 

To assign static IPs to VM, we can either:

  🍭 Method 1: Edit VMware DHCP configuration OR, <br>
  🍭 Method 2: Configure static IP directly in VM.

Method 2 is preferable, as it does not involve editing any VMware own configurations. It also means we bypass the VMware DHCP lease mechanism altogether. 

### Pre-requisites

Ensure you note: 
  💡The current IP address & CIDR assignment. <br>
  💡The current MAC address created for the VM. <br>

Following command display both values.

```
sudo ip address
```
Normally this will be eth0 or ens160 interface IP address. 

<img width="923" height="264" alt="image" src="https://github.com/user-attachments/assets/644e2de8-5605-49fe-8509-e5533824a53a" />
<br>
<br>

### Method 1: Edit VMware DHCP configuration

Each virtualization software has own method on how to configure their built-in DHCP configuration, which it uses to assign IP address to VM. 
VMware by default does not expose this setting on its Fusion GUI. 

The DHCP configurations are defined in _dhcp.conf_ file. 
Depending on the VM's networking mode, this file exists in the following directory: 

| NAT networking mode | File location | 
|---|---|
| Host-Only| `/Library/Preferences/VMware Fusion/vmnet1/` | 
| NAT| `/Library/Preferences/VMware Fusion/vmnet8/` | 

  🚥 Note: This file already contains the pre-configured section of the built-in DHCP, which we should not modify.
     Instead, take note at the "range" field which defines the CIDR of the IP address. <br>
     The current VM IP address was picked from this range. 
     <img width="353" height="71" alt="image" src="https://github.com/user-attachments/assets/a1f90869-d42a-4b2c-a5f8-a2820e17b567" />



Steps below outline how to edit built-in DHCP configuration: 

<details>
  <summary>On MacOS 💻:</summary><br>

  1. Navigate to `/Library/Preferences/VMware Fusion/vmnet{1,8}/` and open the _dhcp.conf_ file.<br>
     In this file, you can adjust settings such as IP ranges, lease times, and other DHCP-related configurations.
     <br>
     <img width="438" height="185" alt="image" src="https://github.com/user-attachments/assets/79c7f283-fa15-402c-9104-06ce555b67de" />
     <br>

  2. Edit the _dhcp.conf_ file.
     ```
     sudo vim /Library/Preferences/VMware\ Fusion/vmnet8/dhcpd.conf
     ```
     
  3. Add the following block to the bottom of the file.

     🚥 Note: It's easier to just turn the current IP address static, but if required, we can use a new IP address within the range.
     
     ```
     host myvm {
      hardware ethernet 00:0c:29:1d:26:15;  # Replace with your VM's MAC
      fixed-address 172.16.121.208;         # Replace with current IP in subnet, or a new desired IP address. 
      }
     ```
     <br>
     After editing, save the file.
     <br>

</details>
<details>
  <summary>On Windows:</summary><br>
  

</details>

### Restart VMware Fusion Networking


### Restart the VM



### Outro

### Tips
