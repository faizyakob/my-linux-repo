
# SELinux - Understanding and Enforcing Mandatory Access Control on Linux

### Table of contents

- [Introduction](#introduction)
- [The short answer](#the-short-answer)
- [DAC vs MAC - why SELinux exists at all](#dac-vs-mac---why-selinux-exists-at-all)
- [Core concepts](#core-concepts)
- [SELinux modes](#selinux-modes)
- [Reading a context](#reading-a-context)
- [Tutorial: Setting up a demo environment](#tutorial-setting-up-a-demo-environment)
- [Demo 1: The classic Apache DocumentRoot denial](#demo-1-the-classic-apache-documentroot-denial)
- [Demo 2: SELinux booleans](#demo-2-selinux-booleans)
- [Demo 3: SELinux and network ports](#demo-3-selinux-and-network-ports)
- [Demo 4: SELinux and containers/Kubernetes](#demo-4-selinux-and-containerskubernetes)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

[#introduction](#introduction)

The moment most people meet SELinux is the moment it ruins their day: a service that works perfectly when tested manually, refuses to start under systemd, and the only clue in the logs is a vague "Permission denied" even though `ls -l` shows the permissions are fine. The usual reaction is `setenforce 0`, the problem "goes away", and SELinux gets blamed and disabled forever without anyone actually understanding what just happened.

I wanted to write this one down properly because SELinux isn't the enemy - it's doing exactly what it was designed to do, which is refuse to trust file permissions alone. This article explains what SELinux actually is, the concepts you need to stop being confused by `avc: denied` messages, and then walks through hands-on demos: a classic web server denial, fixing it properly instead of disabling SELinux, working with booleans, opening up a non-standard port, and how all of this shows up again once you're running containers on a Kubernetes node.

None of this is Apache-specific either - the exact same subject/object/policy check runs for every process on the box, including `containerd-shim`, `kubelet`, and every container process a node runs. If you work with Kubernetes at all, this article doubles as the "why did my pod get a `Permission denied` when the file permissions look completely fine" reference too, so I've called out the container angle throughout rather than bolting it on as an afterthought at the end.

### The short answer

[#the-short-answer](#the-short-answer)

- Regular Linux permissions (`rwx`, owner/group) are **Discretionary Access Control (DAC)**. The owner of a file decides who can access it, and root can override any of it.
- SELinux is **Mandatory Access Control (MAC)**. It adds a second, independent layer of rules that even root cannot bypass. A process is only allowed to do something if both DAC *and* SELinux policy allow it.
- Every process and every file carries an SELinux **context** (a label). Policy rules say which contexts are allowed to interact with which other contexts. If there's no rule permitting it, the action is denied - even if standard file permissions say it's fine.

### DAC vs MAC - why SELinux exists at all

[#dac-vs-mac---why-selinux-exists-at-all](#dac-vs-mac---why-selinux-exists-at-all)

Standard Linux permissions have one big weakness: they trust the process. If Apache runs as root (or gets compromised while running as any user), it can read or write anything that user is allowed to touch - your SSH keys, `/etc/shadow`, other users' home directories - because DAC has no concept of "Apache should only ever touch web content".

SELinux closes that gap by asking a second question on every single access attempt:

```
Process wants to do X to a file
        │
        ▼
 DAC check (rwx, owner/group)
        │
   allowed? ──── no ──► Denied (normal "Permission denied")
        │
       yes
        │
        ▼
 SELinux check (context + policy)
        │
   allowed? ──── no ──► Denied (SELinux avc: denied)
        │
       yes
        │
        ▼
      Access granted
```

This is why a compromised web server process, confined by policy to only touch `httpd_sys_content_t` labeled files, can't read `/etc/shadow` even while running as root - DAC would happily allow it, but SELinux policy never grants `httpd_t` the right to touch `shadow_t`.

### Core concepts

[#core-concepts](#core-concepts)

- **Subjects** - the processes performing an action, identified by their **domain** (e.g. `httpd_t`, `sshd_t`, or `container_t` for a process running inside a container).
- **Objects** - the things being acted on: files, directories, sockets, and **ports**. Identified by their **type** (e.g. `httpd_sys_content_t`, `ssh_port_t`, `http_port_t`).
- **Type Enforcement (TE)** - the core mechanism. Policy defines which domains may perform which actions on which types. This is what's actually being checked on nearly every denial you'll hit, whether the process is a bare-metal daemon or a container.
- **Context (label)** - the full SELinux identity of a process, file, or port, formatted as `user:role:type:level`, e.g. `system_u:object_r:httpd_sys_content_t:s0`. In day-to-day troubleshooting, the **type** field is almost always the part that matters. Containers add one more piece to this: the `level` field also carries an **MCS category pair** (e.g. `s0:c123,c456`) that keeps otherwise-identically-typed containers isolated from each other - more on that in Demo 4.
- **Policy** - the compiled ruleset (targeted policy is the default on RHEL/Fedora/Ubuntu with SELinux enabled) that says what's allowed. You don't write raw policy day-to-day; you either relabel files/ports to the correct type or flip a boolean.
- **Booleans** - runtime on/off switches for optional chunks of policy, so you don't need to recompile policy just to allow a common variation (e.g. "let httpd make outbound network connections", or `container_manage_cgroup` for containers that need to manage their own cgroups).

### SELinux modes

[#selinux-modes](#selinux-modes)

```
$ getenforce
Enforcing
```

- **Enforcing** - policy is applied; denials actually block the action.
- **Permissive** - denials are logged but nothing is blocked. Extremely useful for troubleshooting: you can watch what *would* be denied without breaking anything.
- **Disabled** - SELinux is off entirely, no labeling, no logging. Avoid this; it throws away the whole safety net and re-enabling it later means relabeling everything from scratch.

Mode is a node-wide, kernel-level setting, so it applies to every container runtime and pod running on that node too - there's no separate "SELinux mode for containers". If `getenforce` says `Enforcing` on a Kubernetes node, every container process on it is subject to the same TE checks as any other process.

Switch temporarily (does not survive reboot):

```
$ sudo setenforce 0      # permissive
$ sudo setenforce 1      # enforcing
```

Persistent, via `/etc/selinux/config`:

```
SELINUX=enforcing
SELINUXTYPE=targeted
```

### Reading a context

[#reading-a-context](#reading-a-context)

```
$ ls -Z /var/www/html/index.html
unconfined_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html

$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0    1842 ?  00:00:00 httpd

$ ps -eZ | grep containerd-shim
system_u:system_r:container_t:s0:c123,c456   21044 ?  00:00:02 containerd-shim
```

Reading left to right: SELinux user, role, type, sensitivity level. For nearly all troubleshooting on a single-level (non-MLS) system, the **type** (`httpd_sys_content_t`, `httpd_t`, `container_t`) is the only field you need to look at. The container process's `s0:c123,c456` is the same sensitivity level slot, just with the MCS category pair filled in - that's the piece that keeps two containers, both typed `container_t`, from being able to touch each other's files.

### Tutorial: Setting up a demo environment

[#tutorial-setting-up-a-demo-environment](#tutorial-setting-up-a-demo-environment)

These demos assume a RHEL-family VM (RHEL/Rocky/AlmaLinux/Fedora), since that's where SELinux ships enforcing by default and where `httpd` behaves the way shown below. Ubuntu ships SELinux as available-but-not-default (it uses AppArmor out of the box), so if you're on Ubuntu, install and enable it first:

```
$ sudo apt install -y selinux-basics selinux-policy-default
$ sudo selinux-activate
$ sudo reboot
```

On RHEL-family systems SELinux is already enforcing, so no setup is needed there. Confirm and install the tools used below:

```
$ getenforce
Enforcing

$ sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server audit
```

- `policycoreutils-python-utils` gives you `semanage` and `audit2allow`.
- `setroubleshoot-server` gives you human-readable denial explanations.
- `audit` gives you `ausearch`, reading from `/var/log/audit/audit.log`.

### Demo 1: The classic Apache DocumentRoot denial

[#demo-1-the-classic-apache-documentroot-denial](#demo-1-the-classic-apache-documentroot-denial)

This is the single most common way people first meet SELinux: moving a website's content to a non-standard directory.

**Step 1 - Serve content from a non-default location:**

```
$ sudo mkdir -p /webdata
$ echo "hello from webdata" | sudo tee /webdata/index.html

$ sudo tee /etc/httpd/conf.d/webdata.conf <<'EOF'
<VirtualHost *:80>
    DocumentRoot /webdata
    <Directory /webdata>
        Require all granted
    </Directory>
</VirtualHost>
EOF

$ sudo systemctl restart httpd
$ sudo systemctl status httpd --no-pager
● httpd.service - The Apache HTTP Server
     Active: active (running)
```

The service starts fine - restarting `httpd` itself isn't blocked. The denial shows up on the actual request:

**Step 2 - Request the page and hit the wall:**

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
403
```

A 403 with permissions that look completely correct is the classic SELinux fingerprint.

**Step 3 - Confirm it's SELinux, not DAC:**

```
$ ls -lZ /webdata/index.html
-rw-r--r--. root root unconfined_u:object_r:default_t:s0 /webdata/index.html
```

There it is: `default_t`. Standard `rwx` permissions are fine, but the file has the generic default type, not `httpd_sys_content_t`. `httpd_t` has no policy rule allowing it to read `default_t`.

**Step 4 - Confirm the denial in the audit log:**

```
$ sudo ausearch -m avc -ts recent
type=AVC msg=audit(1721900000.123:456): avc:  denied  { read } for  pid=1842 comm="httpd"
name="index.html" dev="sda1" ino=131074 scontext=system_u:system_r:httpd_t:s0
tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```

<img width="1683" height="254" alt="image" src="https://github.com/user-attachments/assets/bf528c02-8640-43a9-8a2e-c7a8605a14ae" />

`scontext` (the process) is `httpd_t`. `tcontext` (the file) is `default_t`. That mismatch is the entire problem, spelled out explicitly.

**Step 5 - Fix it the right way: relabel, don't disable:**

Two options here. For a one-off, `restorecon` fixes it if the correct type is already known to the system for that path pattern. For a genuinely new path like `/webdata`, tell the policy about it permanently with `semanage fcontext`, then apply it:

```
$ sudo semanage fcontext -a -t httpd_sys_content_t "/webdata(/.*)?"
$ sudo restorecon -Rv /webdata
Relabeled /webdata from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /webdata/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

`semanage fcontext` records the rule persistently (survives future relabels), `restorecon` applies it to the files that already exist.

**Step 6 - Verify:**

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
200

$ ls -lZ /webdata/index.html
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 /webdata/index.html
```

No `setenforce 0` needed anywhere in that fix. The web server stayed confined the entire time.

### Demo 2: SELinux booleans

[#demo-2-selinux-booleans](#demo-2-selinux-booleans)

Some denials aren't about a mislabeled file - they're a deliberate policy default that you need to opt into changing. A common one: by default, `httpd_t` is not allowed to make outbound network connections, which breaks reverse proxy setups.

**Step 1 - Reproduce it:**

```
$ sudo tee -a /etc/httpd/conf.d/webdata.conf <<'EOF'
ProxyPass "/api" "http://127.0.0.1:5000/"
EOF
$ sudo systemctl restart httpd
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api
503
```

**Step 2 - Confirm it's the network-connect boolean, not another mislabel:**

```
$ sudo ausearch -m avc -ts recent | audit2why
type=AVC msg=audit(...): avc:  denied  { name_connect } for  pid=1901 comm="httpd" dest=5000 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket
     Was caused by:
     One of the following booleans was set incorrectly.
     Description: Allow httpd to can network connect
     Allow access by executing:
     setsebool -P httpd_can_network_connect 1
```

`audit2why` translates the raw AVC denial into a plain-English explanation and even gives you the exact fix - this is usually the fastest path from "denied" to "resolved" once you know it exists.

**Step 3 - List and flip the boolean:**

```
$ getsebool httpd_can_network_connect
httpd_can_network_connect --> off

$ sudo setsebool -P httpd_can_network_connect 1

$ getsebool httpd_can_network_connect
httpd_can_network_connect --> on
```

The `-P` makes it persistent across reboots; without it, the change only lasts until the next boot.

**Step 4 - Verify:**

```
$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost/api
200
```

To browse every boolean relevant to a service:

```
$ getsebool -a | grep httpd
```

### Demo 3: SELinux and network ports

[#demo-3-selinux-and-network-ports](#demo-3-selinux-and-network-ports)

Files aren't the only objects with a type - **ports** carry one too, and this is the demo that trips people up when they move a service off its default port, including a container's published port on a Kubernetes node.

**Step 1 - Point httpd at a non-standard port:**

```
$ sudo sed -i 's/^Listen 80/Listen 8585/' /etc/httpd/conf/httpd.conf
$ sudo systemctl restart httpd
Job for httpd.service failed because the control process exited with error code.
```

**Step 2 - Confirm it's SELinux, not just a firewall or a bad config:**

```
$ sudo journalctl -u httpd --no-pager -n 5
httpd[2210]: (13)Permission denied: AH00072: make_sock: could not bind to address [::]:8585
httpd[2210]: (13)Permission denied: AH00072: make_sock: could not bind to address 0.0.0.0:8585

$ sudo ausearch -m avc -ts recent
type=AVC msg=audit(1721900500.221:471): avc:  denied  { name_bind } for  pid=2210 comm="httpd"
src=8585 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0
tclass=tcp_socket permissive=0
```

`tcontext=unreserved_port_t` is the tell. Port 80 is labeled `http_port_t`, which `httpd_t` is allowed to bind. Port 8585 falls into the generic `unreserved_port_t` bucket, which it isn't.

**Step 3 - List which ports httpd is actually allowed to bind:**

```
$ sudo semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
```

8585 is nowhere in that list, confirming the denial makes sense rather than being a fluke.

**Step 4 - Fix it the right way: label the port, don't disable SELinux:**

```
$ sudo semanage port -a -t http_port_t -p tcp 8585
$ sudo semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000, 8585
```

Same pattern as `semanage fcontext` for files: `-a` adds a new persistent mapping, this time for a port instead of a path.

**Step 5 - Verify:**

```
$ sudo systemctl restart httpd
$ sudo systemctl status httpd --no-pager | grep Active
     Active: active (running)

$ curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8585/
200
```

If you'd instead used `semanage port -m` (modify) by mistake on a port already owned by another service type, you'd have just broken that other service - always check with `-l` first to see whether the port is already claimed before adding or modifying.

This exact same denial shows up with containers: if a pod's `containerPort` is set to something outside the range its process's SELinux type is allowed to bind (less common with `container_t` itself, which has broad `name_bind` allowances, but very real for services confined to a tighter custom type, or when a hostNetwork pod binds directly to a node port), `semanage port -a` on the node is the same fix - the container's bind attempt is still a `name_bind` check against the node's port policy, whether the process making it happens to live in a container or not.

### Demo 4: SELinux and containers/Kubernetes

[#demo-4-selinux-and-containerskubernetes](#demo-4-selinux-and-containerskubernetes)

This is the part that actually matters once you move from a standalone VM to running workloads on a Kubernetes node, since the exact same DAC-then-MAC check happens for container processes too - as seen already with `containerd-shim` running as `container_t` back in "Reading a context." Each container additionally gets its own **MCS (Multi-Category Security) categories** (the `c123,c456` part of that context), so even two containers sharing the exact same `container_t` type are still isolated from each other's files by their distinct category pair - it's SELinux quietly doing an extra layer of container-to-container isolation on top of namespaces and cgroups.

**Step 1 - Reproduce a hostPath volume denial:**

A pod mounting a host directory that was never labeled for container access is the container-world equivalent of Demo 1's `/webdata`:

```
$ sudo mkdir -p /hostdata
$ echo "node data" | sudo tee /hostdata/file.txt
$ ls -Z /hostdata/file.txt
unconfined_u:object_r:default_t:s0 /hostdata/file.txt
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: reader
    image: busybox
    command: ["sh", "-c", "cat /data/file.txt; sleep 3600"]
    volumeMounts:
    - name: hostvol
      mountPath: /data
  volumes:
  - name: hostvol
    hostPath:
      path: /hostdata
```

```
$ kubectl apply -f hostpath-demo.yaml
$ kubectl logs hostpath-demo
cat: can't open '/data/file.txt': Permission denied
```

Standard `rwx` permissions on `/hostdata/file.txt` are fine - this is `container_t` being denied access to `default_t`, exactly like `httpd_t` was denied access to it in Demo 1.

**Step 2 - Confirm on the node:**

```
$ sudo ausearch -m avc -ts recent | grep container_t
type=AVC msg=audit(...): avc:  denied  { read } for  pid=21088 comm="cat"
scontext=system_u:system_r:container_t:s0:c123,c456
tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```

**Step 3 - Fix it the container-aware way:**

You have two real options, and which one is correct depends on the situation:

- **Relabel the host path to the container-shared type**, same tool as before, just a different target type:

  ```
  $ sudo semanage fcontext -a -t container_file_t "/hostdata(/.*)?"
  $ sudo restorecon -Rv /hostdata
  ```

- **Or let the kubelet/container runtime relabel it automatically per-pod**, by adding an SELinux options block to the pod spec instead of touching the node manually:

  ```yaml
    securityContext:
      seLinuxOptions:
        level: "s0:c123,c456"
  ```

  This is the Kubernetes-native equivalent of what `restorecon` does by hand - it tells the kubelet to relabel the mounted volume with the pod's own MCS categories on mount, which is generally the safer choice in a shared cluster since it keeps the isolation between unrelated pods intact rather than opening the host path to every container broadly via `container_file_t`.

**Step 4 - Verify:**

```
$ kubectl delete pod hostpath-demo
$ kubectl apply -f hostpath-demo.yaml
$ kubectl logs hostpath-demo
node data
```

**Step 5 - One more thing worth knowing: never fix this with `setenforce 0` on a node.**

On a shared Kubernetes node, disabling SELinux to fix one pod's `hostPath` denial removes the MCS category isolation between *every* container running on that node, not just the one you were debugging - it's a much bigger blast radius than disabling SELinux on a single-purpose VM.

### Outro

[#outro](#outro)

- A `Permission denied` with `rwx`/owner permissions that look completely correct is the single biggest tell that you're looking at an SELinux denial, not a DAC one.
- `setenforce 0` (or `SELINUX=disabled`) is a diagnostic tool, not a fix. Use permissive mode to confirm SELinux is the cause, then relabel or flip a boolean, and go back to enforcing.
- `restorecon` fixes files that already have a known-correct type recorded for their path; `semanage fcontext` is what teaches the system a *new* path pattern in the first place. New paths need both, in that order.
- Booleans exist for the common, intentional policy variations (like outbound network access). `audit2why` will usually tell you directly which boolean to flip.
- Ports carry a type just like files do. A `name_bind` denial when moving a service to a non-standard port is `semanage port -a` territory, and it's the same underlying check whether the process binding is a bare-metal daemon or a container.
- On Kubernetes nodes, the exact same subject/object/policy check applies to container processes, just with an extra MCS category layer for container-to-container isolation on top. A `hostPath` denial is the containerized version of Demo 1's `/webdata` denial, and a container bind-to-port denial is the containerized version of Demo 3's port denial - same root causes, same fix patterns, just with `container_t`/MCS categories in the context.

### Tips

[#tips](#tips)

- Quick way to watch denials live while reproducing an issue:

    `sudo tail -f /var/log/audit/audit.log | grep AVC`

- Generate a custom policy module from accumulated denials, for the rare case where a real app genuinely needs a new rule instead of an existing boolean (review the output before loading it - never load `audit2allow -a` output blindly):

    `sudo ausearch -m avc -ts recent | audit2allow -M mypolicy && sudo semanage module -a -f mypolicy.pp -p mypolicy.pp`

- `sealert` (from `setroubleshoot-server`) turns raw AVC logs into a plain-English explanation with a suggested fix, similar to `audit2why` but more detailed:

    `sudo sealert -a /var/log/audit/audit.log`

- To see every context type applied under a directory tree at a glance:

    `ls -RZ /var/www/html`

- To check what type owns a port before assuming a `name_bind` denial needs a new mapping (it might already belong to a different service):

    `sudo semanage port -l | grep <port_number>`

- On a Kubernetes node, `crictl inspect <container-id>` will show you the exact SELinux label (including MCS categories) a running container was given, which is the container-world equivalent of `ps -eZ`:

    `sudo crictl inspect <container-id> | grep -i selinux`
