---
title: Running distributions in containers
url: https://lwn.net/Articles/462669/
date: "October 12, 2011"
category: "Containers; Virtualization-Containers"
author: "By Jake Edge October 12, 2011"
---

By **Jake Edge**  
October 12, 2011

One of the requests in the recently posted "[Plumber's Wish List](<https://lwn.net/Articles/462076/>)" was for a way for a process to reliably detect that it isn't in the root PID namespace (i.e. is in a "container", at least by some definition). That wish sparked an interesting discussion on linux-kernel about the nature of containers and what people might use them for. Some would like to be able to run standard Linux distributions inside a container, but others are not so sure that is a useful goal. 

A container is a way to isolate a group of processes from the rest of a running Linux system. By using namespaces, that group can have its own private view of the OS--though, crucially, sharing the same kernel with whatever else is running--with its own PID space, filesystems, networking devices, and so on. Containers are, in some ways, conceptually similar to virtualization, with the separate vs. shared kernel being the obvious user-visible difference between the two. But there are straightforward ways to detect that you are running under virtualization and that is not true for containers/namespaces. 

Lennart Poettering--one of the wishing plumbers--[outlined](<https://lwn.net/Articles/462725/>) the need for detecting whether a process is running in a child PID namespace: 

To make a standard distribution run nicely in a Linux container you usually have to make quite a number of modifications to it and disable certain things from the boot process. Ideally however, one could simply boot the same image on a real machine and in a container and would just do the right thing, fully stateless. And for that you need to be able to detect containers, and currently you can't. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

He goes on to list a number of different things that are not "virtualized" by namespaces, including sysfs, `/proc/sys`, SELinux, udev, and more. Standard Linux distributions currently assume that they have full control of the system and the init process will do a wide variety of unpleasant things when it runs inside a container. Distributions could make use of a reliable way of detecting containerization to avoid (or change) actions with effects outside the container. Poettering went on to point out that ""having a way to detect execution in a container is a minimum requirement to get general purpose distribution makers to officially support and care for execution in container environments"". 

Eric W. Biederman, who is one of the namespace developers, [agreed](<https://lwn.net/Articles/462741/>) with the idea: ""I agree getting to the point where we can run a standard distribution unmodified in a container sounds like a reasonable goal."" He suggested two possible solutions for a straightforward detection scheme (either putting a file into the container's root directory or modifying the output of `uname`), but also started looking at all of the different areas that will need to be addressed to make it possible to run distributions inside containers. Much of that depends on finishing up the work on [user (i.e. UID) namespaces](<https://lwn.net/Articles/420624/>). 

But Ted Ts'o is a bit [skeptical](<https://lwn.net/Articles/462743/>) of the need to run full distributions inside a container. The advantage that containers have over virtual machines (VMs) is that they are lighter weight, he said, and adding multiple copies of system services (he mentions udev and D-Bus) starts to remove that advantage. He wonders if it makes more sense to just use a VM: 

If you end up [with] so much overhead to provide the desired security and/or performance isolation, then it becomes fair to ask the question whether you might as well pay a tad bit more and get even better security and isolation by using a VM solution.... 

In a [second message](<https://lwn.net/Articles/462756/>), Ts'o expands on his thinking, particularly regarding security. He is not optimistic about using containers that way: ""given that kernel is shared, trying to use containers to provide better security isolation between mutually suspicious users is hopeless"". The likelihood that an "isolated" user can find a local privilege escalation is just too high, and that will allow the user to escape the container and compromise the system as a whole. He is concerned that adding in more kernel complexity to allow distributions to run unchanged in containers may be wasted effort: 

So if you want that kind of security isolation, you shouldn't be using containers in the first place. You should be using KVM or Xen, and then only after spending a huge amount of effort fuzz testing the KVM/Xen paravirtualization interfaces. So at least in my mind, adding vast amounts of complexities to try to provide security isolation via containers is really not worth it. 

Biederman, though, [thinks](<https://lwn.net/Articles/462775/>) that there are situations where it would be convenient to be able to run distribution images ""just like I find it [convenient] to loopback mount an iso image to see what is on a disk image"". But, firing up KVM to run the distribution may be just as easy, and works today, as Ts'o [pointed out](<https://lwn.net/Articles/462788/>). There are more platforms out there than just those that KVM supports, however, so Biederman [believes](<https://lwn.net/Articles/462789/>) there is a place for supporting containerized distributions: 

You can test a lot more logical machines interacting with containers than you can with vms. And you can test on all the [architectures] and platforms linux supports not just the handful that are well supported by hardware virtualization. 

In the end, Biederman is not convinced that there is a ""good reason to have a design that doesn't allow you to run a full userspace"". He also notes that with the current implementation of containers (i.e. without UID namespaces), all users in the container are the same as their counterparts outside the container, and that includes the root user. Adding UID namespaces would allow a container to partition its users from those of the "external" system, so that root inside the container can't make changes that affect the entire system: 

With user namespaces what we get is that the global root user is not the container root user and we have been working our way through the permission checks in the kernel to ensure we get them right in the context of the user namespace. This trivially means that the things that we allow the global root user to do in /proc/ and /sysfs and the like simply won't be allowed as a container root user. Which makes doing something stupid and affecting other people much more difficult. 

UID namespaces are still a ways out, Biederman said, so problems with global sysctl settings from within containers can still cause weirdness, but ""once the user namespaces are in place accessing a truly global sysctl will result in EPERM when you are in a container and everyone will be happy. ;)"". There are some interesting implications of UID namespaces that may eventually need to be addressed, he said, including persistent UIDs in filesystems: 

So once we have all of the permission checks in the kernel tweaked to care about user namespaces we next look at the filesystems. The easy initial implementation is going to be just associating a user namespace with a super block. But farther out being able to store uids from different user namespaces on the same filesystem becomes an interesting problem. 

We already have things like user mapping in 9p and nfsv4 so it isn't wholly uncharted territory. But it could get interesting. 

Interesting indeed. One might wonder whether there will be some pushback from other kernel hackers about adding mapping layers to filesystems (presumably in the VFS code so that it works for all of them). Since virtualization can solve many of the problems that are still being worked on in containers (at least for some hardware platforms), there may be questions about adding further kernel complexity to support full-scale containerization as envisioned by Biederman (and others). That is essentially the argument that Ts'o is making, and one might guess that others have similar feelings. 

In any case, no patches have yet appeared for detecting that a process is running in a container, but it may not require any changes to the kernel. Poettering [mentioned](<https://lwn.net/Articles/462800/>) that LXC containers set an environment variable that processes can use for that purpose, and Biederman seemed to think that might be a reasonable solution (and wouldn't require kernel changes as it is just a user-space convention). Making a new UTS namespace (and changing the output of `uname`) as Biederman suggested would be another way to handle the problem from user space. That part seems like it will get solved in short order, but the more general questions of containers and security isolation are likely to be with us for some time to come.
