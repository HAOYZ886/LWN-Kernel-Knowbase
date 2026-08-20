---
title: Cpusets and memory policies
url: https://lwn.net/Articles/717797/
date: "March 22, 2017"
category: Cpusets
author: "By Jonathan Corbet March 22, 2017 LSFMM 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017>)

"Cpusets" are a Linux kernel mechanism that enables control over which processors a given process is allowed to run on. The memory policy (or "mempolicy") mechanism, instead, gives control over how a process's memory is allocated across the nodes of a NUMA system. As Vlastimil Babka explained during the memory-management track of the 2017 Linux Storage, Filesystem, and Memory-Management Summit, these two mechanisms do not always play well together, with some surprising and unfortunate consequences. 

Cpusets are an administrator-controlled mechanism; unprivileged processes cannot normally change their CPU assignments. Mempolicies, instead, are under the control of the processes themselves. If both mechanisms are used together, one might logically expect that memory would be allocated on the set of nodes defined by the intersection of the cpuset and the mempolicy. If that intersection is empty, "then there is space for creativity". But that is not what actually happens. 

Babka put up [a pair of slides](<https://lwn.net/Articles/717801/>) showing what can happen. Imagine a process running on a four-node system; initially both the cpuset and the mempolicy are set to nodes zero and one. In this case, memory will be allocated from either of those two nodes, as one might expect. If the cpuset is changed to nodes one and two, the memory allocations will follow to those two nodes. But, if the cpuset is first reduced to a single node (node two in the example), then restored to the original zero and one, the result will be allocations from node zero only; the kernel will have lost track of the fact that the mempolicy called for both nodes to be used. 

This problem was understood and addressed in the 2.6.26 kernel through the addition of a couple of flags to [the `set_mempolicy()` system call](<http://man7.org/linux/man-pages/man2/set_mempolicy.2.html>). If the process sets its mempolicy with the `MPOL_F_STATIC_NODES` flag, that policy will not change when the cpuset is changed. `MPOL_F_RELATIVE_NODES`, instead, causes the policy to move along with cpuset changes while remembering the original policy, so it will never exhibit the single-node allocation behavior described above. 

What happens if there is no intersection between the cpuset and the mempolicy, as can happen, especially, with `MPOL_F_STATIC_NODES`? The answer is that it will allocate memory from the cpuset nodes. Kirill Shutemov suggested that perhaps allocations should fail instead in that circumstance, but that was deemed to be unfriendly behavior and an ABI break as well. It is better to allocate memory on the wrong node than to kill an otherwise working program, especially if that program did work on older kernels. In general, it was agreed, the `set_mempolicy()` interface is broken, but it is going to be hard to fix now. 

One serious problem with the current implementation is its behavior when the cpuset is being changed, forcing the mempolicy to be changed as well. There is a period of time during that change when an empty node list causes the kernel to conclude that it is out of memory. That can lead to spurious invocations of the out-of-memory killer, an outcome that tends to get a cold reception in the user community. 

Fixing that problem seems necessary and urgent. The mempolicy updates associated with cpuset changes have to be maintained, since the alternative is an ABI break. For the static case, the solution is straightforward, since the set of nodes will not change. In the relative case, instead, the remapping will need to be done on the fly; it is a solvable problem but the solution looks complex. There may be no workable fix for the default case. 

The discussion focused on the details of how a fix might work for the remainder of the session. It may involve moving the list of allowed memory zones back into the cpuset itself, which is how mempolicies were once implemented many years ago. The plan, as is so often the case, is to wait for a patch to appear and see how the solution looks at that time.
