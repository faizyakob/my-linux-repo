
# du vs df - Understanding Disk Usage vs Disk Free

### Table of contents

- [Introduction](#introduction)
- [The short answer](#the-short-answer)
- [How du works](#how-du-works)
- [How df works](#how-df-works)
- [Why they can disagree](#why-they-can-disagree)
- [Walkthrough 1: df says full, du says otherwise](#walkthrough-1-df-says-full-du-says-otherwise)
- [Walkthrough 2: du on a directory vs df on its mount point](#walkthrough-2-du-on-a-directory-vs-df-on-its-mount-point)
- [Walkthrough 3: Kubernetes node hits DiskPressure](#walkthrough-3-kubernetes-node-hits-diskpressure)
- [Outro](#outro)
- [Tips](#tips)

### Introduction

[#introduction](#introduction)

I keep seeing people mix up `du` and `df` whenever a system starts complaining about low disk space, so I wanted to write this one down properly.
They sound similar, both report "space" in some way, and both output numbers in similar units. But they measure completely different things, and mixing them up wastes a lot of troubleshooting time.

This article breaks down what each command actually does, why their numbers can look contradictory even when nothing is wrong, and then walks through three real investigations step by step, with the kind of output you'd actually see on a terminal, including one that happens at the Kubernetes node level, since I've hit this exact problem while running a cluster.

### The short answer

[#the-short-answer](#the-short-answer)

- `du` (**d**isk **u**sage) walks through files and directories and adds up how much space they actually consume. It answers: *"how much space are these files using?"*
- `df` (**d**isk **f**ree) asks the filesystem itself how much space is used and free on a mounted volume. It answers: *"how much space is left on this filesystem?"*

`du` looks at files. `df` looks at filesystems. That's the whole distinction, but it has a lot of consequences.

### How du works

[#how-du-works](#how-du-works)

`du` recursively scans a directory tree and sums up the disk blocks allocated to each file it finds.

Common usage:

`du -sh /var/log`

- `-s` summarizes into a single total instead of listing every subdirectory.
- `-h` prints human-readable sizes (K, M, G) instead of raw block counts.

To see which subdirectories are the biggest offenders one level down:

`du -h --max-depth=1 /var | sort -rh`

Because `du` only counts files it can actually see and walk through, it has no idea about anything happening at the filesystem level that isn't tied to a visible file.

### How df works

[#how-df-works](#how-df-works)

`df` doesn't touch individual files at all. It queries the filesystem's own metadata (superblock) for the mounted volume and reports total size, used space, available space, and mount point.

Common usage:

`df -h`

Shows all mounted filesystems in human-readable form.

`df -h /var/log`

Shows just the filesystem that a specific path lives on.

`df -i`

Shows inode usage instead of block usage. This matters because a filesystem can run out of inodes (the "slots" that track files) well before it runs out of raw space, especially on filesystems with millions of tiny files.

### Why they can disagree

[#why-they-can-disagree](#why-they-can-disagree)

A few reasons `df` and `du` won't match, even on a perfectly healthy system:

- **Deleted-but-open files.** If a process still has a file handle open on a file that was deleted, the filesystem still reserves that space until the process closes it or is restarted. `du` can't see the file anymore (it's unlinked from the directory tree), but `df` still counts it as used.
- **Reserved blocks.** Filesystems like ext4 reserve a percentage of space (5% by default) for the root user, so regular users can't ever completely fill the disk. `df` includes this in "total", `du` has no concept of it at all.
- **Sparse files.** A file can report a large logical size but only occupy a fraction of that on disk (holes are not physically written). `du` reports actual disk blocks used, which can be smaller than what `ls -l` shows.
- **Other mount points nested inside a directory.** If you run `du` on a parent directory, it will include the size of anything mounted underneath it too (unless you pass `-x`/`--one-file-system`), while `df` reports per-filesystem, not per-directory.
- **Multiple filesystems sharing a "directory view."** Bind mounts, overlay filesystems (common in containers), and separate partitions can all make a single directory tree span multiple actual filesystems, so `du` on a path and `df` on that same path can be answering different questions entirely.

### Walkthrough 1: df says full, du says otherwise

[#walkthrough-1-df-says-full-du-says-otherwise](#walkthrough-1-df-says-full-du-says-otherwise)

This is the classic one, and it's worth walking through with real-looking numbers so the gap is obvious.

**Step 1 — Confirm the alert with df:**

```
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   47G  1.2G  98% /
```

98% used, 1.2G free. That matches whatever monitoring alert fired.

**Step 2 — Try to find the culprit with du:**

```
$ du -sh /var/log /home /opt /var/lib 2>/dev/null
1.1G    /var/log
3.4G    /home
890M    /opt
6.2G    /var/lib
```

Adding those up gives roughly 11.5G. That's nowhere near the 47G `df` reports as used. Something is invisible to `du`.

**Step 3 — Look for deleted-but-open files:**

```
$ sudo lsof +L1
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NLINK NODE NAME
nginx     1842  www    2w   REG   8,1   34G       0   9821 /var/log/nginx/access.log (deleted)
```

There it is. `nginx` has a 34G log file open that was already deleted (probably by an aggressive `rm` during a manual cleanup, or a `logrotate` config that removes the old file without telling nginx to reopen it). The `NLINK` column showing `0` confirms it's unlinked from the directory tree, which is exactly why `du` never saw it.

**Step 4 — Reclaim the space:**

The safest fix is to make the process release the handle cleanly:

```
$ sudo kill -USR1 1842
```

(nginx reopens its log files on `SIGUSR1`.) For processes that don't support a reopen signal, a full restart of the service works too.

**Step 5 — Verify:**

```
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        50G   13G   35G  27% /
```

Space is back. Going forward, switching `logrotate` to `copytruncate` for this log, or making sure it sends the reopen signal after rotating, prevents this from happening again.

### Walkthrough 2: du on a directory vs df on its mount point

[#walkthrough-2-du-on-a-directory-vs-df-on-its-mount-point](#walkthrough-2-du-on-a-directory-vs-df-on-its-mount-point)

**Step 1 — Check the directory you care about:**

```
$ du -sh /data
40G     /data
```

**Step 2 — Check the filesystem it lives on:**

```
$ df -h /data
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       200G  180G   20G  90% /data
```

40G vs 180G used. Before assuming something's broken, confirm what else is actually on that filesystem:

**Step 3 — Confirm the mount point, then check siblings:**

```
$ findmnt /data
TARGET SOURCE     FSTYPE OPTIONS
/data  /dev/sdb1  ext4   rw,relatime

$ du -h --max-depth=1 /data 2>/dev/null | sort -rh
40G     /data/postgres
12K     /data/tmp

$ ls -la /data
drwxr-xr-x  4 postgres postgres 4096 Jul 10 09:12 postgres
drwxr-xr-x  2 root     root     4096 Jul  9 14:03 tmp
drwx------  2 root     root    16384 Jan  1  2025 lost+found
```

Here, `/data/postgres` only accounts for 40G, but `/dev/sdb1` shows 180G used. This usually means either another process is writing directly into that mount outside of what you're scanning (check permissions and whether `du` silently skipped a directory it couldn't read — rerun with `sudo`), or the reserved-blocks/sparse-file effects mentioned earlier are in play. Re-running `sudo du -sh /data` (with root privileges, in case permission errors were silently swallowed) is always worth doing before concluding something is genuinely wrong.

### Walkthrough 3: Kubernetes node hits DiskPressure

[#walkthrough-3-kubernetes-node-hits-diskpressure](#walkthrough-3-kubernetes-node-hits-diskpressure)

This is the scenario that actually got me to write this article. It trips people up because the symptom shows up in `kubectl`, but the root cause is a plain `df`-vs-`du` mismatch happening on the node.

**Step 1 — Notice pods getting evicted:**

```
$ kubectl get events -A --field-selector reason=Evicted
NAMESPACE   LAST SEEN   TYPE      REASON     OBJECT               MESSAGE
default     3m          Warning   Evicted    pod/api-7d9f6-xk2pl  The node was low on resource: ephemeral-storage.
```

**Step 2 — Check the node condition:**

```
$ kubectl describe node worker-2
Conditions:
  Type             Status  Reason                       Message
  ----             ------  ------                       -------
  DiskPressure     True    KubeletHasDiskPressure        kubelet has disk pressure
```

The kubelet's eviction manager compares `nodefs.available` (and `imagefs.available`) against configured thresholds — this is fundamentally a `df`-style check on the filesystem backing `/var/lib/kubelet` (and `/var/lib/containerd` or `/var/lib/docker` for images), not a per-pod `du` walk.

**Step 3 — SSH into the node and confirm with df:**

```
$ df -h /var/lib/kubelet /var/lib/containerd
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1   97G   92G  1.1G  99% /
```

Both paths share the root filesystem here, and it's essentially full.

**Step 4 — Try du on the obvious suspects:**

```
$ sudo du -h --max-depth=1 /var/lib/containerd | sort -rh
41G     /var/lib/containerd
28G     /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs
9G      /var/lib/containerd/io.containerd.content.v1.content
3G      /var/lib/containerd/io.containerd.metadata.v1.bolt
```

41G accounted for, but `df` said 92G used out of 97G. Same gap as Walkthrough 1 — something is holding space that `du` can't see from this path alone.

**Step 5 — Check for stopped containers and dangling images the kubelet GC hasn't cleaned up yet:**

```
$ sudo crictl ps -a | grep -i exited
CONTAINER    IMAGE                CREATED       STATE     NAME
a1b2c3d4e5   worker-app:v42       2 hours ago   Exited    worker-app
f6g7h8i9j0   worker-app:v41       6 hours ago   Exited    worker-app
...
(47 more Exited containers)

$ sudo crictl images | wc -l
83
```

47 exited containers and 83 cached images on one node is a strong signal that image/container garbage collection either isn't running frequently enough or the thresholds are set too loose for how fast this node churns through deployments.

**Step 6 — Check for the deleted-but-open-file pattern too, since it applies here as well:**

```
$ sudo lsof +L1 | grep -i deleted
containerd-shim 20481 root  5w  REG 259,1  22G  0  481029 /var/log/pods/default_api-7d9f6-xk2pl_.../0.log (deleted)
```

A crashlooping pod had its log rotated/deleted by the container runtime's log rotation, but the shim process handling that pod's logs was still holding the old file open — 22G worth. A crashlooping pod restarting rapidly and logging heavily is exactly the kind of workload that produces this.

**Step 7 — Remediate:**

```
$ sudo crictl rmp $(sudo crictl ps -a -q --state Exited)      # remove stopped containers
$ sudo crictl rmi --prune                                     # remove unused images
```

And separately, fix the crashlooping pod so it stops generating 22G of logs before it gets rescheduled — the disk space fix is a symptom treatment, not the root cause.

**Step 8 — Confirm and prevent recurrence:**

```
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1   97G   38G  56G  40% /
```

To stop this from recurring, tighten the kubelet's garbage collection settings (`--image-gc-high-threshold`, `--image-gc-low-threshold`) or the eviction thresholds (`evictionHard.nodefs.available`) so cleanup kicks in earlier, and make sure log rotation for container logs (`--container-log-max-size`, `--container-log-max-files`) is configured sanely so a single crashlooping pod can't repeat this.

### Outro

[#outro](#outro)

- If `df` and `du` disagree, don't assume one is "wrong". They're answering different questions, and the gap itself is usually a clue about what's going on (most often: something still has a deleted file open).
- Always check `df -i` too. A "disk full" symptom can sometimes actually be "out of inodes", and `du` won't tell you that at all.
- In container/Kubernetes environments, remember that a directory you can see may not map 1:1 to the filesystem `df` is reporting on, and that the kubelet's disk-pressure decisions are `df`-style checks, not `du`-style ones. Confirm mount points before comparing numbers.
- The same `lsof +L1` trick that solves a plain Linux "phantom disk usage" mystery works identically inside a Kubernetes node's shell — it's the same OS underneath.

### Tips

[#tips](#tips)

- Quick one-liner to find the top 10 largest directories from current location:

    `du -h --max-depth=1 . | sort -rh | head -n 10`

- To watch disk usage live while debugging a fast-filling disk:

    `watch -n 5 df -h`

- `ncdu` (if installed) is a much friendlier interactive version of `du` for hunting down large directories, worth installing on any box you SSH into regularly:

    `apt-get install -y ncdu` (Ubuntu/Debian)

    `dnf install -y ncdu` (Red Hat/Fedora)

- On a Kubernetes node, `crictl` is your `du`/`df` equivalent for container runtime state — `crictl ps -a`, `crictl images`, and `crictl info` (look at the `status.conditions` for `DiskPressure` directly from the kubelet's own view) are the first three commands worth running before digging into raw filesystem paths.
