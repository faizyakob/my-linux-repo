### Table of contents

- [Introduction](#introduction)
- [Types of mounting in autofs](#types-of-mounting-in-autofs)
- [Download Ubuntu distro image](#download-ubuntu-distro-image)
- [Load VM onto VMware Fusion](#load-vm-onto-vmware-fusion)
- [Start the VM](#start-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

In Linux, **autofs** is a service that automatically mounts and unmounts filesystems (like network shares, USB drives, or NFS exports) on demand.

Instead of keeping a filesystem mounted all the time, autofs mounts it only when a user or process tries to access it, and automatically unmounts it after a period of inactivity. 

🔑 Key points about autofs:

+ It is controlled by the automount daemon.
+ Uses configuration files like _/etc/auto.master_ and _/etc/auto.*_ to define mount points and rules.
+ Saves system resources by not keeping unused mounts active.
+ Commonly used for NFS mounts in enterprise environments (e.g., home directories on a central server).

**autofs** comes with many advantages. These come in handy when environment becomes more hectic, and manual task keeping getting tedious.
<details>
  <summary> Advantages</summary><br>
  
✅ No manual entries required in /etc/fstab – avoiding error-prone editing, simplifies configuration and reduces static mount dependencies.

✅ Wildcard mounting for multi-user environments – automatically mounts each user’s home directory from an NFS server, enabling centralized control and easy distribution of shared resources.

✅ On-demand mounting – filesystems are mounted only when accessed, avoiding unnecessary mounts.

✅ Automatic unmounting – inactive filesystems are unmounted after a timeout, freeing resources and preventing stale mounts (especially useful with NFS).

✅ Reduced boot delays – system startup isn’t slowed down or blocked by unavailable servers or devices.

✅ Flexible support – works with local disks, NFS, CIFS/SMB, removable media, and even programmatic/script-based mounts.
</details>

👉 Example:
If _/home/users_ is managed by autofs, the directory _/home/users/alice_ won’t be mounted until someone accesses it, and it will unmount automatically later if idle.

### Types of mounting in autofs
