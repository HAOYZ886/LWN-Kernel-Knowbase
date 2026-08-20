---
title: Containers and PID virtualization
url: https://lwn.net/Articles/168093/
date: "January 17, 2006"
category: "Containers; Virtualization-Containers"
author: 
---

The folks at IBM would like to add a "container" capability to the Linux kernel. Containers are a way of walling a group of processes off from the rest of the system; a process within a container will only see its fellow inmate processes and whatever resources are made accessible to that container. This feature has some obvious security-related applications. IBM's plans, evidently, also include the ability to pack up a container and move it to another physical host without disrupting the processes trapped inside. 

The patches which have been circulating so far fall short of the final plan, but they already disturb enough code to have attracted some skeptical criticism. In particular, the [34-part PID virtualization patch](<http://lwn.net/Articles/168068/>) creates a simple container type, and implements a separate process ID space within containers. But, as we'll see, doing even that much involves some significant kernel changes. 

The containers themselves are fairly simple. The patches create a virtual file called `/proc/container`. If a process writes a string to that file, a new container is created for that process, using the string as its name. The namespace is global, so every container on the system must have a unique name. Any child processes created by the newly-contained process will also be trapped within the container, with no way out. 

At this point, being inside a container does not affect a process's life that much. The one thing that does change, however, is that each container has its own process ID (PID) space. Processes within the container can only see others in the same container. There is nothing particularly controversial about that behavior, but the developers have another objective in mind: they want to be able to change the PIDs of contained processes without the processes themselves noticing. In particular, they would like to be able to migrate a container to a different system, which will certainly assign new PIDs to every process within the container. Code written for Unix-like systems does not normally expect its PID to change over time, however; so switching PIDs underneath a process could lead to all kinds of strange behavior. To avoid this problem, the plan is that PIDs remain constant within the container, even if those PIDs change in the real world. 

Implementing constant PIDs (from a viewpoint inside the container) is not a straightforward task; it involves adding a whole new virtualization layer inside the kernel. There are two types of PIDs now, "real" PIDs and the virtual PIDs used by contained processes. Any place in the kernel which deals with PID values must become aware of which type of PID it is using, and convert to the other type when necessary. So, as a general rule, any code which exchanges PIDs with user space must use the virtual variety, while PIDs handled within the kernel are real. 

The PID logic is complicated by a few little details, like: what happens when containers are nested? A process living within a container has a real PID and a virtual PID associated with the container. If that process creates a container of its own, it will acquire yet another PID associated with the new container. So it is not possible to simply convert a real PID to a virtual PID; such questions require a "context" so that the kernel knows _which_ virtual PID is wanted. 

The result of all this is that PID handling within the kernel changes significantly. Code which used to get the current process's PID with `current->pid` must now use `tsk_pid(current)` for the real PID, or `tsk_vpid(current)` for the virtual PID - and it must know which one it wants. In situations where more than one virtual PID might be appropriate, `tsk_vpid_ctx()` must be used to supply the context. Much of the patch set is concerned simply with making these conversions; for good measure, it also renames the `pid` field of `struct task_struct` to catch any code still trying to access it directly. 

Behind all of this is a concept called "pidspaces." The patch carves up the global PID space takes the upper 9 bits of the 32-bit PID value and puts the pidspace number there. A virtual PID as seen within a container is turned into a real kernel PID by stuffing the pidspace number in those upper bits. Since the contained processes only see virtual PIDs, they never see the pidspace number, and they will not notice if that number changes. 

All of this code seems to work, but there is a certain amount of opposition to merging it. As Alan Cox [put it](<https://lwn.net/Articles/168096/>): 

This is an obscure, weird piece of functionality for some special case usages most of which are going to be eliminated by Xen. I don't see the kernel side justification for it at all. 

The developers answer that the ability to checkpoint and restart process trees, possibly moving them in between, will be highly useful. Some other virtualization projects also require this capability - not everybody wants to use Xen. So the pressure for PID virtualization probably won't just go away. 

What might happen is that the hiding of `current->pid` might be taken out, greatly reducing the size of the patch. Another idea which has been floated is to eliminate, to the greatest degree possible, the use of PIDs within the kernel. Almost any in-kernel use of a PID can be replaced with a direct pointer to the task structure. If a PID eventually is reduced to little more than a process-identifying cookie used for communication with user space, it will be easier to virtualize without complicating large amounts of kernel code.
