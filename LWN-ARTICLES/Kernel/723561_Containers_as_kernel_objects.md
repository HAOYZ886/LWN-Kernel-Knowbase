---
title: Containers as kernel objects
url: https://lwn.net/Articles/723561/
date: "May 23, 2017"
category: Containers
author: "By Jonathan Corbet May 23, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 23, 2017

The kernel has, over the years, gained comprehensive support for containers; that, in turn, has helped to drive the rapid growth of a number of containerization systems. Interestingly, though, the kernel itself has no concept of what a container is; it just provides a number of facilities that can be used in the creation of containers in user space. David Howells is trying to change that state of affairs with [a patch set](<https://lwn.net/Articles/723477/>) adding containers as a first-class kernel object, but the idea is proving to be a hard sell in the kernel community. 

Containers can be thought of as a form of lightweight virtualization. Processes running within a container have the illusion of running on an independent system but, in truth, many containers can be running simultaneously on the same host kernel. The container illusion is created using [namespaces](<https://lwn.net/Articles/531114/>), giving each container its own view of the network, the filesystem, and more, and [control groups](<https://lwn.net/Articles/604609/>), which isolate containers from each other and control resource usage. Security modules or seccomp can be used to further restrict what a container can do. The result is a mechanism that, like so many things in Linux, offers a great deal of flexibility at the cost of a fair amount of complexity. Setting up a container in a way that ensures it will stay contained is not a trivial task and, as we'll see, the lack of a container primitive also complicates things on the kernel side. 

#### Adding a container object

Howells's patch creates (or modifies) a set of system calls to make it possible for user space to manipulate containers. It all starts with: 

```
int container_create(const char *name, unsigned int flags);
```

This new system call creates a container with the given `name`. The `flags` mainly specify which namespaces from the caller should be replaced by new namespaces in the created container. For example, specifying `CONTAINER_NEW_USER_NS` will cause the container to be created with a new user namespace. The return value is a file descriptor that can be used to refer to the container. There are a couple of flags that indicate whether the container should be destroyed when the file descriptor is closed, and whether the descriptor should be closed if the calling process calls `exec()`. 

The container starts out empty, with no processes running within it; if it is created with a new mount namespace, there are no filesystems mounted inside it either. Two new system calls (`fsopen()` and `fsmount()`, added in [a separate patch set](<https://lwn.net/Articles/723476/>)) can be used to add filesystems to the container. The "at" versions of the file system calls (`openat()`, for example) can take a container file descriptor as the starting point, easing the creation of files inside the container. It is possible to open a socket within the container with: 

```
int container_socket(int container_fd, int domain, int type, int protocol);
```

The main purpose of `container_socket()` appears to be to make it easy to use netlink sockets to configure the container's networking from the outside. It can help an orchestration system avoid the need to run a process inside the container to do this configuration. 

When it comes time to start things running inside the container, a call can be made to: 

```
pid_t fork_into_container(int container_fd);
```

The new process created by this call will be the init process inside the given container, and will run inside the container's namespaces. It can only be called once for any given container. 

There are a number of things that, Howells said, could still be added to this mechanism. They include the ability to set a container's namespaces directly, support for the management of a container's control groups, the ability to suspend and restart a container, and more. But it is not clear that this work will progress far in its current form. 

#### A poor match?

A number of developers expressed concerns about this proposal, mostly focused on two issues: the proposed container object is not seen as a good match for how containers are actually used now, and it is seen as the wrong solution to a specific problem. On the first issue, the flexibility of the current mechanisms is seen by many as an advantage, one that they would rather not lose. Jessica Frazelle [said](<https://lwn.net/Articles/723562/>): 

Adding a container object seems a bit odd to me because there are so many different ways to make containers, aka not all namespaces are always used as well as not all cgroups, various LSM objects sometimes apply, mounts blah blah blah. The OCI spec was made to cover all these things so why a kernel object? 

Here, she was referring to the [runtime specification](<https://github.com/opencontainers/runtime-spec/blob/master/README.md>) from the Open Containers Initiative. James Bottomley was [more direct](<https://lwn.net/Articles/723563/>), saying that: 

This sounds like a step in the wrong direction: the strength of the current container interfaces in Linux is that people who set up containers don't have to agree what they look like. So I can set up a user namespace without a mount namespace or an architecture emulation container with only a mount namespace. 

He pointed out, in particular, an apparent mismatch between the proposed container object and the concepts of containers and "pods" implemented in Kubernetes. Some namespaces are specific to a container, while others are shared across a pod, blurring the boundaries somewhat. The kernel container object, he [added](<https://lwn.net/Articles/723564/>), ""isn't something the orchestration systems would find usable"". 

Eric Biederman took [an even stronger position](<https://lwn.net/Articles/723565/>) by rejecting the patch outright. As he put it: 

Embracing the complexity of namespaces head on tends to mean all of the goofy scary semantic corner cases are visible from the first version of the design, and so developers can't take short cuts that result in buggy kernel code that persists for decades. I am rather tired of finding and fixing those. 

Unlike the others, he is not so deeply concerned with what existing orchestration systems do; his worries have to do with the exposing of the container object to user space at all. That is where the second issue comes up. 

#### Upcalls

To a great extent, it appears that the motivation behind this patch set isn't to make the management of containers easier for user-space code. Instead, it is trying to solve a nagging problem that has become increasingly irritating for kernel developers: how to make kernel "upcalls" work properly in a containerized environment. 

As a general rule, the kernel, as the lowest level of the system, tries to be self-sufficient in all things. There really is nobody else to rely on to get things done, after all. There are times, though, when the kernel has to ask user space for help. That is typically done with a call to `call_usermodehelper()`, an internal function that will create a user-space process and run a specific program to get something done — "calling up" to user space, in other words. 

There are a number of `call_usermodehelper()` call sites in the kernel. Some of the tasks it is used for include: 

  * The core-dump code can use it to invoke a program to do something useful with the dumped data. 
  * The NFSv4 client can call a helper program to perform DNS resolution. 
  * The module loader can invoke a helper to perform demand-loading of modules. 
  * The kernel's key-management code will call to user space when a key is needed to perform a specific function — to mount an encrypted filesystem, for example. 

Once upon a time, when life was simple, these upcalls would create a process running as root that could run the requested program. Now, however, the action that provoked the upcall in the first place may well have come from inside a container, and it may well be that the upcall should run within that container as well. At least, it should run inside that container's particular mix of namespaces. But, since the kernel has no concept of a container, it has no way to know which container to run any particular upcall within. A kernel upcall that is run in the wrong namespace might do the wrong thing — or allow a process to escape its container. 

Adding a container concept to the kernel is one way to fix this problem. But this particular patch has raised questions of whether (1) a container object is the best solution to the upcall problem, and (2) if a container object does make sense, does it need to be exposed to user space? The kernel might be able to keep track of the proper namespaces to use for specific upcalls without creating a bunch of new infrastructure or exposing a new API that would have to be maintained forever. Biederman [suggested](<https://lwn.net/Articles/723575/>) one possible scheme that could be used to track namespaces for the key-management upcalls, for example. 

Another possible approach, [proposed](<https://lwn.net/Articles/723576/>) by Colin Walters, is to drop the upcall model entirely. Instead, a protocol could be created to report events to a user-space daemon that could act on them in the proper context. That kind of change has been made in the past; device-related events were once handled via upcalls, but now they are communicated directly to the `udev` (or `systemd-udevd`) process instead. But, as Jeff Layton [pointed out](<https://lwn.net/Articles/723577/>), that model only works in some settings. In others, it just leads to a proliferation of daemon processes that clutter up the system and can create reliability issues. So the events model isn't necessarily a replacement for all kernel upcalls. 

This discussion is young as of this writing, and may yet progress in unexpected directions. From the early indications, it seems relatively unlikely that a container object visible to user space will be added to the kernel anytime soon. If, perhaps, some future attempt creates a container concept that is useful to existing orchestration systems, that could change. Meanwhile, we may well see an attempt to improve the kernel's internal ability to determine the proper namespace for any given upcall. Either way, the inherent complexity of the container problem seems likely to be with us for a long time.
