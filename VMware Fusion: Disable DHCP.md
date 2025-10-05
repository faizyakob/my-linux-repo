### Table of contents

- [Introduction](#introduction)
- [Download VMware Fusion Pro](#download-vmware-fusion-pro)
- [Download Ubuntu distro image](#download-ubuntu-distro-image)
- [Load VM onto VMware Fusion](#load-vm-onto-vmware-fusion)
- [Start the VM](#start-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

Some applications running on your VM will require the VM to maintain using same IPs, otherwise it can messed up its configurations, causing them to cease working.
This is true for Kubernetes, especially. 
VMware uses DHCP by default, and it does not expose interface to modify its configuration. 
To assign static IPs to VM, we can either:

  🍭 Edit VMware DHCP configuration, OR
  🍭 Configure static IP directly in VM.
