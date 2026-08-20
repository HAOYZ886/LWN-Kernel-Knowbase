---
title: Moving physical pages from user space
url: https://lwn.net/Articles/944115/
date: "September 18, 2023"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet September 18, 2023"
---

By **Jonathan Corbet**  
September 18, 2023

Processes in a Linux system run within their own virtual address spaces. Their virtual addresses map to physical pages provided by the hardware, but the kernel takes pains to hide the physical addresses of those pages; processes normally have no way of knowing (and no need to know) where their memory is located in physical memory. As a result, the system calls for memory management also deal in virtual addresses. Gregory Price is currently trying to create an exception to this rule with [a proposal](<https://lwn.net/ml/linux-kernel/20230907075453.350554-1-gregory.price@memverge.com/>) for a new system call that would operate on memory using physical addresses. 

#### When physical placement matters

Most of the time, user space is entirely happy to let the kernel worry about where memory should be mapped; all physical pages are alike, so it really does not matter which ones are used by a given process. That situation can change, though, in situations where all physical pages are _not_ alike. Non-uniform memory-access (NUMA) machines are a case in point; these machines are split into multiple nodes, each of which normally contains one or more CPUs and some physical memory. For a process executing on a given node, memory attached to that same node will be faster than memory on other nodes, so the placement of memory matters. 

Kernel developers have been working on the NUMA problem for years, and have developed a number of mechanisms to try to keep processes and their memory together. System calls can be used to bind processes to specific nodes, to ask that memory be allocated on specific nodes, and to move pages of memory from one node to another when needed. There is always room for improvement, but NUMA systems work well most of the time. 

Hardware engineers are creative folks, though, and they have been busily working on other ways to create different types of memory. Contemporary systems still have traditional RAM, but memory might also be located on a peripheral device, in a [non-volatile RAM](<https://en.wikipedia.org/wiki/Non-volatile_random-access_memory>) array, in a bank of [high-bandwidth memory](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>), or in an external [CXL device](<https://lwn.net/Articles/894598/>). In each of those cases, the memory involved will have different performance characteristics than ordinary RAM, once again making the physical placement of a process's pages into an important concern. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Since the NUMA concept already exists and is able to represent different classes of memory, it has been extended to handle these newer memory types as well. Each bank of "different" memory is normally organized into its own CPU-less NUMA node. The existing system calls controlling memory allocation can then be used to locate pages within these special nodes. That solves the management problem at a low level, but it is really only the beginning. 

In many cases, the desired result for systems with multiple memory types is some form of memory tiering, where pages are migrated between memory types depending on how heavily they are used. Ideally, heavily used pages should be located in the system's fastest memory, while rarely used pages can be put out to pasture in slower memory. Finding an optimal way to move pages between memory tiers is [an area of active development](<https://lwn.net/Articles/931421/>), and a number of questions remain open. 

#### Tiering in user space

In this context, Price is seeking to add a new system call to allow some of those migration decisions to be made in user space. There are some existing interfaces that can be used to determine which physical pages are (or are not) in active use; devices providing memory can also, sometimes, provide this information. Using this data, a user-space management process could decide to move pages into the type of memory that is best suited to their current usage profile. 

There is a problem, though. That information, being tied to physical pages, lacks any connection to the processes using those pages. A user-space program wanting to force page migrations based on this information would first have to convert the physical page addresses into (process, virtual-address) pairs for use with the existing system calls. That is a non-trivial and expensive task. Price is looking for a way to move pages between memory types without the need for an awareness of which processes are using those pages. 

The result is a new system call, `move_phy_pages()`, that is patterned after the existing [`move_pages()`](<https://man7.org/linux/man-pages/man2/move_pages.2.html>) call (which uses virtual addresses); it is otherwise completely undocumented at this point. The interface appears to be: 

```
int move_phy_pages(unsigned long count, void **pages, int *nodes,
        		       int *status, int flags);
```

This call will attempt to move `count` pages, the physical addresses of which are stored in the `pages` array; each page will be moved to the NUMA node indicated by the appropriate entry in `nodes`. The `status` array will be filled in with information about what happened to each page; on success, the `status` entry will contain the page's new node number. The only relevant `flags` value appears to be `MPOL_MF_MOVE_ALL`, which instructs the call to move pages that are mapped by multiple processes; otherwise only singly mapped pages are moved. 

If the `nodes` array is `NULL`, the system call will, instead, just store the status of each of the indicated `pages` in `status`. There are limits to how useful that is, since the node number of physical pages is already described by their physical address and does not normally change over time. 

In reviewing the patch, Arnd Bergmann [questioned](<https://lwn.net/ml/linux-kernel/f73d0495-f575-4b97-bc74-a57bd427df98@app.fastmail.com/>) the use of the `void *` type for the `pages` array. The values provided there are not actually pointers that can be dereferenced in any context; instead, they are used by the kernel to obtain the page-frame numbers (PFNs) for the pages of interest. Since, in some 32-bit configurations, full physical addresses may not fit within a normal pointer type, Bergmann [suggested](<https://lwn.net/ml/linux-kernel/2fe03345-01a2-4cfe-9648-ae088493d1af@app.fastmail.com/>) using the `__u64` type instead. 

That conversation also raised the question of whether, instead, user space should be providing PFNs to `move_phy_pages()`. As Bergmann pointed out, there are no system calls that accept PFNs now, so that would be breaking new ground. That, though, reflects the fact that, until now, system calls do not normally deal with physical addresses at all. If this work goes forward, finding a consensus on the best way to refer to such addresses, for `move_phy_pages()` and anything that might follow — will be important. 

Whether this work will actually move forward remains to be seen. It is, almost by definition, an interface to move pages around without knowing which processes are using them; otherwise, `move_pages()` could be used instead. Perhaps the information regarding physical memory and its utilization that is available to user space (Price provided a list of information sources in [this message](<https://lwn.net/ml/linux-kernel/ZPrRcJCjRBvJ9c3N@memverge.com/>)) is sufficient to make useful decisions, but that would probably need to be demonstrated somehow. This patch provides access to functionality that is normally kept deeply within the memory-management subsystem; developers will want to see that the benefits it provides justify that intrusion.
