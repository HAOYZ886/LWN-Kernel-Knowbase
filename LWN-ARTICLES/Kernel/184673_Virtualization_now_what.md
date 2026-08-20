---
title: "Virtualization: now what?"
url: https://lwn.net/Articles/184673/
date: "May 22, 2006"
category: "Virtualization-Containers"
author: 
---

Serge Hallyn recently posted [a new version of the UTS namespaces patch](<http://lwn.net/Articles/184383/>). This code, a small part of the "lightweight virtualization" or "containers" concept, allows various bits of system naming information (the stuff which can be seen with `uname`, essentially) to differ between sets of processes on the same system. It may not seem like a big thing, but, as a piece of container technology which has received the approval of several projects working in this area, it gives a hint of how the larger problem might be solved. 

Andrew Morton responded with [a note praising the way the work has been done](<https://lwn.net/Articles/184674/>), but asking a fundamental question: 

Generally, I think that the whole approach of virtualising the OS so it can run multiple independent instances of userspace is a good one. It's an extension and a strengthening of things which Linux is already doing and it pushes further along a path we've been taking for many years. If done right, it's even possible that each of these featurettes could improve the kernel in its own right - better layering, separation, etc. [...] 

All of which begs the question "now what?". 

The worry is that the kernel developers could merge a large amount of non-trivial code, make a number of internal kernel interfaces more complicated, and still not have an end result that is useful to the containers community. The fact that the developers working in this area were able to agree on a patch for UTS namespaces is encouraging, but it is not a guarantee that consensus will be reached on the more complicated changes. The possibility of an intractable disagreement derailing the whole process partway through is a real one. 

On the other hand, keeping all of the container code out of the kernel until it is reasonably complete has its own costs. Some of the container changes look to be relatively large and intrusive. Maintaining them all out of the tree would not be a great deal of fun. Neither would merging the whole mess at some future point when enough developers can agree that they are "done." 

There are a number of features needed by the projects concerned with virtualization and containers. They include: 

  * The UTS namespace patch mentioned above. 

  * [PID virtualization](<http://lwn.net/Articles/171017/>), isolating each group of processes on the system from each other, and allowing process IDs to be reused between containers. 

  * Namespaces for SYSV interprocess communication primitives (semaphores, shared memory, and message queues). 

  * [Time virtualization](<http://lwn.net/Articles/180375/>), so that each container can have its own idea of what time it is. 

  * Virtualization of user and group ID values. 

  * Network namespaces, intended to give each container a specific set of network interfaces to which it has access. When used in conjunction with IP aliases, this feature can set up a separate IP address for each container and keep containers from accessing each others' traffic. 

The ability to virtualize the view of the filesystem through namespaces is also required, but Linux has had that capability for some years now. Some of the more advanced container capabilities - live checkpointing and process migration, for example - will require yet another set of deep kernel hooks. 

Most container concepts need most of the items from the list above to be able to provide useful isolation. So, somehow, a path must be found to get those features into the kernel without running into a blocking disagreement partway through - assuming that container support is considered desirable in general, of course. 

Andrey Savochkin came up with [a proposal](<https://lwn.net/Articles/184682/>) which could be a good step forward: implement the network namespaces feature first. It is one of the most complex features, and it must be implemented in a way which doesn't upset the highly refined sensibilities of the networking subsystem developers. Some fairly tricky side problems - such as virtualizing access to `/proc` and sysfs - will have to be solved in the process. All told, it may be the hardest part of the problem, and it may be the place where an extended disagreement is most likely to show up. 

Often, developers like to take on the easier parts of a problem first, then apply any lessons learned to the harder parts. In this case, however, starting with the hardest part may make some sense. If no universally acceptable solution can be found, the idea of generalized container support in the kernel can be dropped before too much other code has been merged. If, instead, the developers involved are able to implement something which pleases (or, at least, does not mortally offend) everybody, they should be able to get over any other roadblocks which may show up later on. In that case, the various pieces of the puzzle could be merged with confidence as they become ready.
