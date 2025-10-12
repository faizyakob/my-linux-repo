### Table of contents

- [Introduction](#introduction)
- [Pre-requisites](#pre-requisites)
- [Method 1: Edit VMware DHCP configuration](#method-1-edit-vmware-dhcp-configuration)
- [Method 2: Configure static IP directly in VM](#method-2-configure-static-ip-directly-in-vm)
- [Restart the VM](#restart-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

Some applications running on your VM will require the VM to maintain using same IPs, otherwise it can messed up its configurations, causing them to cease working.
This is true if you run Kubernetes cluster on your VM, especially. 
VMware uses DHCP by default, and it does not expose interface to modify its configuration. 

Using static IP for a VM is desirable, as it prevents various kind of issues. For Kubernetes especially, the kube-apiserver IP address is selected using the current IP address of the main NIC interface (normally ens160 or eth0). During kubeadm initilization, this particular IP address is used in generating the certificates for the Kubernetes core components. 

If this IP changes, Kubernetes will stop working, as the Kubernetes components will unable to talk to kube-apiserver, who is still listening on the previous IP address which no longer exists. 
This is true for other applications that rely on having consistent IP address for listening.  

In this article, we explore how to make an IP address static for a VM. 

To assign static IPs to VM, we can either:<br>

  🍭 Method 1: Edit VMware DHCP configuration OR, <br>
  🍭 Method 2: Configure static IP directly in VM.

Method 2 is preferable, as it does not involve editing any VMware own configurations. It also means we bypass the VMware DHCP lease mechanism altogether. 

### Pre-requisites

Ensure you note: <br>
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
🎃

Each virtualization software has own method on how to configure their built-in DHCP server configuration, which it uses to assign IP address to VM. 
VMware by default does not expose this setting on its Fusion GUI. 

The DHCP configurations are defined in _dhcp.conf_ file. 
Depending on the VM's networking mode, this file exists in the following directory: 

| NAT networking mode | File location | 
|---|---|
| Host-Only| `/Library/Preferences/VMware Fusion/vmnet1/` | 
| NAT| `/Library/Preferences/VMware Fusion/vmnet8/` | 

  🚥 Note: This file already contains the pre-configured section of the built-in DHCP server, which we should not modify.
     Instead, take note at the "range" field which defines the CIDR of the IP address. The current VM IP address was picked from this range. <br>

     
  <img width="353" height="71" alt="image" src="https://github.com/user-attachments/assets/a1f90869-d42a-4b2c-a5f8-a2820e17b567" />
  <br>

Steps below outline how to edit built-in DHCP configuration: 

<details>
  <summary>On MacOS 💻:</summary><br>

  1. Navigate to `/Library/Preferences/VMware Fusion/vmnet{1,8}/` and open the _dhcp.conf_ file.<br>
     In this file, you can adjust settings such as IP ranges, lease times, and other DHCP-related configurations.


     <img width="438" height="185" alt="image" src="https://github.com/user-attachments/assets/79c7f283-fa15-402c-9104-06ce555b67de" />
     <br>

  1. Edit the _dhcp.conf_ file.
     ```
     sudo vim /Library/Preferences/VMware\ Fusion/vmnet8/dhcpd.conf
     ```
     
  2. Add the following block to the bottom of the file.

     🚥 Note: It's easier to just turn the current IP address static, but if required, we can use a new IP address within the range.
     
     ```
     host myvm {
      hardware ethernet 00:0c:29:1d:26:15;  # Replace with your VM's MAC
      fixed-address 172.16.121.208;         # Replace with current IP in subnet, or a new desired IP address. 
      }
     ```
     <br>
     After editing, save the file.

  3. Restart VMware Fusion networking.
     This step is required for the new configuration to take effects. 

     ```
     sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --stop
     sudo /Applications/VMware\ Fusion.app/Contents/Library/vmnet-cli --start
     ```

</details>
<details>
  <summary>On Windows:</summary><br>
  

</details>

### Method 2: Configure static IP directly in VM 
🔥

This method is recommended over changing VMware DHCP, as the configuration remains strictly in the VM itself. 
It uses the guest OS native network manager: Netplan in Ubuntu, or NetworkManager in RHEL/Debian/CentOS. 

<details>
  <summary>For Ubuntu:</summary><br>

  1. Find out the YAML file representing the active network interface. Refer "Pre-requisites" step above.
  2. Edit the YAML file
     ```
     sudo vim /etc/netplan/50-cloud-init.yaml
     ```
     <img width="1574" height="514" alt="image" src="https://github.com/user-attachments/assets/6b5db7a3-1909-4e8a-8826-2dd19a9f382c" />

     🚥 Note: You will notice the field "dhcp4" is set to **true**. This was done by VMware during creation of the VM. 
     
  4. Modify the YAML by setting "dhcp4" field to **no**, and add "addresses", "routes" and "nameservers". <br>
     "addresses" is obtained from `sudo ip address`. <br>
     "routes" is default route, and is obtained from `sudo ip route`. <br>
     "nameservers" is any public DNS. Use Google or Cloudlfare as default.

     <img width="1870" height="814" alt="image" src="https://github.com/user-attachments/assets/50757ad7-2e43-4243-a7e0-56b24fcd0b9c" />

  5. Save and exit the file.
  6. Apply the configuration.
     ```
     sudo netplan apply
     ```
     
     The VM now will be using the IP address as static IP. 
     

</details>
<details>
  <summary>For RHEL/Debian/CentOS:</summary><br>
</details>

  1. Verify current IP address, CIDR and default route.
     Also, check the connection's name, corresponding to the active interface. 

     ```
     sudo ip address
     sudo ip route
     sudo nmcli device status
     ```

     <img width="1028" height="400" alt="image" src="https://github.com/user-attachments/assets/ef0c1a7d-f510-404c-8324-a6bfb90bde16" />

    
     
  2. Edit the connection.
     ```
     nmcli con modify "ens160" \
     ipv4.method manual \
     ipv4.addresses 172.16.121.211/24 \
     ipv4.gateway 172.16.121.2 \
     ipv4.dns 8.8.8.8 \
     ipv4.ignore-auto-dns yes \
     ipv4.ignore-auto-routes yes \
     connection.autoconnect yes
     ```

      🚥 Note:<br>
         * Keep IP within VMware subnet (e.g. 172.16.121.x). Refer its _dhcp.conf_ file. <br>
         * Ensure no other device is using same IP address. <br>
         * You do not need to touch `/etc/sysconfig/network-scripts/*` manually; NetworkManager handles that.
     
  4. Restart the network connection
     ```
     sudo nmcli connection down "ens160" && sudo nmcli connection up "ens160"
     ```

     

     


### Restart the VM

Shut down and restart your VM. 

+ If you are using method 1, VM should now receive the same IP every time from the VMware DHCP server.
+ If you are using method 2, VM bypass VMware's DHCP and use its static IP address configured via Netplan or NetworkManager. 

### Outro

The VM should now have the consistent IP address each time it restarts, or shuts down and turn back on. 

### Tips

- The above work arounds should work when SSID of the network is changed. 
