---
title: "Phyr: a potential scatterlist replacement"
url: https://lwn.net/Articles/931943/
date: "May 19, 2023"
category: Direct memory access
author: "By Jonathan Corbet May 19, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 19, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The "scatterlist" is a core-kernel data structure used to describe DMA I/O operations from the point of view of both the CPU and the peripheral device. Over the years, the shortcomings of scatterlists have become more apparent, but there has not been a viable replacement on the horizon. During a memory-management session at the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Jason Gunthorpe described a possible alternative, known alternatively as "phyr", "physr", or "rlist", that might improve on scatterlists for at least some use cases. 

The buffer for an I/O operation is usually described by an address and a length. In the virtual address space where the operation is requested, that buffer is usually seen as being contiguous. Things may look quite different from a peripheral device's point of view; that seemingly contiguous buffer may be scattered randomly in the physical address space. The virtual address used to locate that buffer has no meaning to the device — and the CPU's physical addresses might not either, especially if there is an I/O memory-management unit (IOMMU) sitting in the middle. Instead, the device works with DMA addresses that may exist in their own space. 

These differing points of view mean that I/O operations must be described in two different ways; that is the role of [`struct scatterlist`](<https://elixir.bootlin.com/linux/v6.4-rc1/source/include/linux/scatterlist.h#L11>). It contains an (address, offset, length) tuple as seen by the CPU, where the address is actually a pointer to the `page` structure for the page holding the buffer; a scatterlist is an array of these structures. The DMA-mapping layer can use that information to augment that array with addresses and lengths visible to the target device. If an IOMMU is used to coalesce a scattered set of pages and make them look contiguous to the device, the set of tuples seen by the device may be shorter than that provided by the CPU. 

[![\[Jason Gunthorpe\]](https://static.lwn.net/images/conf/2023/lsfmm/JasonGunthorpe-sm.png)](<https://lwn.net/Articles/931944/>) Gunthorpe started his session by listing a number of problems experienced by developers when using the kernel's DMA-mapping layer, many of which are tied to the scatterlist representation. It would be useful if functions like `pin_user_pages()` could return folios, but the current API works on individual pages, so I/O operations can end up splitting up huge pages. Since scatterlists work with `page` structures, they cannot represent memory that lacks such structures, as is often the case with memory installed on the devices themselves; this is a problem for [P2PDMA](<https://lwn.net/Articles/767281/>) operations, among others. The block layer would be faster, he said, if I/O requests stored in BIO structures did not need to be converted to scatterlists on their way through. RDMA users want to be able to pin large amounts of memory (he said 100GB) and perform I/O on it "forever"; storing such allocations in a scatterlist is a useless waste of memory. 

Matthew Wilcox added another reason not to like them: simple cleanliness. Gunthorpe agreed that everybody hated scatterlists; they are found everywhere in the kernel, and "abused and misused everywhere". The structure is hopelessly tied to `struct page`. There is no hope, he said, of doing something better with it. 

Gunthorpe's approach is to improve the DMA API to provide better operations; an initial implementation can be found in [his GitHub repository](<https://github.com/jgunthorpe/linux/commit/2339ecbcce0ca56361173a77e54dc71ab83027bc>). It involves creating a "range CPU" iterator that would operate over intervals of (physical) CPU memory; it could be used to create a DMA buffer that would serve as a handle for peer-to-peer memory and which could be stuffed into an IOMMU. There would be an equivalent "range DMA" iterator to iterate over DMA addresses, and various options to map between the two. A new pinning API would take a range CPU iterator as an argument. There would be a number of storage options for the iterator, including scatterlists, BIOs, `page` structures, and more. Users could then iterate over these ranges without worrying about how they are represented. 

He started into the project thinking that "this doesn't sound too bad", but got a quick education, he said. There are 23 separate sets of DMA operations in the kernel, he said, many of which are for "weird old IOMMUs" like [GARTs](<https://en.wikipedia.org/wiki/Graphics_address_remapping_table>). He really doesn't want to touch that code. So, instead, he is working on a performant version of a new set of DMA operations for current architectures without trying to support the older ones. 

Then, he said, there is the perennial issue of the `get_user_pages()` family of functions, which are used in many performance-critical places in the kernel. Getting these functions to return data beyond the `page` structures they handle now will be costly; he wondered if there would be any appetite for slowing down `get_user_pages()` for this improvement. Dan Williams asked what kind of added output was being discussed here; Gunthorpe said that the functions would return a set of folios. 

Wilcox said that there are two types of users of these functions, some of which are performance critical and some of which are not. The former users can continue to use `get_user_pages()`, while the others could call something like `get_user_range()` instead and get the extra information. Gunthorpe said that would involve duplicating much of the `get_user_pages()` code, when there are already a couple of implementations in the kernel. John Hubbard suggested creating the new version of the interface, then opportunistically factoring pieces out as it makes sense. 

The session began to wind down with a seeming consensus that this work is on the right track. Williams said that, if it turns out to be useful, it would eventually be necessary to rewrite all of the existing scatterlist users, but that idea received some pushback. Gunthorpe said it would be great if everybody used the new API, but getting there would be painful work that is not likely to happen. Wilcox agreed that existing scatterlist users should mostly be left alone; they can be converted at leisure later. Gunthorpe, though, repeated that a complete conversion was not likely to ever happen.
