### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [Types of mapping in autofs](#types-of-mapping-in-autofs)
- [Download Ubuntu distro image](#download-ubuntu-distro-image)
- [Load VM onto VMware Fusion](#load-vm-onto-vmware-fusion)
- [Start the VM](#start-the-vm)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

In Linux, **autofs** is a service that automatically mounts and unmounts filesystems (like network shares, USB drives, or NFS exports) on demand.

Instead of keeping a filesystem mounted all the time, autofs mounts it only when a user or process tries to access it, and automatically unmounts it after a period of inactivity. 

🔑 Key points about **autofs**:

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

### Concept

When a user tries to access a local directory (aka mountpoint) in NFS client, **autofs** service will automatically mount the corresponding directory that is mapped on the NFS server for the user. 
**autofs** utilizes few Systemd services for this: mountd, nfs-server and rpcbind. Therefore, these services and their ports need to be running and opened respectively on the NFS server.

<img width="1042" height="300" alt="image" src="https://github.com/user-attachments/assets/9400e4cf-3fa5-4729-88bb-f33731e08e41" />

This mapping between local mountpoints and remote directories determined how those directories are accessed. Mounting options can be specified similarly as in _/etc/fstab_.

**autofs** uses dedicated configuration files in order to work: _/etc/auto.master_ and/or additional configuration files under drop-in directory _/etc/auto.master.d/_.

### Types of mapping in autofs

We can define 3 types of mapping in autofs: 
+ Direct mapping
+ Indirect mapping
+ Wildcard mapping


<details>
  <summary> Direct mapping</summary><br>
  
Direct mapping means that specific filesystem/mountpoint paths are mapped directly to remote (or local) filesystems without going through a parent "mount directory".
As such, 🚩absolute paths MUST BE provided when using direct mapping. 

🔑 How direct mapping Works

1. Uses _/etc/auto.master_ with the special entry ```/-```.
2. Each entry in the corresponding map file points to an absolute path in the client filesystem.
3. When a user or process accesses that exact path, **autofs** mounts the target automatically.

Advantages of using direct mapping is that it lets you mount filesystems exactly where you want them in the directory tree. This is useful when precise and consistency are required all the time.
The specific local filesystem/mountpoint can sync only with the specific remote directory defined in the mapping, nothing else. It's a 1-1 relation.

<img width="1028" height="172" alt="image" src="https://github.com/user-attachments/assets/d28d7934-e7b9-4083-b783-ba6ebba72b32" />

In above sceenshot example, remote directory ```/srv/nfs/direct``` will autmoatically get mounted when a client attemps to access local directory ```/mnt/direct```.

⚠️ It is important to remember that the local mountpoint ```/mnt/direct``` must already existed before we can use direct mapping.

</details>

<details>
  <summary> Indirect mapping</summary><br>
  
Indirect mapping is a type where you specifcy a remote directory and a "base" mountpoint in the local system. Under this base directory, subdirectories are created automatically as per defined (as keys) in configuration files. Compares to direct mapping, these subdirectories do not have to be created in advance, as they are created on the fly by **autofs**.
It is more common style (compared to direct mapping), and widely used in multi-user environment. 

🔑 How indirect mapping Works

1. You define a base mountpoint (a directory) in _/etc/auto.master_.
2. A separate map file contains relative keys that expand under that base mountpoint.
3. When a user accesses one of those subdirectories, **autofs** mounts the corresponding remote filesystem.

Indirect mapping scales better than direct mapping when managing many users or directories.

<img width="1026" height="170" alt="image" src="https://github.com/user-attachments/assets/89676484-50a4-4aef-9535-e6acf32d6aeb" />

In above screenshot example, remote directory ```/srv/nfs/indirect``` will automatically get mounted when a client attemps to access local directory ```/mnt/direct/share1```. However, ```share1``` mountpoint will be automatically created on the client when user cd into it. 

</details>

</details>

<details>
  <summary> Wildcard mapping</summary><br>

  Wildcard mapping works almost similarly like indirect mapping, with exception that no keys need to be defined prior in **autofs** configuration files. 
  Most useful usage of wildcard mapping is in provisioning users' home directories. In this scenario, for example: 

  📍When user alice accesses ```/home/alice``` → **autofs** mounts ```vm2:/export/home/alice```.<br>
  📍When user bob accesses ```/home/bob``` → **autofs** mounts ```vm2:/export/home/bob```.

  There is no need to hardcode each username as key in **autofs** configuration file. Therefore it works best in environments where server and client username directories are consistent.

🔑 How wildcard mapping works?

1. Instead of defining each mount explicitly (e.g., alice, bob, charlie), you use the wildcard character *.
2. The * matches any key requested under the base directory.
3. The & symbol inside the NFS path expands to the same key name.

<img width="1034" height="172" alt="image" src="https://github.com/user-attachments/assets/02508959-2a2e-4127-8c7f-c12ecee8da22" />

In above screenshot example, when a user accesses his home directory, the corresponding directory on NFS sever is mounted. **autofs** will use the username to map it to correct directory. 

⚠️ Things to note : If a user directory doesn’t exist on the NFS server, the path still shows up but will fail on access.

✅ Advantages

+ Scales automatically → No edits required when new users are added on the NFS server.
+ Centralized home directories → Each user’s /home/username is fetched dynamically.
+ Cleaner configuration → One line replaces dozens (or hundreds).

</details>
