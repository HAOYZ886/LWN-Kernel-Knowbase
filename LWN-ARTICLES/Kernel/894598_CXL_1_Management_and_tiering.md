---
title: "CXL 1: Management and tiering"
url: https://lwn.net/Articles/894598/
date: "May 13, 2022"
category: "Compute Express Link CXL; Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 13, 2022 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 13, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

[Compute Express Link](<https://www.computeexpresslink.org/about-cxl>) (CXL) is an upcoming memory technology that is clearly on the minds of Linux memory-management developers; there were five sessions dedicated to the topic at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). The first three sessions, on May 3, covered various aspects of memory management in the presence of CXL. It seems that CXL may bring some welcome capabilities, especially for cloud-service providers, but that will come at the cost of some headaches on the kernel-development side. 

At its core, CXL is a new way to connect memory to a CPU. That memory need not be on the local memory bus; indeed, it is likely to be located on a different device entirely. CXL vendors seemingly envision "memory appliances" that can provide memory to multiple systems in a flexible manner. Supporting CXL raises a number of interesting issues around system boot, memory hotplug, memory tiering, and more. 

#### A CXL memory interface for containers

The first session was led by Hongjian Fan over a remote link; it was focused on how to use CXL memory to support containers. Figuring this out, he said, is complicated by the fact that CXL is new technology and there are no real devices to play with yet. So, much of the work being done is at the conceptual level. The Kubernetes [container storage interface](<https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/>) provides a flexible way to allocate storage to containers; he is working on a "container memory interface" (CMI) to do the same thing with CXL memory. 

Systems can use CMI to provide functionality like [memory tiering](<https://lwn.net/Articles/857133/>) and to manage resources in a pooled-memory system. There are a few scenarios that Fan envisions for how this would all work. One would be that containers would have access to all of the memory available to the system (though managed by the control-group memory controller, of course); in this case, CXL would bring little change. If, instead, the container implements tiered memory, then CMI will control access to the different memory types. There are also pooled-memory scenarios, where the memory is located on an appliance somewhere. 

Fan had a series of questions he was seeking to answer. The first would be whether it is possible to create a common CMI standard that would work across all CXL vendors. With regard to memory tiering, he asked, is it better to do it within the containers, or instead at the host level? There are also open questions about how to manage pooled-memory servers. An attendee started the discussion by asking whether all of this could be managed with control groups, with different types of data packaged as if they were CPUless NUMA nodes. That might be the simplest place to start, Fan answered, but he was not sure that control groups had sufficient flexibility. 

Michal Hocko said that cpusets could perhaps help with the management, but they provide no way to control how memory is distributed across nodes. Dave Hansen said that there is interest in providing control over memory allocation; providers could charge lower rates for access to slower memory, for example. The problem exists now, and people try to manage things with the `numactl` utility, but it's not up to the task. It can block users from certain types of RAM, he said, but it's an all-or-nothing deal. It can't provide the finer quality-of-service control that providers want. 

Dan Williams said that the current work has been focused on DRAM and slower types of memory. CXL is going to bring a broader spectrum of vendors and speeds, and multiple performance classes. While it might make sense to design a system to handle two tiers of memory service now, developers should be thinking about five tiers in the future. Matthew Wilcox said that enterprise vendors are unlikely to want to manage that many tiers, though. 

Adam Manzanares suggested starting with well-defined uses cases and just two tiers. Otherwise, he worries that things will get out of control quickly. Wilcox said that there is a sane three-tier case consisting of CXL memory, DRAM, and persistent memory. But Hansen warned of multiple CXL-attached tiers, and that developers should expect "a lot of weird CXL devices". It is an open standard, and vendors are free to do interesting things with it. 

Fan said that, for any sort of management to work, the kernel will need some idea of the relative performance of each available memory tier. Hansen answered that there is a lot of standards work in this area. ACPI has a way of enumerating NUMA latency, for example, and other mechanisms are under development. The Heterogeneous Memory Attribute Table (HMAT), for example, can provide bandwidth information for each memory type. UEFI, meanwhile, has specified the [Coherent Device Attribute Table](<https://uefi.org/sites/default/files/resources/Coherent%20Device%20Attribute%20Table_1.01.pdf>) (CDAT) with CXL memory, among other types, in mind. 

Williams said that Linux is too dependent on the notion of NUMA distance as a way of describing memory capabilities. There is better information about memory available from the firmware now, but the memory-management code does not make use of it. A baby step might be to boil that information down into a single distance value to at least make some use of it. Manzanares said that distance doesn't work for persistent memory, though, since it cannot capture the asymmetry between read and write speeds. 

Hansen said that the relevant information is available now if an application knows where to look. The harder problem is making decisions about memory placement in the kernel. Different workloads may have different preferences depending on their access patterns; currently, applications have to figure out which memory they want and set up an appropriate NUMA policy. But the kernel could be using memory information to make smarter decisions; moving frequently written pages off of persistent memory, for example. 

There was some discussion about where decisions on tiering should be made. Putting the logic into the kernel makes life easy for applications that don't care about NUMA placement, which is most of them, Williams said, but he worried that there could be fights between the kernel and user space about tiering. Hansen said those fights could happen now, but the kernel's NUMA-placement logic mostly stays out of the way if user space has set an explicit policy. That may be sufficient for future needs as well. 

Williams asked for an explanation of the perceived deficiencies in the current NUMA API. Fan answered that there needs to be a way to set memory limits on a per-node basis; that will require a new control-group or `numactl` knob. Manzanares suggested adding better tiered-memory support to QEMU so that this work could go forward, but Davidlohr Bueso pointed out that it's not possible to get real performance numbers that way. The concern at this point, Manzanares said, is to work out the interface issues rather than to optimize performance. Hansen said that a lot can be done by putting some persistent memory into a system and treating it like another tier; the result "kind of looks like CXL if you squint at it funny". That would give ways to play with interfaces and get some initial performance data. 

Fan thanked the group for having provided a bunch of good information for him to work with, and the session drew to a close. 

#### Managing CXL memory

[![\[Jon Trantham\]](https://static.lwn.net/images/conf/2022/lsfmm/JonTrantham-sm.png)](<https://lwn.net/Articles/894621/>) The next session, led by Jon Trantham, delved into some of the other issues that come up when trying to manage CXL memory. CXL, he said, is a way to attach memory devices that cannot go onto the DDR memory bus. Putting DDR interfaces onto devices can be hard for manufacturers, and DDR does not work all that well with persistent memory. But CXL memory has different performance characteristics than normal RAM. Its latency and bandwidth will differ, and they can change as the device ages. Persistence, endurance, and reliability can all differ as well. 

There are various ways of reporting the characteristics and status of CXL memory, starting with the above-mentioned CDAT table. The CDAT is useful in that it can be updated as performance changes. CXL devices can also produce a stream of event records, indicating that maintenance is required or that performance is falling, for example. CXL 2.0 enables switches that can sit between memory and the computer, allowing memory to live in a different enclosure entirely. That makes actions like hot unplugging possible, but it will be necessary to figure out how to communicate that to the kernel. 

CXL devices must implement decisions about how much memory to allocate to each processor; this can involve an "out-of-band fabric manager" to control the switches. Memory can be interleaved at a granularity as small as 64 bytes, which is great for performance but harder for error recovery; memory failure can leave small holes in the address space. Wilcox answered that the usual management technique in such situations is crashing. 

On the security side, there are access and encryption keys shared between hosts and devices; that brings in the whole key-management problem. The sum of all this, he said, is that help is needed. How is all of this to be managed? Should it be done in the kernel or in user space? 

Williams asked if the encryption features were only for persistent memory. Evidently CXL can provide link encryption for DRAM, but does not encrypt data at rest. Hansen said that it will never be possible for the kernel to recover from errors on a 64-byte boundary; it only handles memory at the page level. He suggested looking at the existing mechanisms and asking whether anything different was really needed; perhaps all of those CXL capabilities aren't really necessary. 

Williams said that CXL makes it possible to turn bare metal into virtual machines; techniques like memory ballooning become possible. So it seems that the same interfaces should be used. Hocko says that ballooning relies on memory hotplug, which "mostly works", but shrinking memory is hard. The memory to be removed can only be used for movable allocations. This is equivalent to a return to the old high-memory systems, where much of the installed memory could not be used by the kernel. 

Hansen answered that the kernel does a reasonable job of emptying a memory area that is to be removed, but there is always the case where a few pages simply cannot be cleared. If there were some way to retain those pages after the memory goes, he said, life would be easier and the whole mechanism would be more reliable. 

The session closed with Manzanares suggesting more coordination between developers and vendors. Perhaps there needs to be some sort of regular group call where these issues are worked out. Chances are that something like that will be set up soon. 

#### Tiering

[![\[Jongmin Gim\]](https://static.lwn.net/images/conf/2022/lsfmm/JongminGim-sm.png)](<https://lwn.net/Articles/894622/>) The final CXL session on Tuesday was led by Jongmin Gim, who wanted to talk about tiering in particular. A lot of things are changing in the CXL 2.0 specification, he began, including the addition of a number of memory types. Tiering will allow the system to make the best use of those memory types, putting frequently used pages in fast memory while using slower memory to hold pages that are not needed as often. 

Support for tiering is not currently upstream, but developers are working on it. There are various issues around promotion and demotion of pages between tiers to be worked out. The demotion side is easy, he said; if there is not enough fast memory available, kick out some pages. Promotion turns out to be harder, though. Current patches (described in [this article](<https://lwn.net/Articles/893024/>)) use the NUMA-balancing scan to try to determine which pages in slower memory are currently being used. When hot pages are found, they can be migrated to faster memory. A heuristic requiring two accesses before promoting a page helps to prevent rapid bouncing of pages between memory types. 

One possible optimization might be to promote contiguous groups of pages together in a single operation. There was some discussion of implementing some sort of predictive algorithm to improve page promotion, but it was all at a fairly high level. 

Manzanares said that the kernel's NUMA balancing was designed when all nodes in a system were more-or-less equal, and it is CPU-centric. He wondered whether the assumptions built into NUMA balancing are still valid in the CXL world. Gorman said that there is no assumption that nodes are the same size in the current code. Hansen said that NUMA balancing is used now for moving data to and from slower persistent-memory nodes, which are always mismatched in size, and it seems to be working now. 

The discussion wandered around the details of NUMA balancing with no real conclusion. At the end of the session, though, there were two points of agreement: CXL devices are highly diverse, and that tiering is the way to manage them.
