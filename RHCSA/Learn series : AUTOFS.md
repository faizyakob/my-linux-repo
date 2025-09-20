### Table of contents

- [Introduction](#introduction)
- [Concept](#concept)
- [Types of mapping in autofs](#types-of-mapping-in-autofs)
- [Configuring autofs service](#configuring-autofs-service)
- [Test autofs service](#test-autofs-service)
- [Tips and outro](#tips-and-outro)

### Introduction

In Linux, **autofs** is a service that automatically mounts and unmounts filesystems (like network shares, USB drives, or NFS exports) on demand.

Instead of keeping a filesystem mounted all the time, **autofs** mounts it only when a user or process tries to access it, and automatically unmounts it after a period of inactivity. 

🔑 Key points about **autofs**:

+ It is controlled by the automount daemon.
+ Uses configuration files like _/etc/auto.master_ and _/etc/auto.*_ to define mount points and rules.
+ Saves system resources by not keeping unused mounts active.
+ Commonly used for NFS mounts in enterprise environments (e.g., home directories on a central server).

**autofs** comes with many advantages. These come in handy when environment becomes more hectic, and manual task keeping getting tedious.
<details>
  <summary> Advantages</summary><br>
  
✅ No manual entries required in _/etc/fstab_ – avoiding error-prone editing, simplifies configuration and reduces static mount dependencies.

✅ Wildcard mounting for multi-user environments – automatically mounts each user’s home directory from an NFS server, enabling centralized control and easy distribution of shared resources.

✅ On-demand mounting – filesystems are mounted only when accessed, avoiding unnecessary mounts.

✅ Automatic unmounting – inactive filesystems are unmounted after a timeout, freeing resources and preventing stale mounts (especially useful with NFS).

✅ Reduced boot delays – system startup isn’t slowed down or blocked by unavailable servers or devices.

✅ Flexible support – works with local disks, NFS, CIFS/SMB, removable media, and even programmatic/script-based mounts.
</details>

👉 Example:
If _/home/users_ is managed by **autofs**, the directory _/home/users/alice_ won’t be mounted until someone accesses it, and it will unmount automatically later if idle.

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

  📍When user alice accesses ```/home/alice``` → **autofs** mounts ```vm-2:/export/home/alice```.<br>
  📍When user bob accesses ```/home/bob``` → **autofs** mounts ```vm-2:/export/home/bob```.

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
+ Cleaner configuration (than indirect mapping) → One line replaces dozens (or hundreds). 

</details>

### Configuring autofs service

Let's walkthrough simple configurations to showcase **autofs** functionality, using two demo VMs, VM-1 and VM-2. 
VM-1 will act as NFs client, while VM-2 as NFS server.
NFS has few versions, but commonly used are NFSv3 and NFSv4, in which both are running simultaneuosly for backward compatibility. 

Configurations are required on both VMs:
  + On VM-1, **autofs** service needs to be installed and running.
  + On VM-2, **nfs-server**, **mountd** and **rpcbind** services needs to be installed and running. In addition, **firewalld** needs to whitelist those services ports so NFS client can access those services.
    Whitelist similar ports in **ufw** if you are using Ubuntu.

<details>
  <summary> VM-2</summary><br>
  
1. Install and enable the ```nfs-server``` package.
     Ports used by nfs-server is 2049, while for rpcbind is 111.
     Port for mountd varies, but usually it is 20048.
     > In RHEL-based distro, nfs-server package also includes rpcbind and mountd. There is no need for separate packages install.

     As root user, run:

     ```
     dnf install -y nfs-utils
     systemctl enable --now nfs-utils
     systemctl status nfs-server
     ```
    <img width="1264" height="268" alt="image" src="https://github.com/user-attachments/assets/9ae88409-7c2e-4432-8016-5ab2852c6c70" /><br>

     To check the ports are successfully listening, run:
     ```
     rpcinfo -p
     ```
     <img width="402" height="358" alt="image" src="https://github.com/user-attachments/assets/71ad7161-0715-437b-a974-f268de0700c5" /><br>

     Output also displays the NFS version currently in used. Often, both NFSv3 and NFSv4 are running on port 2049. 

2. Whitelist the services or ports in firewall.
   If access is not opened, NFS client won't be able to reach above services.
   
    We can either add the services, or the ports they are using to **firewalld** configurations.
   
    As root user, run:

   ```
   firewall-cmd --add-service=nfs --add-service=mountd --add-service=rpc-bind --permanent
   firewall-cmd --reload
   ```
   
   Alternatively, use port numbers:
   
   ```
   firewall-cmd --add-port=111/tcp --add-port=20048/tcp --add-port=2049/tcp  --permanent
   firewall-cmd --reload
   ```

   List the services and port to ensure successful addition.

    ```
   firewall-cmd --list-services
   firewall-cmd --list-ports
   ```
   <img width="446" height="76" alt="image" src="https://github.com/user-attachments/assets/58e20f95-c211-4939-8a1b-9691f38f2cf3" /><br>

 4. Create directories and includes them in ```/etc/exports```.
    NFS uses ```/etc/exports``` file to expose the directories for mounting.

    Create 3 directories, each to showcase different type of mapping.
    > Note: We assumed user1 and user2 exist on VM-2. If not, create them and note their UID.

    ```
    mkdir -p /srv/nfs/direct
    mkdir -p /srv/nfs/indirect
    mkdir -p /srv/nfs/home/{user1,user2}
    ```
    Give appropriat permission for those directories. For demo purposes, we allow all access.

    ```
    chmod -R 755 /srv/nfs
    chown -R nobody:nobody /srv/nfs
    ```

    In real enviornment, we probably wants to limit who can access which directories.

    ```
    chmod -R 755 /srv/nfs
    chown user1:user1 /srv/nfs/home/user1
    chown user2:user2 /srv/nfs/home/user2
    ```
    
     Add following entries in ```/etc/exports``` file, each on separate line.

     + /srv/nfs/direct       *(rw,sync,no_subtree_check,no_root_squash)
     + /srv/nfs/indirect     *(rw,sync,no_subtree_check,no_root_squash)
     + /srv/nfs/home         *(rw,sync,no_subtree_check,no_root_squash)
       
     > We can replace the * with specific IP address in CIDR format. Usual mounting options are defined in brackets.
     
     Restart nfs-server service, and check export status.
   
     ```
     systemctl restart nfs-server
     exportfs -v
     ```

     If done correctly, the service should display which directories are mountable. 
     In our case, the above 3 directories.

     <img width="968" height="99" alt="image" src="https://github.com/user-attachments/assets/29fe8673-0d17-46e4-9f3b-4fc944322a55" />

</details>

<details>
  <summary> VM-1</summary><br>

NFS client is where the mountpoints will be defined. **autofs** will alleviate the tasks of mounting the filesystem, which normally requires manual configurations. Note that while **autofs** itself is a separate program that manages the automatic mounting of directories, it relies on the underlying NFS client tools to perform the actual mounting of NFS shares. 

🚧 Note that there is no need for **nfs-server** service to be running on NFS client. In fact, it MUST BE disabled, otherwise it will interfere with **autofs** service.

We'll be using ```/mnt``` subdirectories as demo for the direct & indirect mapping mountpoints. For wildcard mapping, we'll be using user1 & user2 ```/home``` directories.

Run the following commands as root user.

1. Install required packages:
   ```
   apt update
   dnf install -y nfs-utils autofs
   ```
   > For Ubuntu, **nfs-common** is used instead of **nfs-utils**.
   
2.  Enable autofs service:
     ```
     systemctl enable --now autofs
     systemctl status autofs
     ```
     <img width="1212" height="342" alt="image" src="https://github.com/user-attachments/assets/809af98b-7549-4fe0-9958-1c789555f4a5" /><br>

3.  Check NFS client is aware of the exposed directories on NFS server.

     ```
     showmount -e vm-2
     ```
     It will output the directories on NFS server available to be mounted, as defined in ```/etc/exports```.
     Also, this implies firewall whitelisting is working fine.
    
      <img width="328" height="92" alt="image" src="https://github.com/user-attachments/assets/e3e3ac93-50a4-4701-9881-c90cd5335124" /><br>
 
4.  Create /mnt subdirectories.
     ```
     mkdir -p /mnt/direct
     mkdir -p /mnt/indirect
     ```
     
5.  Edit **autofs** configuration files.
     This is the most immportant step, as it defines how the mapping will happen.

     + #### /etc/auto.master<br>
       The main **autofs** configuration file. It's the entry point where type of mapping is defined.
       For direct mapping, symbol ```/-``` is used. <br>
       For indirect mapping, the base mountpoint is defined. <br>
       For wildcard mapping, base home directory is used. <br>
       > We assumed the default value ```/home``` from useradd.<br>
       
       Enter the following entries in ```/etc/auto.master``` file, each on separate line.
        ```
        /-                   /etc/auto.direct
        /mnt/indirect        /etc/auto.indirect
        /home                /etc/auto.home
        ```
        
       Each entry is pointing to a file, where further configurations are added.
       Create all these files before proceeding with next step.

       > Alternatively, you can use autofs /etc/auto.master.d/ drop-in directory. Only difference is that each line above will be inside a separate file:<br>
       > /etc/auto.master.d/direct.autofs <br>
       > /etc/auto.master.d/indirect.autofs <br>
       > /etc/auto.master.d/wildcard.autofs <br>

    + #### /etc/auto.direct<br>
      Enter following entry in this file:
      
      ```
      /mnt/direct   vm-2:/srv/nfs/direct
      ```

      The meaning of this entry is that ```/mnt/direct``` will be the local mountpoint for the remote directory ```/srv/nfs/direct``` on NFS server.

    + #### /etc/auto.indirect<br>
      Enter following entry in this file:

      ```
      share1   vm-2:/srv/nfs/indirect
      ```

      The meaning of this entry is that ```/mnt/direct/share1``` will be the local mountpoint for the remote directory ```/srv/nfs/indirect``` on NFS server.<br>
      💡 Note that we did not create ```share1``` subdirectory prior. **autofs** will take care of this. 

    + #### /etc/auto.home<br>
      Enter following entry in this file:

      ```
      *   vm-2:/srv/nfs/home/&
      ```

      Symbol ```*``` is used, to represent anything as the subdirectory on local mountpoint, and ```&``` instruct autofs to map its corresponding directory on NFS server.<br>
      For example: <br>
      If ```/home/user1``` is accessed, autofs will mount ```srv/nfs/home/user1```. <br>
      If ```/home/user2``` is accessed, autofs will mount ```srv/nfs/home/user2```. <br>

6.  Restart **autofs** service.

     ```
     systemctl restart autofs
     ```
     
</details>
     
### Test autofs service

Once we have configured **autofs** in previous section, let's try it. 

  + Direct mapping
      Change directory to ```/mnt/direct```, and create a test file. <br>
      This file should be visible in NFS server's ```/srv/nfs/direct``` directory and vice versa.<br>
      <img width="440" height="60" alt="image" src="https://github.com/user-attachments/assets/fe33fbb5-5f2e-4c00-8367-280e3786db51" /><br>
      <img width="572" height="60" alt="image" src="https://github.com/user-attachments/assets/6b8a2bc8-0e50-4951-80f2-139e90f9a20b" /><br>

  + Indirect mapping
      Change directory to ```/mnt/indirect/share1```, and create a test file. <br>
      This file should be visible in NFS server's ```/srv/nfs/indirect``` directory and vice versa.<br>
      <img width="584" height="114" alt="image" src="https://github.com/user-attachments/assets/9282ab3d-faef-49f8-8035-dfd365277a0a" /><br>
      <img width="594" height="118" alt="image" src="https://github.com/user-attachments/assets/5482a872-8d8e-463f-bdb4-70236c65c286" /><br>

  + Wildcard mapping

      Create 2 new users on NFS client, without creating their home user: <br>
      ```
      useradd -M user1
      useradd -M user2
      ```
      Give password to these users using ```passwd```
      
      Switch to these users from root, and create a test file.<br>
      You will notice their ```/home/user1``` and ```/home/user2``` are automatically created as mountpoints. 
      
      ```
      su - user1
      su - user2
      ```
      <img width="486" height="112" alt="image" src="https://github.com/user-attachments/assets/7fa8bdc9-b230-43f8-a71b-3980d5dce34b" /><br>
      <img width="494" height="100" alt="image" src="https://github.com/user-attachments/assets/60bbf357-f175-4249-87bd-9072c52a4b14" /><br>
      <img width="686" height="130" alt="image" src="https://github.com/user-attachments/assets/8b170bfe-1eff-4eb9-9687-04981b0aca54" /><br>

      **autofs** takes care of mounting the correct remote directories, using the username as the key. <br>

      We can verify **autofs** is at work by verifyign the current mount usage.<br>
      ```
      mount | grep autofs
      ```
     
      <img width="1292" height="206" alt="image" src="https://github.com/user-attachments/assets/e0a4b2ec-320a-4d4f-ae33-1620471ada4e" />

### Tips and outro

**autofs** is used for automating the mounting of filesystems. It is most useful in multi-user environment where there are many directories mounting tasks to handle, which becoming repetitive over time. It also provides some automation with wildcard mapping, when centralized home directories is necessary to provide stricter control.

Tips:
1. If you are unable to change directory into the mountpoints, check the directory permission on the NFS server.
2. If solely using NFSv4, both **mountd** and **nfs-server** services share port 2049, so there are only 2 ports to be whitelisted. 



