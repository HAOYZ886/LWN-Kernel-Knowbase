---
title: ID mapping for mounted filesystems
url: https://lwn.net/Articles/837566/
date: "November 19, 2020"
category: "Filesystems-Mounting; Namespaces-User namespaces"
author: "By Jonathan Corbet November 19, 2020"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 19, 2020

Almost every filesystem (excepting relics like VFAT) implements the concept of the owner and group of each file; the higher levels of the operating system then use that information to control access to those files. For decades, it has usually sufficed to track a single owner and group for each file, but there is an increasing number of use cases wanting to make that ownership relative to the environment any given process is running in. Developers have been working for a few years to find solutions to this problem; the latest attempt is the [ID-mapped mounts patch set](<https://lwn.net/ml/linux-fsdevel/20201115103718.298186-1-christian.brauner@ubuntu.com/>) from Christian Brauner. 

In truth, the ID-mapping problem is not exactly new. User and group IDs for files only make sense across a management domain if there is a single authority controlling the assignment of those IDs. Since that is often not the case, network filesystems like NFS have had the ability to remap IDs for many years. The growth of virtualization and container technologies has brought the problem closer to home; there can be multiple management domains running on a single machine. The NFS ID-remapping mechanism is of little use if NFS itself is not being used. 

For example, container runtime systems may want to provide a common root image to each container. [User namespaces](<https://lwn.net/Articles/532593/>) may be used to ensure that each container is running with a set of nonprivileged IDs on the host system, but those containers should be able to access their root images with root privileges. Mounting that image with ID remapping would make this possible. Similarly, ID remapping would make it easier to share filesystems between containers regardless of the IDs in use within each container. Or consider [systemd-homed](<https://systemd.io/HOME_DIRECTORY/>), which provides consistent access to a user's home directory across machines. If a user logs into a system and is given a user ID that doesn't match the ownership of their home directory, systemd-homed will change the ownership of all files in and below the home directory — not an especially efficient operation. ID remapping would solve the problem in a more satisfying way. 

There have been a number of previous attempts to address these use cases. The [shiftfs](<https://lwn.net/Articles/718639/>) filesystem was designed to be stacked on top of an ordinary filesystem; it would then remap user and group IDs in operations as they passed through. That idea then evolved into [shifting bind mounts](<https://lwn.net/Articles/809125/>), which moved the ID-mapping function into the virtual filesystem (VFS) layer. Shortly after that, Brauner proposed [FSID mappings](<https://lwn.net/Articles/812504/>), which repurposed the kernel's filesystem-ID abstraction to perform the remapping. Now, with ID-mapped mounts, the remapping is again handled within the VFS, but with a twist. 

This patch set adds a new pointer to the [`vfsmount`](<https://elixir.bootlin.com/linux/v5.9.8/source/include/linux/mount.h#L70>) structure that represents a mounted filesystem; this pointer, called `mnt_user_ns`, points to a user namespace. One of the key features of user namespaces is, of course, ID remapping; a process that is running within a user namespace will already have its user and group IDs remapped for any operation, including filesystem operations, that reaches outside of the namespace. But user namespaces have a single map that applies to all operations, and to all mounted filesystems; attaching a user namespace to the `vfsmount` structure allows every mounted filesystem to have a different mapping. 

Setting up ID-mapped mounts, thus, involves the creation of user namespaces to contain the ID-mapping tables. These user namespaces will, most likely, never have processes running within them; in a sense, much of their functionality is wasted in this context. But this approach made it possible to use all of the existing ID-mapping helpers, while creating a more focused ID-mapping abstraction would require duplicating much of that functionality. 

By default, mounted filesystems will point to the initial user namespace, which is taken as an indication that no remapping is to be done at that layer. Code that wants to add ID mapping to a mounted filesystem has to start by creating a new user namespace; this is a bit of a roundabout procedure that is not directly supported by the kernel. In [a sample `mount-idmapped` tool](<https://github.com/brauner/mount-idmapped/blob/master/mount-idmapped.c>) written by Brauner, this task is done by creating a new process within its own user namespace. The child process does nothing but suspend itself with a `SIGSTOP` signal while the parent creates a reference to the child's user namespace by opening the associated `/proc` file. 

The next step is to establish the ID mapping in the newly created user namespace; this is done by writing appropriate values to the `uid_map` and `gid_map` files in the child process's `/proc` directory. Once that has been done, the child can just be killed off; the open file descriptor to its user namespace will ensure that it will stay around after the process is gone. 

Actually associating the user namespace is done with the `mount_setattr()` system call, which is also added by this patch set: 

```
struct mount_attr {
    	__u64 attr_set;
    	__u64 attr_clr;
    	__u64 propagation;
    	__u64 userns_fd;
        };
    
        int mount_setattr(int dfd, const char *path, unsigned int flags,
        		      struct mount_attr *attr, size_t attr_size);
```

The `attr_set` and `attr_clr` fields of the `mount_attr` structure describe the attributes to be set and cleared, respectively; `propagation` controls whether this operation affects only the filesystem indicated by `dfd` and `path` or whether it also affects all filesystems currently mounted underneath it. To add ID mapping to a filesystem, the caller (who must have the `CAP_SYS_ADMIN` capability in the current patches) should set `MOUNT_ATTR_IDMAP` in `attr_set`, and set `userns_fd` to the file descriptor for the relevant user namespace. 

While ID mapping can apparently be set up for any filesystem mount, the feature is expected to be mostly used with bind mounts, which create a new view of an existing filesystem. The above-linked cover letter for the patch series gives a number of examples of how this capability could be used. A simple one involves just providing a view of a directory with the files owned by a different user ID. Another creates an identity mapping (so IDs don't change), but that mapping lacks user ID 0, preventing access as root. Filesystems without the concept of user IDs (such as VFAT) can have those IDs grafted onto them with ID-mapped mounts. And so on. 

The [previous posting](<https://lwn.net/ml/linux-fsdevel/20201029003252.2128653-1-christian.brauner@ubuntu.com/>) of this patch set generated a certain amount of interest. This work seems to have the approval of the VFS developers, which is a significant hurdle that any patch in this area must overcome. So it might just be that a solution to the ID-mapping problem has finally been found and there will be no need for yet another attempt — maybe.
