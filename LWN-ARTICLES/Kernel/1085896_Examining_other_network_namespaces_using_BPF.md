---
title: Examining other network namespaces using BPF
url: https://lwn.net/Articles/1085896/
date: "August 5, 2026"
category: "BPF-Networking"
author: "By Daroc Alden August 5, 2026 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Daroc Alden**  
August 5, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Jordan Rife's work involves writing BPF programs for [ Cilium](<https://cilium.io/>) that interface with [ Kubernetes](<https://kubernetes.io/>) networking. As part of that work, he wants to enable BPF programs with appropriate permissions to iterate through the sockets of a different network namespace. He led a session about the idea at the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) where the BPF developers in attendance were quick to suggest a number of related alternatives. 

Socket-lb is a Cilium feature that uses the hooks for sockets associated with a given control group to balance connections across remote servers, while avoiding the per-packet overhead of network-address translation (NAT). In a setup that does not use socket-lb, a client might make a request to a frontend server that then uses NAT to route the connection transparently to a chosen backend server. Socket-lb improves on this by having the routing happen directly on the client device, still transparently to user space, but avoiding the extra network hop implied by NAT. In practice, socket-lb has some limitations, Rife said. If a selected backend goes away, for example, traffic will continue to be directed at that non-existent backend until Cilium cleans things up. Today, the software does that by using BPF iterators to walk through the available sockets and destroy them if needed. 

That works, but only within a specific network namespace, since the namespaces are isolated from one another and cannot see each other's sockets. In practice, this means that Cilium's user-space component must enter the namespace of each Kubernetes pod running on the host, scan its sockets, and then exit the namespace and move on to the next one. This adds significant overhead to the whole process that isn't really needed, since Cilium has administrative access to all of the network namespaces anyway. 

Worse, sockets aren't sorted by network namespace in the kernel, so every time Cilium's BPF program does its scan, it has to walk the entire hash table of sockets; the kernel filters sockets from other network namespaces before the BPF program sees them. So, Cilium ends up walking the list of every socket in the system once for each network namespace, which is inefficient. In his work, Rife said he had seen as many as 256 namespaces per computer. 

His proposed solution is simple: create a new iterator, usable by BPF programs with appropriate privileges, that iterates through all of the sockets on the system at once, even those from foreign namespaces. That way, Cilium would not need to change network namespaces at all; it could simply use a BPF program to periodically scan the whole system for stale socket-lb sockets. 

Jakub Sitnicki asked why Rife didn't keep a list of which sockets correspond to which backend directly, and then use those. BPF programs can store pointers to socket objects in [ socket maps](<https://www.kernel.org/doc/html/latest/bpf/map_sockmap.html>), which can be used to store weak references (pointers that don't increment the socket's reference count, and therefore don't prevent it from being closed). So, such a scheme wouldn't keep sockets alive after their natural deaths, he noted. Rife did try that in 2025, but making it work was more complicated than it really should have been. In current kernels, BPF programs cannot destroy sockets from the context of an iterator over a socket map, because they would need to acquire the socket's lock. Fixing that requires rewriting a bunch of locking logic around access to sockets. 

One workaround would be to add a sleepable variant of [ `bpf_sock_destroy()`](<https://elixir.bootlin.com/linux/v7.2-rc5/source/net/core/filter.c#L12653>) that would acquire the socket lock itself, which Martin Lau had suggested to him, he said. Ultimately, Rife had abandoned the idea, but he could come back to it if the whole-machine iterator he was proposing wasn't acceptable. Sitnicki wanted to know why the socket map iterator wasn't sleepable in the first place. Rife didn't know, although he speculated that there might be some challenges around the interaction of RCU and the socket lock. Lau clarified that many BPF iterators are sleepable, it is just the socket map in particular that has this problem. 

One of Rife's colleagues is working on tracking metadata for all sockets in a separate map, and that might also be a workaround. Yet another approach would be to make `bpf_sock_destroy()` itself work from the context of a socket map iterator, Rife said. That's hard to do without adding a reference count, and therefore some memory overhead, or changing the semantics of the iterator. But allowing sufficiently privileged BPF programs to iterate over all network namespaces is a much simpler solution all around, he thought. 

Daniel Borkmann mentioned that the BPF maintainers had discussed adding an iterator over network namespaces; perhaps references from that iterator could be passed to the socket map iterator as a trusted input, he suggested, which would let the existing socket map iterator work without requiring Cilium's user-space component to change network namespaces. Rife pointed out that would still end up walking all of the sockets on the machine multiple times, it would just do it from entirely within the kernel. 

Sitnicki and Lau had a brief side conversation about how the current code finds network namespace references, and potential improvements there. Rife agreed to look into it, and consult the networking maintainers, but observed that it still wouldn't solve his efficiency problem. 

John Fastabend noted that Cilium caches raw socket pointers in several places, because BPF socket maps are not expandable, ""which is obnoxious."" Pointers to sockets stored in BPF socket maps are automatically cleaned up when the socket is destroyed; socket pointers stored as opaque values in a BPF arena are much more convenient to store for tracking purposes, but aren't automatically cleaned up and can't be dereferenced. Sitnicki pointed out that storing socket pointers in BPF arenas wouldn't work well for Rife's use case. Fastabend replied that it doesn't work well for him either, but there are lots of reasons to want to stash sockets somewhere more flexible. He suggested changing BPF socket maps to be resizeable. 

It would be nice to be able to mount sockfs (a virtual filesystem that allows users to manage sockets, including permissions, using the filesystem API) and pin sockets that way, Sitnicki remarked. That would avoid the need for BPF maps entirely. Lau commented that someone had a patch in progress that does something tangentially related to that, but still thought that being able to iterate over everything, as Rife proposed, would be useful. 

Sitnicki asked whether Rife had considered obtaining a duplicate file descriptor for the UDP socket backend processes that might go away, holding onto them in a daemon, and using those references to clean them up when needed. Rife explained that he'd like to do the whole thing from BPF. Fastabend wanted to know whether this was also a problem for TCP sockets. Borkmann explained that the normal TCP reset mechanism would take care of things, but only after a long-enough timeout that some customers preferred the faster failover provided by socket-lb, so it was used for both UDP and TCP. 

At that point the session devolved into a discussion about what other applications could potentially use a resizeable socket map, before running out of time. Despite the many alternatives the assembled developers raised to Rife's idea, none seemed hostile to it, just interested in painting the bikeshed a different color. As of the beginning of August, there is not yet any accepted solution to Rife's problem.
