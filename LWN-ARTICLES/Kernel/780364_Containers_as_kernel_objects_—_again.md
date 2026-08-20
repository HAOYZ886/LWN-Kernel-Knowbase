---
title: Containers as kernel objects — again
url: https://lwn.net/Articles/780364/
date: "February 22, 2019"
category: Containers
author: "By Jonathan Corbet February 22, 2019"
---

By **Jonathan Corbet**  
February 22, 2019

Linus Torvalds once famously [said](</2001/1206/a/no-design.php3>) that there is no design behind the Linux kernel. That may be true, but there are still some guiding principles behind the evolution of the kernel; one of those, to date, has been that the kernel does not recognize "containers" as objects in their own right. Instead, the kernel provides the necessary low-level features, such as namespaces and control groups, to allow user space to create its own container abstraction. This refusal to dictate the nature of containers has led to a diverse variety of container models and a lot of experimentation. But that doesn't stop those who would still like to see the kernel recognize containers as first-class kernel-supported objects. 

One of those people is David Howells, who has posted [a patch set](<https://lwn.net/ml/linux-kernel/155024683432.21651.14153938339749694146.stgit@warthog.procyon.org.uk/>) designed to stir up this debate. It starts by defining a container as a kernel object that has a set of namespaces, a root directory, and a set of processes running inside it, one of which is deemed to be the "init" process. Creating one of these containers would be done by calling one of a set of new system calls: 

```
int container_create(const char *name, unsigned int flags);
```

The provided `name` can appear in a new `/proc/containers` file, but does not otherwise seem to be used much. The `flags`, instead, control which namespaces should be inherited from the creator and which should be created for the container as part of this call. Other flags include `CONTAINER_KILL_ON_CLOSE` to cause the container to be killed if the returned file descriptor is closed, and `CONTAINER_NEW_EMPTY_FS_NS` to cause the container to be created with a new mount namespace containing no filesystems at all. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The return value is a file descriptor used to manipulate the container, as described below. A `poll()` call on this descriptor will indicate `POLLHUP` should the last process in the container die. 

On creation, though, the container contains no processes; that is changed with a call to: 

```
pid_t fork_into_container(int container_fd);
```

This call is like `fork()`, with the exception that the new child process will be running inside the indicated container as its init process. There can be only one init process, so `fork_into_container()` will only work once for any given container. 

A socket can be created inside a container (from the outside) by calling: 

```
int container_socket(int container_fd, int domain, int type, int protocol);
```

It is also possible to mount filesystems inside the container from the outside using Howells's [proposed new filesystem mounting API](<https://lwn.net/Articles/759499/>), which has also [been updated](<https://lwn.net/ml/linux-fsdevel/155059610368.17079.2220554006494174417.stgit@warthog.procyon.org.uk/>) recently. A call to `fsopen()` would create a superblock as usual; the `fsconfig()` call could then be used to indicate that the superblock should exist inside the container object. This allows a container's filesystem tree to be constructed from outside, which is undoubtedly a useful feature; it eliminates the need to perform privileged mounting operations from inside the container itself. The `mount()` and (proposed) `move_mount()` calls could also be used to move a mounted filesystem into a container. 

The "*`at()`" series of system calls (such as [`openat()`](<http://man7.org/linux/man-pages/man2/open.2.html>)) allows the provision of a file descriptor to indicate where the search for a given pathname should start. Howells's patch set extends this functionality by allowing a file descriptor representing a container to be passed into these calls; the result would be to start the search at the container's root directory. 

Finally, there is also a mechanism by which key-management "upcalls" (wherein the kernel calls out to user space to request that a cryptographic key be provided) can be intercepted by the container creator. That, along with the addition of a separate keyring to each container, gives the creator control over which keys are seen (and can be used) by processes inside the container. The intended use case here is to allow containers to make use of keys to authenticate access to remote filesystems. 

If all of this has a bit of a familiar ring to it, that should not be surprising; Howells [proposed much of this functionality](<https://lwn.net/Articles/723561/>) back in 2017. He ran into a fair amount of opposition at the time to the idea of adding a container concept to the kernel, and this work disappeared from view. Now that it has returned, many of the same objections are being raised. James Bottomley, for example, [said](<https://lwn.net/ml/linux-kernel/1550432358.2809.21.camel@HansenPartnership.com/>): 

I thought we got agreement years ago that containers don't exist in Linux as a single entity: they're currently a collection of cgroups and namespaces some of which may and some of which may not be local to the entity the orchestration system thinks of as a "container". 

Howells, however, [is unimpressed](<https://lwn.net/ml/linux-kernel/19562.1550617574@warthog.procyon.org.uk/>) by this complaint: ""I wasn't party to that agreement and don't feel particularly bound by it"". In a world where there is no overall design behind the kernel such a position may be tenable, but it's not, on its own, a particularly compelling argument for why the status quo should be changed in such a significant way. It is also not the best way to win friends, which could be helpful; as things stand, some of the [rejections](<https://lwn.net/ml/linux-kernel/8736ojybw7.fsf@xmission.com/>) of this work have been less than entirely amicable. 

One could perhaps make an argument that the lack of a proper container object was necessary in the early days, when the community was still trying to figure out how containers should work in general. Now that we have several years of experience and a set of emerging container-related standards, perhaps the time has come to codify some of what has been understood into kernel features that make containers as a whole easier to deal with. This argument has not yet been made, though, with the result that the status quo has a high likelihood of winning out here. We may yet get a formal container abstraction in the kernel, but this version of this patch set seems unlikely to be it.
