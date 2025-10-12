### Table of contents

- [Introduction](#introduction)
- [Method 1: Edit VMware DHCP configuration](#method-1:-edit-vmware-dhcp-configuration)
- [Method 2: Configure static IP directly in VM](#method-2:-configure-static-ip-directly-in-vm)
- [Restart the VM](#restart-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

Some applications running on your VM will require the VM to maintain using same IPs, otherwise it can messed up its configurations, causing them to cease working.
This is true fif you run Kubernetes cluster on your VM, especially. 
VMware uses DHCP by default, and it does not expose interface to modify its configuration. 
To assign static IPs to VM, we can either:

  🍭 Edit VMware DHCP configuration OR,
  🍭 Configure static IP directly in VM.

Using static IP for a VM is desirable, as it prevents various kind of issues. For Kubernetes especially, the kube-apiserver IP address is selected using the current IP address of the main NIC interface (normally ens160 or eth0). During kubeadm initilization, this particular IP address is used in generating the certificates for the Kubernetes core components. 

If this IP changes, Kubernetes will stop working as the Kubernetes components will unable to talk to kube-apiserver, who is still listening on the previous IP address which is no longer exists. 
This is true for other applications that rely on having consistent IP address for listening.  
