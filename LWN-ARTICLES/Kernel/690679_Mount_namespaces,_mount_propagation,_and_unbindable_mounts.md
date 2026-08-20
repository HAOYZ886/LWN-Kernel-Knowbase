---
title: "Mount namespaces, mount propagation, and unbindable mounts"
url: https://lwn.net/Articles/690679/
date: "June 15, 2016"
category: "Bind mounts; Namespaces-Mount namespaces; Shared subtrees"
author: "June 15, 2016 This article was contributed by Michael Kerrisk . Namespaces in operation"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

June 15, 2016

This article was contributed by [Michael Kerrisk](<http://man7.org/>).

* * *

[Namespaces in operation](<https://lwn.net/Articles/531114/#series_index>)

In [the previous installment](<https://lwn.net/Articles/689856/>) of our article series on namespaces, we looked at the key concepts underlying mount namespaces and shared subtrees, including the notions of mount propagation types and peer groups. In this article, we provide some practical demonstrations of the operation of the various propagation types: `MS_SHARED`, `MS_PRIVATE`, `MS_SLAVE`, and `MS_UNBINDABLE`. 

#### `MS_SHARED` and `MS_PRIVATE` example

As we saw in the previous article, the `MS_SHARED` and `MS_PRIVATE` propagation types are roughly opposites. A shared mount point is a member of peer group. Each of the member mount points in a peer group propagates mount and unmount events to the other members of the group. By contrast, a private mount point is not a member of a peer group; it neither propagates events to peers, nor receives events propagated from peers. In the following shell session, we demonstrate the different semantics of these two propagation types. 

Suppose that, in the initial mount namespace, we have two existing mount points, `/mntS` and `/mntP`. From a shell in the namespace, we then mark `/mntS` as shared and `/mntP` as private, and view the mounts in `/proc/self/mountinfo`: 

```
sh1# **mount --make-shared /mntS**
        sh1# **mount --make-private /mntP**
        sh1# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        77 61 8:17 / /mntS rw,relatime shared:1
        83 61 8:15 / /mntP rw,relatime
```

From the output, we see that `/mntS` is a shared mount in peer group 1, and that `/mntP` has no optional tags, indicating that it is a private mount. (As noted in the previous article, most mount and unmount operations require that the user is privileged, as indicated by the '#' prompt.) 

On a second terminal, we create a new mount namespace where we run a second shell and inspect the mounts: 

```
sh2# **unshare -m --propagation unchanged sh**
        sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        222 145 8:17 / /mntS rw,relatime shared:1
        225 145 8:15 / /mntP rw,relatime
```

The new mount namespace received a copy of the initial mount namespace's mount points. These new mount points maintain the same propagation types, but have unique mount IDs (first field in the records). 

In the second terminal, we then create mounts under each of `/mntS` and `/mntP` and inspect the outcome: 

```
sh2# **mkdir /mntS/a**
        sh2# **mount /dev/sdb6 /mntS/a**
        sh2# **mkdir /mntP/b**
        sh2# **mount /dev/sdb7 /mntP/b**
        sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        222 145 8:17 / /mntS rw,relatime shared:1
        225 145 8:15 / /mntP rw,relatime
        178 222 8:22 / /mntS/a rw,relatime shared:2
        230 225 8:23 / /mntP/b rw,relatime
```

From the above, it can be seen that `/mntS/a` was created as shared (inheriting this setting from its parent mount) and `/mntP/b` was created as a private mount. 

Returning to the first terminal and inspecting the set-up, we see that the new mount created under the shared mount point `/mntS` propagated to its peer mount (in the initial mount namespace), but the new mount created under the private mount point `/mntP` did not propagate: 

```
sh1# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        77 61 8:17 / /mntS rw,relatime shared:1
        83 61 8:15 / /mntP rw,relatime
        179 77 8:22 / /mntS/a rw,relatime shared:2
```

#### `MS_SLAVE` example

Making a mount point a slave allows it to receive propagated mount and unmount events from a master peer group, while preventing it from propagating events to that master. This is useful if we want to (say) receive a mount event when an optical disk is mounted in the master peer group (in another mount namespace), but we want to prevent mount and unmount events under the slave mount from having side effects in other namespaces. 

We can demonstrate the effect of slaving by first marking two (existing) mount points in the initial mount namespace as shared: 

```
sh1# **mount --make-shared /mntX**
        sh1# **mount --make-shared /mntY**
        sh1# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        132 83 8:23 / /mntX rw,relatime shared:1
        133 83 8:22 / /mntY rw,relatime shared:2
```

On a second terminal, we create a new mount namespace and inspect the replicated mount points: 

```
sh2# **unshare -m --propagation unchanged sh**
        sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        168 167 8:23 / /mntX rw,relatime shared:1
        169 167 8:22 / /mntY rw,relatime shared:2
```

In the new mount namespace, we then mark one of the mount points as a slave. The effect of changing a shared mount to a slave mount is to make it a slave of the peer group of which it was formerly a member. 

```
sh2# **mount --make-slave /mntY**
        sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        168 167 8:23 / /mntX rw,relatime shared:1
        169 167 8:22 / /mntY rw,relatime master:2
```

In the above output, the `/mntY` mount point is marked with the tag `master:2`. The tag name is perhaps counterintuitive: it indicates that the mount point is a _slave_ mount that is receiving propagation events from the master peer group with the ID 2. In the case where a mount is both a slave of another peer group, and shares events with a peer group of its own, then the optional fields in the `/proc/PID/mountinfo` record will show both a `master:M` tag and a `shared:N` tag. 

Continuing in the new namespace, we create mounts under each of `/mntX` and `/mntY`: 

```
sh2# **mkdir /mntX/a**
        sh2# **mount /dev/sda3 /mntX/a**
        sh2# **mkdir /mntY/b**
        sh2# **mount /dev/sda5 /mntY/b**
```

When we inspect the state of the mount points in the new mount namespace, we see that `/mntX/a` was created as a new shared mount (inheriting the "shared" setting from its parent mount) and `/mntY/b` was created as a private mount (i.e., no tags shown in the optional fields): 

```
sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        168 167 8:23 / /mntX rw,relatime shared:1
        169 167 8:22 / /mntY rw,relatime master:2
        173 168 8:3 / /mntX/a rw,relatime shared:3
        175 169 8:5 / /mntY/b rw,relatime
```

Returning to the first terminal, we see that the mount `/mntX/a` propagated to the `/mntX` peer in the initial namespace, but the mount `/mntY/b` did not propagate: 

```
sh1# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        132 83 8:23 / /mntX rw,relatime shared:1
        133 83 8:22 / /mntY rw,relatime shared:2
        174 132 8:3 / /mntX/a rw,relatime shared:3
```

Next, we create a new mount point under `/mntY` in the initial mount namespace: 

```
sh1# **mkdir /mntY/c**
        sh1# **mount /dev/sda1 /mntY/c**
        sh1# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        132 83 8:23 / /mntX rw,relatime shared:1
        133 83 8:22 / /mntY rw,relatime shared:2
        174 132 8:3 / /mntX/a rw,relatime shared:3
        178 133 8:1 / /mntY/c rw,relatime shared:4
```

When we examine the mount points in the second mount namespace, we see that in this case the new mount has been propagated to the slave mount point, and that the new mount is itself a slave mount (to peer group 4): 

```
sh2# **cat /proc/self/mountinfo | grep '/mnt' | sed 's/ - .*//'**
        168 167 8:23 / /mntX rw,relatime shared:1
        169 167 8:22 / /mntY rw,relatime master:2
        173 168 8:3 / /mntX/a rw,relatime shared:3
        175 169 8:5 / /mntY/b rw,relatime
        179 169 8:1 / /mntY/c rw,relatime master:4
```

#### An aside: bind mounts

In a moment, we'll look at the use of the `MS_UNBINDABLE` propagation type. However, beforehand, it's useful to briefly describe the concept of a bind mount, a feature that first appeared in Linux 2.4. 

A bind mount can be used to make a file or directory subtree visible at another location in the single directory hierarchy. In some ways, a bind mount is like a hard link, but it differs in some important respects: 

  * It is not possible to create a hard link to a directory, but it is possible to bind mount to a directory. 

  * Hard links can be made only to files on the same filesystem, while a bind mount can cross filesystem boundaries (and even reach out of a `chroot()` jail). 

  * Hard links entail a modification to the filesystem. By contrast, a bind mount is a record in the mount list of a mount namespace--in other words, a property of the live system. 

A bind mount can be created programmatically using the `mount()` `MS_BIND` flag or on the command line using `mount --bind`. In the following example, we first create a directory containing a file and then bind mount that directory at a new location: 

```
# **mkdir dir1**                 # Create source directory
        # **touch dir1/x**               # Populate the directory
        # **mkdir dir2**                 # Create target for bind mount
        # **mount --bind dir1 dir2**     # Create bind mount
        # **ls dir2**                    # Bind mount has same content
        x
```

Then we create a file under the new mount point and observe that the new file is visible under the original directory as well, indicating that the bind mount refers to the same directory object: 

```
# **touch dir2/y**
        # **ls dir1**
        x  y
```

By default, when creating a bind mount of a directory, only that directory is mounted at the new location; if there are any mounts under that directory tree, they are not replicated under the mount target. It is also possible to perform a _recursive bind mount_ , by calling `mount()` with the flags `MS_BIND` and `MS_REC`, or from the command line using the `mount --rbind` option. In this case, each mount under the source tree is replicated at the corresponding location in the target tree. 

#### `MS_UNBINDABLE` example

The shared, private, and slave propagation types are about managing propagation of mount events between peer mounts (which are typically in different namespaces). Unbindable mounts exist to solve a different problem, one that preceded the existence of mount namespaces. That problem is the so-called "mount point explosion" that occurs when repeatedly performing recursive bind mounts of a higher-level subtree at a lower-level mount point. We'll now walk through a shell session that demonstrates the problem, and then see how unbindable mounts provide a solution. 

To begin with, suppose we have a system with the two mount points, as follows: 

```
# **mount | awk '{print $1, $2, $3}'**
        /dev/sda1 on /
        /dev/sdb6 on /mntX
```

Now suppose that we want to recursively bind mount the root directory under several users' home directories. We'll do this for the first user and inspect the mount points. However, we first create a new namespace in which we recursively mark all mount points as slaves, to prevent the steps that we perform from having any side effects in other mount namespaces: 

```
# **unshare -m sh**
        # **mount --make-rslave /**
        # **mount --rbind / /home/cecilia**
        # **mount | awk '{print $1, $2, $3}'**
        /dev/sda1 on /
        /dev/sdb6 on /mntX
        /dev/sda1 on /home/cecilia
        /dev/sdb6 on /home/cecilia/mntX
```

When we repeat the recursive bind operation for the second user, we start to see the explosion problem: 

```
# **mount --rbind / /home/henry**
        # **mount | awk '{print $1, $2, $3}'**
        /dev/sda1 on /
        /dev/sdb6 on /mntX
        /dev/sda1 on /home/cecilia
        /dev/sdb6 on /home/cecilia/mntX
        /dev/sda1 on /home/henry
        /dev/sdb6 on /home/henry/mntX
        /dev/sda1 on /home/henry/home/cecilia
        /dev/sdb6 on /home/henry/home/cecilia/mntX
```

Under `/home/henry`, we have not only recursively added the `/mntX` mount, but also the recursive mount of that directory under `/home/cecilia` that was created in the previous step. Upon repeating the step for a third user and simply counting the resulting mounts, it becomes obvious that the explosion is exponential in nature: 

```
# **mount --rbind / /home/otto**
        # **mount | awk '{print $1, $2, $3}' | wc -l**
        16
```

We can avoid this mount explosion problem by making each of the new mounts unbindable. The effect of doing this is that recursive bind mounts of the root directory will not replicate the unbindable mounts. Returning to the original scenario, we make an unbindable mount for the first user and examine the mount via `/proc/self/mountinfo`: 

```
# **mount --rbind --make-unbindable / /home/cecilia**
        # **cat /proc/self/mountinfo | grep /home/cecilia | sed 's/ - .*//'**
        108 83 8:2 / /home/cecilia rw,relatime unbindable
        ...
```

An unbindable mount is shown with the tag `unbindable` in the optional fields of the `/proc/self/mountinfo` record. 

Now we create unbindable recursive bind mounts for the other two users: 

```
# **mount --rbind --make-unbindable / /home/henry**
        # **mount --rbind --make-unbindable / /home/otto**
```

Upon examining the list of mount points, we see that there has been no explosion of mount points, because the unbindable mounts were not replicated under each user's directory: 

```
# **mount | awk '{print $1, $2, $3}'**
        /dev/sda1 on /
        /dev/sdb6 on /mntX
        /dev/sda1 on /home/cecilia
        /dev/sdb6 on /home/cecilia/mntX
        /dev/sda1 on /home/henry
        /dev/sdb6 on /home/henry/mntX
        /dev/sda1 on /home/otto
        /dev/sdb6 on /home/otto/mntX
```

#### Concluding remarks

Mount namespaces, in conjunction with the shared subtrees feature, are a powerful and flexible tool for creating per-user and per-container filesystem trees. They are also a surprisingly complex feature, and we have tried to unravel some of that complexity in this article. However, there are actually several more topics that we haven't considered. For example, there are detailed rules that describe the propagation type that results when performing bind mounts and move (`mount --move`) operations, as well as rules that describe the result when changing the propagation type of a mount. Many of those details can be found in the kernel source file `[Documentation/filesystems/sharedsubtree.txt](<https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt>)`.
