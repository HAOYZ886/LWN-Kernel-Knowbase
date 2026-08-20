---
title: Plans for hot adding and removing memory
url: https://lwn.net/Articles/553199/
date: "June 12, 2013"
category: "Hotplug-Memory"
author: "By Jake Edge June 12, 2013 LinuxCon Japan 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 12, 2013

* * *

[LinuxCon Japan 2013](<https://lwn.net/Archives/ConferenceByYear/#2013-LinuxCon_Japan>)

At [LinuxCon Japan](<http://events.linuxfoundation.org/events/linuxcon-japan>), Yasuaki Ishimatsu of Fujitsu talked about the status of memory hotplug, with a focus on what still needs to be done to fully support both hot adding and hot removing memory. If a memory device is broken in a laptop or desktop, you can just replace that memory, but for servers, especially ones that need to stay running, it is more difficult. In addition, having a way to add and remove memory would allow for dynamic reconfiguration on systems where the hardware has been partitioned into two or more virtual machines. 

The focus of the memory hotplug work is for both scenarios: broken memory hardware and dynamic reconfiguration. Memory hotplug will be supported in KVM, Ishimatsu said. It is currently supported by several operating systems, but Linux does not completely support it yet. Fixing that is the focus this work. 

There are two phases to memory hotplug: physically adding or removing memory (hot add or hot remove) and logically changing the amount of memory available to the system (onlining or offlining memory). Both phases have to be completed before Linux can use any new memory, and taking the memory offline (so that Linux is no longer using it) is required before it can be removed. 

The memory management subsystem manages physical memory by using two structures, he said. The page tables hold a direct mapping for virtual to physical addresses. The virtual memory map manages page structures. In order to offline memory, any data needs to be moved out of the memory and those data structures need to be updated. Likewise, when adding memory, new page table and virtual memory map entries must be added. 

Pages are managed in zones and, when using the sparse memory model that is needed for memory hotplug systems, zones are broken up into sections that are 128M in size. Sections can be switched from online to offline and vice versa using the `/sys/devices/system/memory/memoryX/state` file. By echoing `offline` or `online` into that file, the pages in that section have their state changed to unusable or usable respectively. 

In the 3.2 kernel, hot adding memory and onlining it were fully supported. Offlining memory was supported with limitations, and hot removing it was not supported at all. Work started in July 2012 to remove the offline limitations and to add support for hot remove, Ishimatsu said. 

The work for hot remove has been merged for the 3.9 kernel. It will invalidate page table and virtual memory map entries that correspond to the memory being removed. But, since the memory must be taken offline before it is removed, the limitations on memory offline still make it impossible to remove arbitrary memory hardware from the system. 

When memory that is to be offlined has data in it, that data is migrated to other memory in the system. But the only pages that are migratable this way are the page cache and anonymous pages, which are known as "movable" pages. If the memory contains non-movable memory, which Ishimatsu called "kernel memory", the section cannot be offlined. 

There are two ways to handle that problem that are being considered. The first is to support moving kernel memory when offlining pages that contain it. The advantages to that are that all memory can be offlined and there is no additional performance impact for NUMA systems since there are no restrictions on the types of allocations that can be made. On the downside, though, the kernel physical to virtual address relationship will need to change completely. The other alternative is to make all of a node's memory movable, which would reuse the existing movable memory feature, but means that only page cache and anonymous pages can be stored there, which will impact the performance of that NUMA node. 

Ishimatsu said that he prefers the first solution personally, but, as a first step they are implementing the second: creating a node that consists only of movable memory. Linux has the idea of a movable zone (i.e. `ZONE_MOVABLE`), but zones of that type are not created automatically. If a node consists only of movable memory, all of it can be migrated elsewhere so that the node can be taken offline. 

A new boot option, `movablecore=acpi`, is under development that will use the memory affinity structure in the ACPI static resource affinity table (SRAT) to choose which nodes will be constructed of movable memory. The existing use for `movablecore` allows setting aside a certain amount of memory that will be movable in the system, but it spreads it evenly across all of the nodes rather than concentrating it only on the nodes of interest. The "hotpluggable" bit for a node in the SRAT will be used to choose the target nodes in the new mode. 

Using the `online_movable` flag to the sysfs memory `state` file (rather than just `online`) allows an administrator to tell the system to make that memory movable. Without that, the onlined memory is treated as `ZONE_NORMAL`, so it may contain kernel memory and thus not be able to be offlined. The `online_movable` feature was merged for 3.8. That reduces the limitations on taking memory offline, but there is still work to do. 

Beyond adding the `movablecore=acpi` boot option (and possibly a `vm.hotadd_memory_treat_as_movable` sysctl), there are some other plans. Finding a way to put the page tables and virtual memory map into the hot-added memory is something Ishimatsu would like to see, because it would help performance on that node, but would not allow that memory to be offlined unless those data structures can be moved. He is thinking about solutions for that. Migrating `vmalloc()` data to other nodes when offlining a node is another feature under consideration. 

Eventually, being able to migrate any kernel memory out of a node is something he would like to see, but solutions to that problem are still somewhat elusive. He encouraged those in attendance to participate in the discussions and to help find solutions for these problems. 

[I would like to thank the Linux Foundation for travel assistance to Tokyo for LinuxCon Japan.]
