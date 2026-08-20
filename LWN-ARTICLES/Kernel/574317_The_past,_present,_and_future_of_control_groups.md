---
title: "The past, present, and future of control groups"
url: https://lwn.net/Articles/574317/
date: "November 20, 2013"
category: Control groups
author: "By Jonathan Corbet November 20, 2013 Korea 
Linux Forum"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 20, 2013

* * *

[Korea Linux Forum](<https://lwn.net/Archives/ConferenceByYear/#2013-Korea_Linux_Forum>)

Much has been said about the problems surrounding control groups and the changes that will need to be made with this kernel subsystem. At the 2013 [Korea Linux Forum](<http://events.linuxfoundation.org/events/korea-linux-forum>), control group co-maintainer Tejun Heo provided a comprehensive overview of how we got into the current situation, what the problems are, and what is being done to fix them. 

The idea behind control groups is relatively simple: divide processes into a hierarchy of groups and use those groups to provision resources in the system. The reality has turned out to be rather messier. So, Tejun asked: how did we get to this point? To begin with, he said, much of what is being done with control groups is new; all of it is new to Linux in particular, and some is new in general. So the community did not have any sort of model to follow when designing this new feature. 

Beyond that, though, it is worth looking at who did the work. Control groups started as a new interface to the "cpuset" mechanism, which is used to partition the CPUs in a system among groups of processes. Few people, [![\[Tejun Heo\]](https://static.lwn.net/images/conf/2013/klf/TejunHeo1-sm.jpg)](<https://lwn.net/Articles/574326/>) Tejun said, cared much about this feature, which is used mostly by the high-performance computing crowd. So few kernel developers paid much attention to what was being done. 

Then control groups gained the memory controller, a mechanism for restricting the amount of memory used by each group. The core memory management developers did not really care about this work, so they did not participate in it and did not want to hear about it. The block controller came about the same way; Tejun does work in the block subsystem, but he had no real interest in the block controller and mostly just wanted it to stay out of his way. This environment led to a situation where controllers were written by developers without extensive experience in the subsystems they were working with; those controllers had to work on a non-interference basis so that the core developers could ignore them. As a result, controllers have been "bolted onto the side" of the existing kernel subsystems. 

The result, Tejun said, is "not pretty." Even worse, though, is that the barriers between the controllers and the subsystems they work with inevitably broke down over time. So control groups are, as a whole, isolated and poorly integrated with the rest of the kernel, but they still manage to complicate the development of the rest of the kernel. 

The developers who did all this work were good programmers, Tejun said, but they were not all that experienced with kernel development. So the code they produced was "kind of alien," not conforming to the usual coding style and practices. They repeated a lot of mistakes that the community has made and fixed in the past — repetition that could have been avoided with more review, but, he said, few people were paying active attention to the work being done in this area. 

#### Mistakes were made

What kinds of mistakes were made? Start with hierarchy support — or the lack thereof — in a number of controllers. Control groups allow the organization of processes into a true hierarchy, with policies applied at various levels in the tree. But making a truly hierarchical controller is hard, so a number of controller developers simply didn't bother; instead, they ignored the tree structure and treated every group as if it were placed directly under the root. This was not a good decision, Tejun said; if a controller could not be made hierarchical, it should have at least refused to work with nested control groups. That would have indicated to users that things wouldn't work as they might expect and avoided the creation of a non-hierarchical interface that must now be supported. 

The "`.use_hierarchy`" flag added by the memory controller to enable hierarchical behavior in subtrees was an especially confusing touch, he said. 

Another clear mistake was the "`release_agent`" mechanism. The idea was to notify some process when a control group becomes empty; it was a good idea, he said, in that it allows that process to clean up groups that are no longer in use. But it was implemented as a user-mode helper — every time a control group becomes empty, the kernel creates a new process to run the release agent program. This is an expensive and complex operation for the simple task of sending a notification to user space. The rest of the kernel had moved away from this kind of process-based notification years ago, but the control group developers reimplemented it. We have much better notification mechanisms that should have been used instead, but nobody who could have pointed that out was paying attention when this code was merged. 

Yet another problem is the heavy entanglement with the virtual filesystem (VFS) layer. Many years ago, the original sysfs implementation was also deeply tied to the VFS with the idea that it would simplify things. But that didn't work; the results were, instead, lots of memory used and locking-related [![\[Tejun Heo\]](https://static.lwn.net/images/conf/2013/klf/TejunHeo4-sm.jpg)](<https://lwn.net/Articles/574327/>) problems. So sysfs was reworked to look a lot like a distributed filesystem, and things have worked better ever since. When the control group developers set out to create their administrative filesystem, though, they repeated the sysfs mistake. So now control groups have a number of related problems, such as locking hassles whenever an operation needs to work across multiple groups. Tejun is now working on separating things properly; some of that work was merged for the 3.13 kernel. 

In engineering, Tejun said, nothing is free; everything comes down to a tradeoff between objectives. Or, in other words, "extremes suck," but control groups went to an extreme with regard to flexibility. Allowing multiple, independent hierarchies is the biggest example; this feature results in a situation where the kernel cannot tell which control group a given process belongs to. Instead, that membership is expressed by a list of arbitrary length. Controllers are all entirely separate from each other, with no coordination between them; they also behave in inconsistent ways. All this flexibility makes it difficult to make changes to the code, since there is no way to know what might break. 

Flexibility also led to a range of implementation issues beyond the lack of hierarchical support in some controllers. The core code is complex and fragile. Developers took a lot of shortcuts in areas like security, leading to problems like denial-of-service issues. But, perhaps worst of all, the kernel community committed to a new ABI for control groups without really thinking about it; as a result, we ended up with a lot of accidental features. The ability to assign a process's threads to different control groups is one of those — most controllers only make sense at the process level. The control interface is filesystem-based, but no thought went into permissions, so it is possible to change the ownership of subdirectories, essentially delegating ownership of a subtree of groups to another user. The control group developers have, for all practical purposes, created a new set of system calls without the kind of review that system calls must normally go through. 

#### What now?

The first step has been to fix the controllers that do not support the full control group hierarchy. Unfortunately, they cannot simply be fixed in place without breaking existing users. So there will have to be a "version 2" of the control interface that users can opt into. Controllers must be fully hierarchical or they will simply be unavailable in the new interface. The interface change will also allow the developers to enforce a certain degree of consistency between controllers. It will be possible, Tejun said, to mix use of the old and new interfaces without breaking things. 

The multiple control group hierarchies will be going away. Most users will not really notice the change, but some were using multiple hierarchies to avoid enabling expensive controllers for processes that don't need them. In the new scheme, that need will be met by making it possible to enable or disable specific controllers at any level of the hierarchy. But all controllers will see the same process hierarchy; among other things, that will make it possible for them to cooperate more effectively. The resulting system will not be as flexible as multiple hierarchies are, but there seems to be an emerging consensus that it will suffice for the known use cases out there. 

A lot of controllers will need updates to work in the new scheme, he said. There are a number of people working in the problem and the work is "70-80% there" at this point. 

There will be, Tejun said, "no more faking things that we can't do properly." That is especially true when it comes to security which, he said, is a matter of noting and dealing with all of the relevant details — something that has not been done in the control group subsystem. In particular, the whole concept of delegating subtrees of the control group hierarchy to untrusted users is "broken"; there is no way to prevent denial-of-service attacks (or worse) under that scenario. To allow users to move to the new API without breaking things, it will still be possible to do this kind of delegation by changing the ownership of control group directories, but, he said, it will not be secure, just like it is not secure now. 

A more secure approach might be the use of a trusted user-space agent process — something that is likely to be necessary in the future anyway. A number of these agents already exist: systemd is one, Google has its own, Ubuntu has one based on Google's code, and Android has an agent as well. In the Android case, Google actually has to "break the kernel" to make it work the way it wants. There is a need for some kind of common scheme so that processes can interoperate with any agent without having to know which one it is. 

Tejun had hoped to have a prototype implementation of a reworked control group subsystem available by about now, but that has not happened. It may be ready by the end of the year, with, hopefully, the work being complete around the middle of 2014. 

In summary, he repeated that control groups embody a lot of functionality that has not existed in Linux before. When he looks at the current code, he often gets angry at the mistakes that were made, but he is quite confident that he is making plenty of horrible mistakes of his own. So he fully expects future developers to be just as angry with him. That just goes with the territory. The important thing, he said, is to minimize the commitment that is made to user space; in that way, he hopes, we will not get locked into too many mistakes in the future. 

[Your editor thanks the Linux Foundation for travel assistance to attend the Korea Linux Forum.]
