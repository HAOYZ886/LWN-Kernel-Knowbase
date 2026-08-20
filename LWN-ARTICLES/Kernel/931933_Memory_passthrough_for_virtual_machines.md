---
title: Memory passthrough for virtual machines
url: https://lwn.net/Articles/931933/
date: "May 19, 2023"
category: "Memory management-Virtualization"
author: "By Jonathan Corbet May 19, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 19, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Memory management is tricky enough on it own, but virtualization adds another twist: now there are two kernels (host and guest) managing the same memory. This duplicated effort can be wasteful if not implemented carefully, so it is not surprising that a lot of effort, from both hardware and software developers, has gone into this problem. As Pasha Tatashin pointed out during a memory-management-track session at the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), though, there are still ways in which these systems run less efficiently than they could. He has put some effort into improving that situation. 

The translation of a virtual address to a physical address is a more complex affair than it seems. The lookup operation must work through as many as five levels of page tables. At each level, a memory load must be performed and TLB misses are possible, meaning that the lookup operation can be slow. It gets worse when this happens in the guest, though; guest "physical" addresses are virtual addresses in the host space; as a result, the lookup at each level of the guest page-table hierarchy can require walking through the full hierarchy in the host. The worst-case lookup, when both the host and the guest are running with five-level page tables, could require 35 loads, which can hurt. 

Optimizing this situation, he said, starts from a recognition that work is being duplicated in the virtualized environment. He was not just referring to page-table lookups; memory is also zeroed twice when virtualization is in use. The solution Tatashin has in mind is to push as much of the work as possible to the host system in ways that are not transparent to the guest. 

[![\[Pasha Tatashin\]](https://static.lwn.net/images/conf/2023/lsfmm/PashaTatashin-sm.png)](<https://lwn.net/Articles/931936/>) Specifically, he has implemented a driver for a "memctl" device that is present on the guest side; this device provides many of the memory-management operations that are already available through the guest's system-call interface: `mmap()`, `mlock()`, `madvise()`, and so on. The difference is that, for the most part, these operations are passed through to the host for execution there rather than being handled by the guest. Additionally, the memctl device does not zero memory on the guest side; it counts on the host take care of that when needed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The other piece of the puzzle is that memctl would allocate pages in the guest's physical address space at the 1GB huge-page size. On the host side, though, these pages are mapped at a smaller size — as either 2MB huge pages or 4KB base pages. The use of 1GB pages on the guest shorts out most of the address-translation overhead at that level, speeding access considerably. The smaller pages on the host side avoid fragmentation issues; guest memory can be managed in smaller units. This only works, though, if all operations on that memory are done by the host, which is why the memctl device must provide equivalents for all of the relevant system calls. 

David Hildenbrand suggested that the real optimization in this setup is avoiding the need to zero pages on the guest side and, perhaps, from not having to allocate all of the guest's memory immediately on the host. He thought that some of these optimizations could be done within the balloon driver as well, but probably not all of them. The virtio-balloon is "the dumping ground" for a lot of similar code, he said. 

Tatashin continued, wondering whether and how it might be possible to upstream this code. Andrew Morton asked where the changes live; the answer is that almost all of the work is in the new memctl device, which is a separate driver. So there would be little impact on the core memory-management code. But Tatashin worried about maintaining the ABI after the code goes upstream and wanted to be sure that he is not adding any security problems. He was advised to copy the patches widely, and the community would figure it out somehow. 

As the session ran out of time, an attendee asked whether this mechanism required changing functions like `malloc()`. Since memory-management operations have to send commands to the memctl device, the answer was "yes", code like allocators would have to change. Perhaps someday it would be possible to do a lot of the basic memctl operations from within the kernel, but more specialized applications would have to do their own memctl calls.
