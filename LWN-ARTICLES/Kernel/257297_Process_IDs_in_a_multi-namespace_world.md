---
title: "Process IDs in a multi-namespace world"
url: https://lwn.net/Articles/257297/
date: "November 6, 2007"
category: "Containers; Development model-User-space ABI; Virtualization-Containers"
author: "By Jonathan Corbet November 6, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 6, 2007

[Last week's article on containers](<http://lwn.net/Articles/256389/>) discussed process ID namespaces. The purpose of these namespaces is to manage which processes are visible to a process inside a container. The heavy use of PIDs to identify processes has caused this particular patch to go through a long period of development before being merged for 2.6.24. It appears that there are some remaining issues, though, which could prevent this feature from being available in the next kernel release. As is often the case, the biggest problems come down to user-space API issues. 

On November 1, Ingo Molnar [pointed out](<https://lwn.net/Articles/257298/>) that [some questions raised by Ulrich Drepper](<http://www.nabble.com/Re%3A-question%3A-pid-space-semantics.-p3409990.html>) back in early 2006 remained unanswered. These questions all have to do with what happens when the use of a PID escapes the namespace that it belongs to. There are a number of kernel APIs related to interprocess communication and synchronization where this could happen. Realtime signals carry process ID information, as do SYSV message queues. At best, making these interfaces work properly across PID namespaces will require that the kernel perform magic PID translations whenever a PID crosses a namespace boundary. 

The biggest sticking point, though, would appear to be the robust futex mechanism, which uses PIDs to track which process owns a specific futex at any given time. One of the key points behind futexes is that the fast acquisition path (when there is no contention for the futex) does not require the kernel's involvement at all. But that acquisition path is also where the PID field is set. So there is no way to let the kernel perform magic PID translation without destroying the performance feature that was the motivation for futexes in the first place. 

Ingo, Ulrich, and others who are concerned about this problem would like to see the PID namespace feature completely disabled in the 2.6.24 release so that there will be time to come up with a proper solution. But it is not clear what form that solution would take, or if it is even necessary. 

The approach [seemingly favored by Ulrich](<https://lwn.net/Articles/257300/>) is to eliminate some of the fine-grained control that the kernel currently provides over the sharing of namespaces. With the 2.6.24-rc1 interface, a process which calls `clone()` can request that the child be placed into a new PID namespace, but that other namespaces (filesystems, for example, or networking) be shared. That, says Ulrich, is asking for trouble: 

This whole approach to allow switching on and off each of the namespaces is just wrong. Do it all or nothing, at least for the problematic ones like NEWPID. Having access to the same filesystem but using separate PID namespaces is simply not going to work. 

Coalescing a number of the namespace options into a single "new container" bit would help with the current shortage of clone bits. But it might well not succeed in solving the API issues. Even processes with different filesystem namespaces might be able to find the same futex via a file visible in both namespaces. The passing of credentials over Unix-domain sockets could throw in an interesting twist. And it would seem that there are other places where PIDs are used that nobody has really thought of yet. 

Another possible approach, one which hasn't really featured in the current debate, would be to create globally-unique PIDs which would work across namespaces. The current 32-bit PID value could be split into two fields, with the most significant bits indicating which namespace the PID (contained in the least significant bits) is defined in. Most of the time, only the low-order part of the PID would be needed; it would be interpreted relative to the current PID namespace. But, in places where it makes sense, the full, unique PID could be used. That would enable features like futexes to work across PID namespaces. 

There are still problems, of course. The whole point of PID namespaces is to completely hide processes which are outside of the current namespace; the creation and use of globally-unique PIDs pokes holes in that isolation. And there's sure to be some complications in the user-space API which prove to be hard to paper over. 

Then, there is the question of whether this problem is truly important or not. Linus [thinks not](<https://lwn.net/Articles/257303/>), pointing out that the sharing of PIDs across namespaces is analogous to the use of PIDs in lock files shared across a network. PID-based locking does not work on NFS-mounted files, and PID-based interfaces will not work between PID namespaces. Linus concludes: 

I don't understand how you can call this a "PID namespace design bug", when it clearly has nothing what-so-ever to do with pid namespaces, and everything to do with the *futexes* that blithely assume that pid's are unique and that made it part of the user-visible interface. 

One could argue that the conflict with PID namespaces was known when the robust futex feature was merged and that something could have been done at that time. But that does not really help anybody now. And, in any case, there are issues beyond futexes. 

PID namespaces are a significant complication of the user-space API; they redefine a basic value which has had a well-understood meaning since the early days of Unix. So it is not surprising that some interesting questions have come to light. Getting solid answers to nagging API questions has not always been the strongest point of the Linux development process, but things could always change. With luck and some effort, these questions can be worked through so that PID namespaces, when they become available, will have well-thought-out and well-defined semantics in all cases and will support the functionality that users need.
