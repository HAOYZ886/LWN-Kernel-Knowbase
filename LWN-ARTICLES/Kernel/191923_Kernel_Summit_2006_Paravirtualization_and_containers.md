---
title: "Kernel Summit 2006: Paravirtualization and containers"
url: https://lwn.net/Articles/191923/
date: "July 19, 2006"
category: Containers; Virtualization; paravirt ops
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Virtualization remains an area of high interest in the development community. The "paravirtualization" mode, as epitomized by the Xen project, is still getting the lion's share of the attention. With paravirtualization, an i386-like architecture is defined, and the kernel is ported to that architecture. Virtualized systems are then run on that architecture with a hypervisor standing by to handle privileged requests. 

An increasing part of the mind share is being taken by "lightweight virtualization" or "container" approaches, however. With containers, virtualized systems run directly on the host kernel, contained within various walls which give those systems the appearance of having the computer to themselves. Containers have the advantage of being much cheaper; far more containerized systems than paravirtualized systems can be run on the same host. 

### Paravirtualization

The Kernel Summit featured two sessions devoted to these topics, with paravirtualization coming first. Xen hacker Keir Fraser started out with a discussion of Xen, which, he says, is the only free paravirtualization solution out there. Work on getting the Xen patches ready for merging continues, with a new patch set having been posted on the day this session was held. The bulk of these patches define the new architecture type, while a smaller subset is dedicated to providing I/O services to paravirtualized systems. 

The biggest sticking point came up early in the discussion. Despite the claim that Xen is the only free system out there, the kernel developers (and certain proprietary virtualization vendors) have a strong interest in supporting more than one paravirtualization solution. It would be nice to have only one set of virtualization hooks in the client-side kernel, and it would be nice if that kernel could run, unmodified, under more than just one type of hypervisor. 

One solution to this problem is the [VMI interface](<http://lwn.net/Articles/175706/>) proposed by the folks at VMWare; Zachary Amsden was there to promote this approach. VMI abstracts out the system-specific operations, allowing them to be filled in at run time. The way these operations are filled in, however, is not particularly popular: it involves injecting a binary "hypervisor ROM" into the client system. The kernel developers are not enthusiastic about adding hooks for the addition of binary code, so this idea has met resistance. 

The alternative is to use some sort of impedance-matching layer which is loaded like a shared library. Rusty Russell has a proposal called "paravirt_ops" which takes this approach; it involves no binary code blobs. The consensus at the meeting seemed to be that this approach was the right way forward, so that is how things are likely to go. The only question seems to be whether Xen should be merged first, then evolved toward paravirt_ops, or not; there was little enthusiasm, however, for merging an approach which is destined to be ripped out before being shipped to anybody. 

The problem remains, however, that nailing down the paravirtualization API will be a bit of a challenge. It is early in the game, and there are still a number of lessons yet to be learned in this area. So, while something may well find its way into the kernel before too long, it should be expected to be fluid for a while. There doesn't seem to be much of a sense of urgency, however, in nailing things down; the target time frame appears to be a year or so from now, when Novell and Red Hat will be pulling together their next-generation enterprise distributions. As long as things are in shape by then, most of the people involved should be happy. 

### Containers

The containers session was less contentious; the (numerous) players involved seem determined to work together, so it's mostly a matter of finding the best solutions to the problems. Those problems are, in essence, finding the best way to turn a large number of global namespaces into private namespaces which can be different from one container to the next. There is a large number of these namespaces, including the filesystem hierarchy, process IDs, resource limits, network interfaces, and more. Patches for many of these have been circulated (and covered on LWN); it's mostly a matter of getting them into good shape and convincing the rest of the world that they are worth merging. 

One open question is whether the kernel needs an explicit container concept which would pull together all those private namespaces. Adding that type might make it easier to stay on top of a heavily containerized system, but it might also make it harder to provide fine-grained control over which namespaces are shared and which are not. 

A big problem is finding a solution for `/proc` and `/sys`. These virtual filesystems are global namespaces with no concept of multiple views. Filtering invisible processes out of `/proc` would be relatively easy, but the other files there (including everything in `/proc/sys`) are harder. Providing separate versions of these filesystems looks, in general, to be a painful task. 

It was suggested that processes within containers might simply not see the bulk of `/proc` at all. That might require changes in a few system applications, but, when the developers were asked if they thought whether requiring modified distributions to run within containers was a problem, nobody spoke up. It was even suggested that `/proc`-free containers could be the path by which much old `/proc` cruft is cleaned up for the world as a whole. 

Finally, there was some concern that containers might prove to be a useful tool for rootkit writers. With a bit of effort, a rootkit could put everybody within a container and, thus, easily hide itself. How this problem will be solved is not entirely clear; one part of the solution may be providing an unambiguous way for a process to determine whether it is running within a container or not. 

  * [Next: Automated testing](<https://lwn.net/Articles/191924/>).
