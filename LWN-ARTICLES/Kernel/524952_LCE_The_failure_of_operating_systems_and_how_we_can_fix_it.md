---
title: "LCE: The failure of operating systems and how we can fix it"
url: https://lwn.net/Articles/524952/
date: "November 14, 2012"
category: "Containers; Virtualization-Containers"
author: "By Michael Kerrisk November 14, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk**  
November 14, 2012

The [abstract](<http://linuxconeurope2012.sched.org/event/bf1a2818e908e3a534164b52d5b85bf1?iframe=no&w=900&sidebar=yes&bg=no#.UKPuE3npvNA>) of Glauber Costa's talk at LinuxCon Europe 2012 started with the humorous note ""I once heard that hypervisors are the living proof of operating system's incompetence"". Glauber acknowledged that hypervisors have indeed provided a remedy for certain deficiencies in operating system design. But the goal of his talk was to point out that, for some cases, containers may be an even better remedy for those deficiencies. 

#### Operating systems and their limitations

Because he wanted to illustrate the limitations of _traditional_ UNIX systems that hypervisors and containers have been used to address, Glauber commenced with a recap of some operating system basics. 

In the early days of computing, a computer ran only a single program. The problem with that mode of operation is that valuable CPU time was wasted when the program was blocked because of I/O. So, Glauber noted ""whatever equivalent of Ingo Molnar existed back then wrote a scheduler"" in order that the CPU could be shared among processes; thus, CPU cycles were no longer wasted when one process blocked on I/O. 

A later step in the evolution of operating systems was the addition of virtual memory, so that (physical) memory could be more efficiently allocated to processes and each process could operate under the illusion that it had an isolated address space. 

However, nowadays we can see that the CPU scheduling and virtual memory abstractions have limitations. For example, suppose you start a browser or another program that uses a lot of memory. As a consequence, the operating system will likely start paging out memory from processes. However, because the operating system makes memory-management decisions at a global scope, typically employing a least recently used (LRU) algorithm, it can easily happen that excessive memory use by one process will cause _another_ process to suffer being paged out. 

There is an analogous problem with CPU scheduling. The kernel allocates CPU cycles globally across all processes on the system. Processes tend to use as much CPU as they can. There are mechanisms to influence or limit CPU usage, such as setting the nice value of a process to give it a relatively greater or lesser share of the CPU. But these tools are rather blunt. The problem is that while it is possible to control the priority of individual processes, modern applications employ _groups_ of processes to perform tasks. Thus, an application that creates more processes will receive a greater share of the CPU. In theory, it might be possible to address that problem by dynamically adjusting process priorities, but in practice this is too difficult, since processes may come and go quite quickly. 

The other side of the resource-allocation problem is denial-of-service attacks. With traditional UNIX systems, local denial-of-service attacks are relatively easy to perpetrate. As a first example, Glauber gave the following small script: 

```
$ while true; do mkdir x; cd x; done
```

This script will create a directory structure that is as deep as possible. Each subdirectory "x" will create a dentry (directory entry) that is pinned in non-reclaimable kernel memory. Such a script can potentially consume all available memory before filesystem quotas or other filesystem limits kick in, and, as a consequence, other processes will not receive service from the kernel because kernel memory has been exhausted. (One can monitor the amount of kernel memory being consumed by the above script via the `dentry` entry in `/proc/slabinfo`.) 

Fork bombs create a similar kind of problem that affects unrelated processes on the system. As Glauber noted, when an application abuses system resources in these ways, then it should be the application's problem, rather than being everyone's problem. 

#### Hypervisors

Hypervisors have been the traditional solution to the sorts of problems described above; they provide the resource isolation that is necessary to prevent those problems. 

By way of an example of a hypervisor, Glauber chose KVM. Under KVM, the Linux kernel is itself the hypervisor. That makes sense, Glauber said, because all of the resource isolation that should be done by the hypervisor is already done by the operating system. The hypervisor has a scheduler, as does the kernel. So the idea of KVM is to simply re-use the Linux kernel's scheduler to schedule virtual machines. The hypervisor has to manage memory, as does the kernel, and so on; everything that a hypervisor does is also part of the kernel's duties. 

There are many use cases for hypervisors. One is simple resource isolation, so that, for example, one can run a web server and a mail server on the same physical machine without having them interfere with one another. Another use case is to gather accurate service statistics. Thus, for example, the system manager may want to run `top` in order to obtain statistics about the mail server without seeing the effect of a database server on the same physical machine; placing the two servers in separate virtual machines allows such independent statistics gathering. 

Hypervisors can be useful in conjunction with network applications. Since each virtual machine has its own IP address and port number space, it is possible, for example, to run two different web servers that each use port 80 inside different virtual machines. Hypervisors can also be used to provide root privilege to a user on one particular virtual machine. That user can then do anything they want on that virtual machine, without any danger of damaging the host system. 

Finally, hypervisors can be used to run different versions of Linux on the same system, or even to run different operating systems (e.g., Linux and Windows) on the same physical machine. 

#### Containers

Glauber noted that all of the above use cases can be handled by hypervisors. But, what about containers? Hypervisors handle these use cases by running multiple kernel instances. But, he asked, shouldn't it be possible for a single kernel to satisfy many of these use cases? After all, the operating system was originally designed to solve resource-isolation problems. Why can't it go further and solve these other problems as well by providing the required isolation? 

From a theoretical perspective, Glauber asked, should it be possible for the operating system to ensure that excessive resource usage by one group of processes doesn't interfere with another group of processes? Should it be possible for a single kernel to provide resource-usage statistics for a logical group of processes? Likewise, should the kernel be able to allow multiple processes to transparently use port 80? Glauber noted that all of these things should be possible; there's no theoretical reason why an operating system couldn't support all of these resource-isolation use cases. It's simply that, historically, operating systems were not built with these requirements in mind. The only notable use case above that couldn't be satisfied is for a single kernel to run a _different_ kernel or operating system. 

The goal of containers is, of course, to add the missing pieces that allow a kernel to support all of the resource-isolation use cases, without the overhead and complexity of running multiple kernel instances. Over time, various patches have been made to the kernel to add support for isolation of various types of resources; further patches are planned to complete that work. Glauber noted that although all of those kernel changes were made with the goal of supporting containers, a number of other interesting uses had already been found (some of these were touched on later in the talk). 

Glauber then looked at some examples of the various resource-isolation features ("namespaces") that have been added to the kernel. Glauber's first example was [network namespaces](<https://lwn.net/Articles/219794/>). A network namespace provides a private view of the network for a group of processes. The namespace includes private network devices and IP addresses, so that each group of processes has its own port number space. Network namespaces also make packet filtering easier, since each group of processes has its own network device. 

Mount namespaces were one of the earliest namespaces added to the kernel. The idea is that a group of processes should see an isolated view of the filesystem. Before mount namespaces existed, some degree of isolation was provided by the `chroot()` system call, which could be used to limit a process (and its children) to a part of the filesystem hierarchy. However, the `chroot()` system call did not change the fact that the hierarchical relationship of the mounts in the filesystem was global to all processes. By contrast, mount namespaces allow different groups of processes to see different filesystem hierarchies. 

[User namespaces](<https://lwn.net/Articles/491310/>) provide isolation of the "user ID" resource. Thus, it is possible to create users that are visible only within a container. Most notably, user namespaces allow a container to have a user that has root privileges for operations inside the container without being privileged on the system as a whole. (There are various other namespaces in addition to those that Glauber discussed, such as the [PID](<https://lwn.net/Articles/259217/>), [UTS](<https://lwn.net/Articles/179345/>), and IPC namespaces. One or two of those namespaces were also mentioned later in the talk.) 

Control groups (cgroups) provide the other piece of infrastructure needed to implement containers. Glauber noted that cgroups have received [a rather negative response](<https://lwn.net/Articles/464288/>) from some kernel developers, but he thinks that somewhat misses the point: cgroups have some clear benefits. 

A cgroup is a logical grouping of processes that can be used for resource management in the kernel. Once a cgroup has been created, processes can be migrated in and out of the cgroup via a pseudo-filesystem API (details can be found in the kernel source file [`Documentation/cgroups/cgroups.txt`](<https://lwn.net/Articles/524935/>)). 

Resource usage within cgroups is managed by attaching controllers to a cgroup. Glauber briefly looked at two of these controllers. 

The CPU controller mechanism allows a system manager to control the percentage of CPU time given to a cgroup. The CPU controller can be used both to guarantee that a cgroup gets a guaranteed minimum percentage of CPU on the system, regardless of other load on the system, and also to set an upper limit on the amount of CPU time used by a cgroup, so that a rogue process can't consume all of the available CPU time. CPU scheduling is first of all done at the cgroup level, and then across the processes within each cgroup. As with some other controllers, CPU cgroups can be nested, so that the percentage of CPU time allocated to a top-level cgroup can be further subdivided across cgroups under that top-level cgroup. 

The [memory controller](<https://lwn.net/Articles/443241/>) mechanism can be used to limit the amount of memory that a process uses. If a rogue process runs over the limit set by the controller, the kernel will page out _that_ process, rather than some other process on the system. 

#### The current status of containers

It is possible to run production containers today, Glauber said, but not with the mainline kernel. Instead, one can use the modified kernel provided by the open source [OpenVZ](<http://wiki.openvz.org/Main_Page>) project that is supported by Parallels, the company where Glauber is employed. Over the years, the OpenVZ project has been working on upstreaming all of its changes to the mainline kernel. By now, much of that work has been done, but some still remains. Glauber hopes that within a couple of years (""I would love to say months, but let's get realistic"") it should be possible to run a full container solution on the mainline kernel. 

But, by now, it is already possible to run subsets of container functionality on the mainline kernel, so that some people's use cases can already be satisfied. For example, if you are interested in just CPU isolation, in order to limit the amount of CPU time used by a group of processes, that is already possible. Likewise, the network namespace is stable and well tested, and can be used to provide network isolation. 

However, Glauber said, some parts of the container infrastructure are still incomplete or need more testing. For example, fully functional user namespaces are quite difficult to implement. The current implementation is usable, but not yet complete, and consequently there are some limitations to its usage. Mount and PID namespaces are usable, but likewise still have some limitations. For example, it is not yet possible to migrate a process into an existing instance of either of those namespaces; that is a desirable feature for some applications. 

Glauber noted some of the kernel changes that are still yet to be merged to complete the container implementation. Kernel memory accounting is not yet merged; that feature is necessary to prevent exploits (such as the dentry example above) that consume excessive kernel memory. Patches to allow [kernel-memory shrinkers](<https://lwn.net/Articles/516531/>) to operate at the level of cgroups are still to be merged. Filesystem quotas that operate at the level of cgroups remain to implemented; thus, it is not yet possible to specify quota limits on a particular user inside a user namespace. 

There is already a wide range of tooling in place that makes use of container infrastructure, Glauber said. For example, the [libvirt](<http://libvirt.org/>) library makes it possible to start up an application in a container. The OpenVZ `vzctl` tool is used to manage full OpenVZ containers. It allows for rather sophisticated management of containers, so that it is possible to do things such as running containers using different Linux distributions on top of the same kernel. And ""love it or hate it, systemd uses a lot of the infrastructure"". The `unshare` command can be used to run a command in a separate namespace. Thus, for example, it is possible to fire up a program that operates in an independent mount namespace. 

Glauber's overall point is that containers can already be used to satisfy several of the use cases that have historically been served by hypervisors, with the advantages that containers don't require the creation of separate full-blown virtual machines and provide much finer granularity when controlling what is or is not shared between the processes inside the container and those outside the container. After many years of work, there is by now a lot of container infrastructure that is already useful. One can only hope that Glauber's "realistic" estimate of two years to complete the upstreaming of the remaining container patches proves accurate, so that complete container solutions can at last be run on top of the mainline kernel.
