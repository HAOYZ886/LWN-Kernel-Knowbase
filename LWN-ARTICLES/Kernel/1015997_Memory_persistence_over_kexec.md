---
title: Memory persistence over kexec
url: https://lwn.net/Articles/1015997/
date: "April 1, 2025"
category: Kexec; Virtualization
author: "By Jonathan Corbet April 1, 2025 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 1, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel's [kexec mechanism](<https://man7.org/linux/man-pages/man8/kexec.8.html>) allows one kernel to directly boot a new one; it can be thought of as a sort of kernel equivalent to the [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) system call. Kexec has a number of uses, including booting a special kernel to perform dumps after a crash. Normally, one does not expect user-space processes to survive booting into a new kernel, but that has not stopped developers from trying to implement that ability. Mike Rapoport ran a memory-management-track session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit to discuss one piece of that problem: enabling the contents of memory to persist across a kexec handover so that the new kernel can pick up where the old one left off. 

The use case that is being worked on is to allow large cloud providers to update the kernel on a system hosting virtual machines without disturbing those machines in the process. The proposed new subsystem to enable this functionality is called [Kexec HandOver](<https://lwn.net/ml/linux-kernel/20250320015551.2157511-1-changyuanl@google.com/>), or KHO, but KHO needs layers on top to implement the persistence of memory contents, and how that will work is not yet entirely clear. There has to be some way for the system to serialize and deserialize state, and a controlling state machine to manage the whole procedure. One partial solution, called [FDBox](<https://lwn.net/ml/linux-kernel/20250307005830.65293-1-ptyadav@amazon.de/>), can preserve file descriptors, mostly associated with in-memory objects like memfds. 

[![\[Mike Rapoport\]](https://static.lwn.net/images/conf/2025/lsfmm/MikeRapoport-sm.png)](<https://lwn.net/Articles/1016000/>) The KHO layer is meant to provide the needed serialization mechanism and to preserve the contents of non-movable memory ranges. Outside of those ranges, it will create scratch areas that the replacement kernel can use for early allocation without stepping on the preserved memory contents. There are several of these scratch areas, one of which is global, one in low memory for devices with DMA limitations, and one per NUMA node. These areas are reserved when the second kernel boots; once the boot process is complete, the scratch areas can be turned into [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) regions. 

Subsystems that need to preserve memory across a boot will have to mark it specifically with calls to functions like `kho_preserve_folio()`. There may eventually be a call to preserve slab-allocated memory as well. When the second kernel boots, it will mark the preserved areas as reserved and not set up the memory map for them until the preserved memory is claimed. As the regions are claimed, the physical memory they occupy will be configured to look as if it had been allocated from the second kernel's buddy allocator. 

There are, Rapoport allowed, some ""minor issues"" with this whole scheme. This handover dance is expected to happen repeatedly over time, not just once, and that leads to problems with the management of the scratch areas. They can only be allocated when a handover is called for, and the outgoing kernel can only guess at how much memory the new kernel will need. The availability of memory for the scratch areas will depend on how fragmented memory has become on the old system; it may not be possible to set aside as much memory as the outgoing kernel thinks is needed. If the resulting scratch areas are insufficient for the new kernel, the handover will fail badly. There are currently not a lot of good ideas circulating on how to address this problem. 

The handover mechanism currently cannot preserve movable memory, since the preserved pages cannot be in a scratch area. So, eventually, movable pages will have to be migrated out of scratch areas (which may not be possible, as was [discussed](<https://lwn.net/Articles/1015551/>) earlier that day), then pinned in their new location to prevent them from being moved back. Restoring folios will become harder as the transition to memory descriptors goes forward, but that is a problem for 2026, he said. Matthew Wilcox said that drivers should be allocating pages, not folios, so the problem might not be that bad. Jason Gunthorpe said, though, that drivers for I/O memory-management units will be allocating [page-table descriptors](<https://lwn.net/Articles/937839/>), which will complicate the story. 

The biggest problem, Rapoport concluded, is making this whole handover process fast. Preserving random pages is expensive, and the best data structure to use for that task is not clear, though [maple trees](<https://lwn.net/Articles/845507/>) look promising. There may be a need to coalesce memory to be preserved at allocation time as a way to reduce the cost of preserving it, perhaps requiring the use of a new GFP flag. Performance would also be helped if the maintenance of memory blocks could be bypassed during deserialization. 

As the discussion wound down, Gunthorpe said that performance may not be as big a problem as it seems. The target systems for KHO are likely to be using a lot of 1GB huge pages, so memory will not fragment that much. There are, meanwhile, a lot of questions yet about how much memory really needs to be preserved; he suggested getting the functionality working, then reevaluating performance questions afterward. Liam Howlett threw in a final suggestion that maybe the existing hibernation mechanism could be used rather than implementing an entirely new handover subsystem, but Rapoport thought that any solution using hibernate would be far too slow.
