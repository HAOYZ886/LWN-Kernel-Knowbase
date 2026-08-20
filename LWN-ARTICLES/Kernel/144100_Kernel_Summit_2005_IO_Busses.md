---
title: "Kernel Summit 2005: I/O Busses"
url: https://lwn.net/Articles/144100/
date: "July 19, 2005"
category: Direct memory access; PCI
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
The session on I/O buses got off to a bit of a slow start. PCI-Express support was the first topic to cover, but the fact that, for the most part, PCI-Express simply works put a damper on the discussion. There are some potential issues with interrupt ordering, but nothing anybody was all that worried about. 

PCI error recovery was the next topic. Ben Herrenschmidt discussed the two current patches out there; these patches were discussed in detail in [the July 13 LWN Kernel Page](<http://lwn.net/Articles/142906/>), so there is no need to repeat that material here. The discussion did not produce any clear consensus on which patch - if either - should be merged. 

The bulk of the time was spent discussing I/O memory management units. The IOMMU, on systems which have them, implements a virtual address space seen by peripherals. This layer of indirection has a couple of advantages: it can be used to make otherwise unreachable memory available for DMA operations, and it can cause physically distributed memory to appear to be contiguous. The IOMMU, thus, can be used to implement scatter/gather I/O operations without the target device knowing about it. 

As it turns out, there are several different scatter/gather interfaces in the kernel. The block layer has its own, well developed code; networking, too, has a strong scatter/gather implementation. There are, however, a number of char drivers which wish to do scatter/gather I/O, and each one has its own implementation; some are better than others. There have been suggestions for a generic scatter/gather implementation for the whole kernel before, and James raised the idea again. Nobody appears to be in a hurry to reimplement kernel scatter/gather operations, though; the parts of the kernel which benefit most from the capability already have solid implementations that work well. 

Just because a system has an IOMMU does not mean that it should always be used. Some hardware implements its own scatter/gather operations, and, in many cases, that implementation is quite efficient. Currently there is no mechanism in the kernel for using - or bypassing - the IOMMU on a per-device basis, and no real way to know which option performs the best. What became clear in the discussion is that somebody needs to put some serious effort into measuring the performance impact of the IOMMU on various architectures, and with various peripherals. For now, what most people have seems to be guesswork. 

Peripherals which perform their own scatter/gather often can support both 32-bit and 64-bit descriptors. For many reasons, the use of 32-bit descriptors tends to be more efficient. The question was asked: is it worth using the 64-bit modes at all? One answer came from one hardware company representative: support for 32-bit descriptors can be expected to fade away. Commercial pressures may lead to 64-bit being the only well-supported mode available. 

Finally, there is the issue of support for systems without an IOMMU. Such systems cannot work with 64-bit DMA addresses. In fact, many of them do not even handle full 32-bit addresses; it is fairly common to find hardware which can only address 31 bits. The kernel does not currently handle such hardware well. The memory zone mechanism was designed around this _type_ of problem, but there are only two zones of interest; for a device which cannot deal with 32-bit addresses, the only safe zone is the DMA zone, which, reflecting its history, only uses the bottom 24 bits. This zone is thus constrained to be small. There really needs to be a way to allocate memory which is outside of the traditional DMA zone, but which still fits within a constrained address mask. 

Thus, asks James, do we need a new memory allocation API? He proposed either `kmalloc_dev()` or `kmalloc_mask()`. The former would take a pointer to a `device` structure; it would have the advantage of also being able to allocate local memory on a NUMA system. The latter, instead, would simply try to find memory addressable within a given mask. In either case, the new functions would not be implemented through the creation of new memory zones - nobody wants to add more zone balancing challenges to the kernel. Instead, a best-effort attempt would be made to allocate suitable memory from the lower end of `ZONE_NORMAL`; if that does not work, the attempt will fail. 

As for the question of which API should be implemented: the developers decided that they wanted both. James promised to post a patch.
