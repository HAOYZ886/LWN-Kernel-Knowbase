---
title: "A call to reconsider address-space isolation"
url: https://lwn.net/Articles/909469/
date: "September 29, 2022"
category: "Memory management-Address-space isolation; Security-Meltdown and Spectre"
author: "By Jonathan Corbet September 29, 2022 LPC"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 29, 2022

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

When the kernel is running, it has access to its entire address space — usually including all of physical memory — even if only a small portion of that address space is actually needed. That increases the kernel's vulnerability to speculative attacks. An [address-space isolation patch set](<https://lwn.net/ml/linux-kernel/20220223052223.1202152-1-junaids@google.com/>) aiming to change this situation has been circulating for a few years, but has never been seriously considered for merging into the mainline. At the [2022 Linux Plumbers Conference](<https://lpc.events/>), Ofir Weisse sought to convince the development community to reconsider address-space isolation. 

Weisse began by pointing out that there seems to be a steady supply of new speculative-execution attacks that need to be mitigated; "[Retbleed](<https://lwn.net/Articles/900917/>)" is just one of the latest examples. The performance costs of mitigations for these vulnerabilities can be high, to the point that a lot of companies are simply not using them. The cost is also high in terms of development time, with each new variant requiring months of work to address. 

Address-space isolation (ASI) is the technique of unmapping memory that is not immediately needed, making it inaccessible to the current running context. Speculative-execution attacks cannot target memory that is not mapped, so the contents of unmapped memory can no longer be exfiltrated via such an attack. One example of ASI is [kernel page-table isolation](<https://lwn.net/Articles/741878/>), which was adopted in response to the Meltdown vulnerability. There have been numerous proposals for using ASI in other contexts in recent years, but none have been merged. The specific proposal under discussion in this session is meant to protect hosts against hostile virtual machines. 

[![\[Ofir Weisse\]](https://static.lwn.net/images/conf/2022/lpc/OfirWeisse-sm.png)](<https://lwn.net/Articles/909473/>) Wider use of ASI in the kernel would eliminate much of the work of mitigating speculative vulnerabilities, Weisse said. It would reduce the task of addressing a new vulnerability to "three-to-ten lines of code by a single engineer" with no new performance impact. The "new" in that claim is important, though; the ASI patch set itself has a performance impact of 2-14%, depending on which benchmark is run. There is room for improvement in those numbers, though, he said. 

The patch set (which is described in more detail in [this article](<https://lwn.net/Articles/886494/>)) is "a bitter pill" to swallow, he continued. It is large and requires significant changes to the memory-management subsystem and to many calls to allocation functions like `kmalloc()`. In short, ASI depends on the marking "sensitive" parts of memory that should be shielded from speculative-execution attacks; those are the portions that are unmapped when the isolation is in effect. That means new GFP flags for memory allocations, similar flags for slab creation and `vmalloc()`, and new annotations for local and global variables. Doing a complete job would require checking each allocation and declaration site and determining whether the memory involved is sensitive or not. 

When the kernel hands control to a virtual machine, it first calls `asi_enter()` to unmap all of the memory that has been marked as being sensitive, making that memory inaccessible to speculative attacks. When that virtual machine exits back into the host kernel, that memory will initially remain unmapped while the host kernel processes the request from the virtual machine. Many of the reasons for a virtual-machine exit can be handled without access to the sensitive memory, he said. In such cases, the request will be handled and control will return to the virtual machine without ever mapping the sensitive memory. 

Sometimes, though, there will be a need to access sensitive memory. An important observation here, Weisse said, is that speculative execution will never cause a page fault. So, if the kernel faults while trying to do something with sensitive memory, the access is known to not be speculative; the kernel responds by mapping the sensitive ranges and continuing execution. If simultaneous multi-threading (SMT) is in use, any sibling CPUs will be "stunned" (forced idle) before mapping that memory as an additional defense. On return to the virtual machine, that memory will be unmapped again and sibling CPUs will be resumed. 

The key to making this mechanism work well is determining which memory should be classified as being sensitive. Increasing the number of requests that can be handled without mapping sensitive memory will improve both performance and security. This is being done by running various workloads of interest and looking that the percentage of virtual-machine exit events that require mapping sensitive memory; ideally it should be low. In cases where it is not, the task is to look at the memory that is being accessed and determine whether it is sensitive or not. In the latter case, the allocation site can be changed to mark the memory accordingly. 

Weisse concluded by saying that ASI could make it easier to address the speculative-execution vulnerabilities that are sure to come in the future with a performance cost that is far less than that imposed by current mitigations. The development community should, he said, reconsider swallowing the bitter pill. 

Dave Hansen asked whether ASI could be extended to work more generally on bare-metal systems, rather than being specific to the KVM interface. Weisse answered that it should be possible, but that there would be a lot more work involved to get to that point. 

Christian Brauner asked why the patch set had been rejected in the past. Junaid Shahid, who posted the most recent version of this work, said that there hasn't been any real opposition to the idea, but neither has there been much interest in getting it merged. Hansen said that he didn't like it because it is a large amount of code for a fairly narrow use case; it doesn't address the system-call path at all. The need to determine the sensitivity of memory would impose a large maintenance burden in every corner of the kernel, he added. 

The session came to a close without any real conclusions on the future of this work. Unless a wave of enthusiasm for ASI materializes from some direction, it seems likely to languish outside of the mainline indefinitely. The pill, it seems, remains too bitter for most developers. 

[Thanks to LWN subscribers for supporting my travel to this event.]
