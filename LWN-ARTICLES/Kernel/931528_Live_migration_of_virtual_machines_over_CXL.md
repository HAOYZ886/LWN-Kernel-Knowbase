---
title: Live migration of virtual machines over CXL
url: https://lwn.net/Articles/931528/
date: "May 15, 2023"
category: Compute Express Link CXL; Virtualization
author: "By Jonathan Corbet May 15, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 15, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Virtual-machine hosting can be a fickle business; once a virtual machine has been placed on a physical host, there may arise a desire to move it to a different host. The problem with migrating virtual machines, though, is that there is a period during which the machine is not running; that can be disruptive even if it is brief. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Dragan Stancevic, presenting remotely, showed how [CXL](<https://www.computeexpresslink.org/>) shared memory can be used to migrate virtual machines with no offline time. 

Traditional migration, Stancevic began, is a four-step process. As much of the virtual machine as possible is pre-copied to its new home while that machine continues to run, after which the virtual machine is quiesced — stopped so that its memory no longer changes. A second copy operation is done to update anything that may have changed during the pre-copy phase then, finally, the moved machine is relaunched in its new home. Stancevic's goal is to create a "nil-migration" scheme that takes much of the work — and the need to quiesce the target machine — out of the picture. 

[![\[Dragan
Stancevic\]](https://static.lwn.net/images/conf/2023/lsfmm/DraganStancevic-sm.png)](<https://lwn.net/Articles/931656/>) Specifically, this scenario is meant to work in situations where both physical hosts have access to the same pool of CXL shared memory. In such a setting, [`migrate_pages()`](<https://man7.org/linux/man-pages/man2/migrate_pages.2.html>) can be used to move the virtual machine's pages to the shared-memory pool without disturbing the operation of the machine itself; at worst, its memory accesses slow down slightly. Once the memory migration is complete, the virtual machine can be quickly handed over to the new host, which also has access to that memory; the machine should be able to begin executing there almost immediately. The goal is to make virtual-machine migration as fast as task switching on a single host — an action that could happen transparently between that machine's normal time slices. Eventually, the new host could migrate the virtual machine's memory into its own directly attached RAM. 

This work, he said, is still in an early stage. It does, however, have a mailing list and [a web site at nil-migration.org](<https://nil-migration.org/>). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

David Hildenbrand asked about pass-through devices — devices on the host computer that are made directly available to a virtual machine. Those, he said, cannot be migrated through CXL memory, or in any other way, for that matter. Stancevic agreed that such configurations simply would not work. Dan Williams asked whether migration through CXL memory was really necessary or if, instead, virtual machines could just live in CXL memory all the time. In Stancevic's use case, CXL shared-memory pools are only used for virtual-machine migration, but other configurations are possible. 

Another audience member asked whether it would be possible to do a pre-copy phase over the network first, and only use CXL for any remaining pages just before the move takes place. Stancevic answered that it could work, but would defeat the purpose of keeping the virtual machine running at all times. 

Yet another attendee pointed out that CXL memory may not be mapped at the same location on each physical host, and wondered how the nil-migration scheme handles that. The answer is that, so far, this scheme has only been tested with QEMU-emulated hardware (CXL 3.0 hardware won't be available for a while yet), and it is easy to make the mappings match in that environment. This will be a problem when real hardware arrives, though, and a solution has not yet been worked out. 

The final question, from Williams, was whether the nil-migration system would need new APIs to identify the available CXL devices. Stancevic answered that having CXL resources show up as available as NUMA nodes is the best solution, but that it would be good to have some metadata show up in sysfs to help with figuring out the paths between the hosts.
