---
title: "Kernel Summit: Clustered storage"
url: https://lwn.net/Articles/94559/
date: "July 21, 2004"
category: "Clusters-Filesystems; GFS"
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
Ken Preslen led a session on clustered storage and Linux. He named three ongoing projects: GFS (Red Hat), OCFS2 (Oracle) and Lustre. It was quickly pointed out that HP's CFS is still going strong as well. Others also certainly exist. 

Ken's interest is primarily in the GFS effort. This project has been restarted after Red Hat relicensed the code; it has been working on splitting out the various components of the GFS system. Thus, there are now separate pieces for the cluster manager (which decides who is in or out of the cluster), the distributed lock manager, a cluster configuration system (for synchronizing configuration information across the cluster), I/O fencing (keeping nodes which have fallen out of the cluster from writing to the filesystem), a clustered logical volume manager, a new network block I/O subsystem, a user-space application failover system, and, incidentally, the GFS filesystem code. 

The real point of contention, as far as the kernel developers are concerned, is whether certain pieces (the cluster and lock managers, in particular) really need to be in the kernel. These tasks have been handled in user space in the past, in other implementations. There are arguments for putting this code in the kernel, keeping latency to a minimum being one of them. But this discussion is not complete. 

It was also noted that, even if, say, the lock manager ends up in the kernel, there is no room for several different lock managers. Somehow the various cluster filesystem projects are going to have to make use of more common code and concentrate on what is truly unique. 

[>> Next: kexec and fast booting](<https://lwn.net/Articles/94560/>).
