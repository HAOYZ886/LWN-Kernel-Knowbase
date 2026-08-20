---
title: "CXL 2: Pooling, sharing, and I/O-memory resources"
url: https://lwn.net/Articles/894626/
date: "May 19, 2022"
category: Compute Express Link CXL; IO memory
author: "By Jonathan Corbet May 19, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 19, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

During the final day of the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), attention in the memory-management track turned once again to the challenges posed by the upcoming Compute Express Link (CXL) technology. Two sessions looked at different problems posed by CXL memory, which can come and go over the operation of the system. CXL offers a lot of flexibility, but changes will be needed for the kernel to be able to take advantage of it. 

#### Pooled and shared memory

Hongjian Fan, who led one of [Tuesday's CXL sessions](<https://lwn.net/Articles/894598/>) returned on Wednesday (via videoconference) for a discussion that was dedicated to pooled and shared memory. These are concepts that apply to memory appliances, where the goals are to share memory across multiple systems, improve memory utilization and, naturally, to reduce costs. Sharing memory from a central appliance can reduce the need to put large amounts of memory into every server; when a given machine needs more, it can get a temporary allocation from the appliance. 

Pooled memory is partitioned on the appliance and allocated in chunks to servers, which only have access to the memory that has been given to them. Requesting memory from a pooled appliance creates a hotplug event, where new memory suddenly becomes addressable. Supporting pooled memory requires the ability to generate and manage the hotplug events, as well as a virtual-device driver that monitors memory use and requests or releases memory as appropriate. 

Shared memory is, instead, shared across all servers, though it will probably not be possible for any given server to allocate it all. With a shared appliance, the memory is always in each server's physical address space, but it may not all be usable. The kernel can provide a sysfs file that indicates which memory is available at any given time; tracking of allocations can done by the appliance or via communication between servers, though the latter mode can create a lot of traffic. 

Dave Hansen said that CXL memory behaves a lot like RAM today, but it requires some extra care. There may be cache-coherency issues not present with RAM, and the kernel can't keep any of its own data structures in this memory since those structures cannot be moved and would thus block removal. Fan said that cache coherency is part of the CXL protocol and shouldn't be a problem. Hansen added that there is little that is new with CXL memory appliances, they are much like how memory is managed with virtualization. But now it is being done in hardware, which scares him a bit. Memory removal success is "a matter of luck" now, he said, and calling this memory "CXL" won't change that. 

An attendee asked what the benefit of the shared mode was, given that all memory will still be used exclusively by one system at any given time. Fan answered that the problem with pooled access is fast and reliable hotplugging, while the problem with shared access is communication between the systems. Hansen asked how access to shared memory is cut off when memory is reallocated, but Fan was unable to answer the question. 

Dan Williams said that access control is not really visible to the kernel, and that it was necessary to "trust the switch". He added that users want to be able to manage this memory with the existing NUMA APIs, but they also want hard guarantees that it will be possible to remove memory from a system; those two goals are in conflict. It will be necessary to reset expectations about removal, he said; it will be a learning experience for the industry. Hansen said that the use of hotplug will be no different in this scenario, but Williams said there will now be a whole new level of software behind hotplug to manage the physical address space. That is something that the firmware has always done, but now the kernel will have to deal with it; the CXL specification group is still trying to figure out the details of how that will work. 

Fan said some other changes will be necessary as well. There will need to be a mechanism to warn about available capacity on the appliance. Since memory can be requested and added to the system on the fly, the out-of-memory handler should perhaps wait for more memory to materialize before it starts killing processes. David Hildenbrand said that the out-of-memory scenario scares him; people think that it's possible to just wait for memory to appear, but it's not true. If the system is going into the out-of-memory state, there will be other allocations failing at the same time. What is needed is a way to determine that the system is short of memory, then wait for more memory in a safe way, before running out. Hansen added that plugging in more memory is an act that, in itself, requires allocating memory, and an out-of-memory situation is not a good time to try to do that. Williams said, as the session came to a close, that the system cannot be reactionary, and that memory requirements should be handled in user space at the job-scheduling level. 

#### Managing the resource tree

Management of the physical address space was the topic of the second CXL session of the day. The [`resource` structure](<https://elixir.bootlin.com/linux/v5.17.8/source/include/linux/ioport.h#L21>) is one of the oldest data structures in the kernel; it was added in the 2.3.11 release in 1999. Its job is to track the resources available to the system and, in the form of the `iomem_resource` variable, the layout of the computer's physical address space. It forms a tree structure with some resources (a PCI bus, for example) containing other resources (attached devices) within their address ranges. This tree is represented in `/proc/iomem`, which must be opened as root to show the actual addresses involved. 

[![\[Ben Widawsky\]](https://static.lwn.net/images/conf/2022/lsfmm/BenWidawsky-sm.png)](<https://lwn.net/Articles/895430/>) The kernel's I/O-memory resource tree was not designed with CXL in mind; for Linus Torvalds to have been so short-sighted in 1999 is perhaps forgivable. But, said Ben Widawsky in his session, that shortcoming is threatening to create problems now. In current systems, `iomem_resource` is initially created from the memory map provided by the boot firmware; architecture-specific code and drivers then modify it and subdivide the resources there as needed. Once a given range of physical address space has been assigned to a specific use, it can never be reassigned — only subdivided. 

The core of the problem is that CXL memory can come and go, and it may not be present at boot time. When this memory is added, it essentially overrides a piece of the physical address space, which is something that `iomem_resource` is not prepared to handle. If the space used by CXL were disjoint from local system resources, Widawsky said, there wouldn't be a problem; traditional resources could be put into one range, and CXL in another. But that is not how things are going to work. RAM added via CXL will overlap the space already described by `iomem_resource`. What, he asked, can be done to properly represent these resources? 

Mike Rapoport questioned the need to put CXL memory into `iomem_resource` at all. The problem, Hansen explained, is that CXL memory might be the only memory in the system. People tend to see CXL as a sort of add-on card, but it is closer to the core than that. On a system using only CXL, it would not be possible to boot without having that memory represented in `iomem_resource`. David Hildenbrand said that `iomem_resource` should describe everything in the system. 

Widawsky said that there is a need to keep device-private memory from taking address space intended for CXL; this is another reason to represent CXL memory in the resource tree. He suggested that attempts to take pieces of memory assigned to CXL should be blocked. Hildenbrand suggested creating the CXL region as a device and adding some special calls to allocate space from that region. This could be tricky, Widawsky said. System RAM may already be set up in the resource tree; making it part of a special device would involve reparenting that RAM, which, he said, has never been done. Matthew Wilcox contradicted the "never been done" claim, but without details on when it had been done. 

John Hubbard said that the kernel should keep `iomem_resource` as "the one truth" about the layout of the physical address space. Williams said that `struct resource` is old; there are people around who love to add new structures to the kernel, perhaps the time has come to do that for this problem. Wilcox referenced a "20-year-old patch" in Andrew Morton's tree, but didn't identify it. Hildenbrand said that the structure as a whole is difficult to traverse and work with; any work to improve it would be appreciated. 

Widawsky asked if there was a path to a solution that involved a bit less hard work. Williams suggested adding resources in smaller chunks, with a number of entries for the CXL CFMWS ("fixed memory window structures") areas. Some of those entries could later be removed, Widawsky added, if it turned out they weren't being used for CXL memory. 

The session came to an end with Wilcox asking what would happen in response to a discovery that an assigned resource's range is too small. Could it be expanded somehow? Williams said it would be good to be able to update the address map as more information became available. All told, the session described a problem but did not get close to finding a solution. This is a problem that has been seen in numerous other contexts as computers have become more dynamic. Solutions have been found in the past and will surely be found this time too, but it may be challenging to find one that doesn't involve a fair amount of hard work.
