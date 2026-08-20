---
title: "Kernel Summit 2005: Convergence of network and storage paths"
url: https://lwn.net/Articles/144273/
date: "July 20, 2005"
category: "Block layer; Memory management-Conference sessions"
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
Once upon a time, the networking and block subsystems were entirely separate from each other. The venerable NFS protocol brought the two closer together, but current protocols (such as iSCSI) are truly blurring this once-rigid distinction. Is iSCSI a network protocol which happens to carry SCSI commands, or a block transport which happens to run over a network interface? Either way, the convergence of these two areas is creating difficulties. Roland Dreier led a session which looked at this issue. 

The worst of these problems has been [covered before](<http://lwn.net/Articles/129703/>) in LWN. When memory gets tight, the system must be able to write dirty pages to their backing store. When that writing involves a network link, however, it is often necessary to allocate more memory to free memory. This situation can lead quickly to deadlocks, where the system is unable to free memory and continue getting work done. Block devices can have similar problems, but things are far worse in the networking world; a network-based block device must be able to transmit _and receive_ data while somehow dealing with unrelated packets on the same interfaces. 

To make things worse, some protocols might even require help from user space if connections are lost and must be re-established. 

There was some talk of ways to approach this problem, but Linus had to simply put his foot down: this problem is hard, and people have been trying to solve it for decades. Rather than continuing to beat our heads against the wall, wouldn't it just be better to tell people to buy a local disk? Diskless systems were shown to be a bad idea back in the 1980's; why repeat the same mistakes 20 years later? 

The simple answer to that question is that people running 4,000-node clusters do not wish to spend their lives replacing failing disk drives. Linus does recognize the issue, and is resigned to the fact that work will go into trying to make this sort of hardware work reliably. But he thinks the community should also push back and recommend that people use better, safer alternatives. 

The classic solution to this sort of problem is to set aside memory for emergency use. The block layer uses memory pools for this purpose, but, with straight block hardware, it is easy to know how much memory is needed to be able to make reasonable forward progress in all situations. When network-based storage is involved, there is no easy answer to that question. Setting aside one half of memory would probably solve the problem, but that is a cost that few users are willing to pay. 

The _real_ solution is to realize that, in the end, this is a virtual memory problem. If the VM subsystem could throttle a process before it manages to dirty the bulk of the memory in the system, this sort of memory pressure would not arise. But that is easier said than done: the VM subsystem is not normally notified when a process dirties a page; that happens all the time, and involving the kernel would slow things down greatly. That said, one can envision schemes where the system operates normally until it notices that a significant portion of its pages are dirty. At that point, the remaining clean pages could be write protected and the system would go into a defensive mode. Whenever a process faults on a write-protected page, it could be forced to sleep if the system needs to catch up on its memory reclamation work. The performance penalty could be significant, but the performance of a deadlocked system is even worse. 

One other potential problem which was raised was kernel stack usage. If you have a filesystem involving some pathological combination of NFS, cluster filesystems, the device mapper, iSCSI, IPsec, and more, a "simple" filesystem operation could end up calling deeply into the kernel. There was no real discussion of this issue, however.
