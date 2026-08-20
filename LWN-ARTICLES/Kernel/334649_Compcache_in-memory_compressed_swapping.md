---
title: "Compcache: in-memory compressed swapping"
url: https://lwn.net/Articles/334649/
date: "May 26, 2009"
category: "Memory management-Swapping"
author: "May 26, 2009 This article was contributed by Nitin Gupta"
---

May 26, 2009

This article was contributed by Nitin Gupta

The idea of memory compression--compress relatively unused pages and store them in memory itself--is simple and has been around for a long time. Compression, through the elimination of expensive disk I/O, is far faster than swapping those pages to secondary storage. When a page is needed again, it is decompressed and given back, which is, again, much faster than going to swap. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

An implementation of this idea on Linux is currently under development as the [compcache](<http://code.google.com/p/compcache/>) project. It creates a virtual block device (called ramzswap) which acts as a swap disk. Pages swapped to this disk are compressed and stored in memory itself. The project home contains use cases, performance numbers, and other related bits. The whole aim of the project is not just performance -- on swapless setups, it allows running applications that would otherwise simply fail due to lack of memory. For example, Edubuntu [included](<https://wiki.edubuntu.org/Compcache>) compcache to lower the RAM requirements of its installer. 

The [performance page](<http://code.google.com/p/compcache/wiki/Performance>) on the project wiki shows numbers for configurations that closely match netbooks, thin clients, and embedded devices. These initial results look promising. For example, in the benchmark for thin clients, ramzswap gives nearly the same effect as doubling the memory. Another [benchmark](<http://code.google.com/p/compcache/wiki/Performance/SwapDiskVsRamz>) shows that average time required to complete swap requests is reduced drastically with ramzswap. With a swap partition located on a 10000 RPM disk, average time required for swap read and write requests was found to be 168ms and 355ms, respectively. While with ramzswap, corresponding numbers were mere 12µs and 7µs, respectively -- this includes time for checking zero-filled pages and compressing/decompressing all non-zero pages. 

The approach of using a virtual block device is a major simplification over earlier attempts. The previous implementation required changes to the swap write path, page fault handler, and page cache lookup functions (`find_get_page()` and friends). Those patches did not gain widespread acceptance due to their intrusive nature. The new approach is far less intrusive, but at a cost: compcache has lost the ability to compress page cache (filesystem backed) pages. It can now compress swap cache (anonymous) pages only. At the same time, this simplicity and non-intrusiveness got it included in [Ubuntu](<https://wiki.ubuntu.com/Compcache>), ALT Linux, [LTSP](<http://www.ltsp.org/twiki/bin/view/Ltsp/AltLinux>) (Linux Terminal Server Project) and maybe other places as well. 

It should be noted that, when used at the hypervisor level, compcache can compress any part of the guest memory and for any kind of guest OS (Linux, Windows etc) -- this should allow running more virtual machines for a given amount of total host memory. For example, in KVM the guest physical memory is simply anonymous memory for the host (Linux kernel in this case). Also, with the recent [MMU notifier](<http://lwn.net/Articles/266320/>) support included in the Linux kernel, nearly the entire [guest physical memory is now swappable [PDF]](<http://compcache.googlecode.com/svn/wiki/files/documents/KvmForum2008_kdf2008_15.pdf>). 

#### Implementation

All of the individual components are separate kernel modules: 

  * LZO compressor: lzo_compress.ko, lzo_decompress.ko (already in mainline)
  * xvMalloc memory allocator: xvmalloc.ko
  * compcache block device driver: ramzswap.ko

Once these modules are loaded, one can just enable the ramzswap swap device: 

```
swapon /dev/ramzswap0
```

Note that ramzswap cannot be used as a generic block device. It can only handle page-aligned I/O, which is sufficient for use as a swap device. No use case has yet come to light that would justify the effort to make it a generic compressed read-write block device. Also, to minimize block layer overhead, ramzswap uses the "no queue" mode of operation. Thus, it accepts requests directly from the block layer and avoids all overhead due to request queue logic. 

The ramzswap module accepts parameters for "disk" size, memory limit, and backing swap partition. The optional backing swap partition parameter is the physical disk swap partition where ramzswap will forward read/write requests for pages that compress to a size larger than `PAGE_SIZE/2` -- so we keep only highly compressible pages in memory. Additionally, purely zero filled pages are checked and no memory is allocated for such pages. For "generic" desktop workloads (Firefox, email client, editor, media player etc.), we typically see 4000-5000 zero filled pages. 

#### Memory management

One of the biggest challenges in this project is to manage variable sized compressed chunks. For this, ramzswap uses memory allocator called [xvmalloc](<http://code.google.com/p/compcache/wiki/xvMalloc>) developed specifically for this project. It has O(1) malloc/free, [very low fragmentation](<http://code.google.com/p/compcache/wiki/xvMallocPerformance>) (within 10% of ideal in all tests), and can use highmem (useful on 32-bit systems with >1G memory). It exports a _non-standard_ allocator interface: 

```
struct xv_pool *xv_create_pool(void);
        void xv_destroy_pool(struct xv_pool *pool);
    
        int xv_malloc(struct xv_pool *pool, u32 size, u32 *pagenum, u32 *offset, gfp_t flags);
        void xv_free(struct xv_pool *pool, u32 pagenum, u32 offset);
```

`xv_malloc()` returns a `<pagenum, offset>` pair. It is then up to the caller to map this page (with `kmap()`) to get a valid kernel-space pointer. 

The justification for the use of a custom memory allocator was provided when the compcache [patches](<http://lkml.org/lkml/2009/3/30/182>) were posted to linux-kernel. Both the SLOB and SLUB allocators were found to be unsuitable for use in this project. SLOB targets embedded devices and [claims](<http://lwn.net/Articles/157944/>) to have good space efficiency. However, it was found to have some major problems: It has O(n) alloc/free behavior and can lead to large amounts of wasted space as detailed in [this LKML post](<http://lkml.org/lkml/2009/3/18/210>). 

On the other hand, SLUB has different set of problems. The first is the usual fragmentation issue. The data presented [here](<http://code.google.com/p/compcache/wiki/AllocatorsComparison>) shows that kmalloc uses ~43% more memory than xvmalloc. Another problem is that it depends on allocating higher order pages to reduce fragmentation. This is not acceptable for ramzswap as it is used in tight-memory situations, so higher order allocations are almost guaranteed to fail. The xvmalloc allocator, on the other hand, always allocates zero-order pages when it needs to expand a memory pool. 

Also, both SLUB and SLOB are limited to allocating from _low memory_. This particular limitation is applicable only for 32-bit system with more than 1G of memory. On such systems, neither allocator is able to allocate from the high memory zone. This restriction is not acceptable for the compcache project. Users with such configurations reported memory allocation failures from ramzswap (before xvmalloc was developed) even when plenty of high-memory was available. The xvmalloc allocator, on the other hand, is able to allocate from the high memory region. 

Considering above points, xvmalloc could potentially replace the SLOB allocator. However, this would involve lot of additional work as xvmalloc provides a non-standard malloc/free interface. Also, xvmalloc is [not scalable](<http://code.google.com/p/compcache/issues/detail?id=24>) in its current state (neither is SLOB) and hence cannot be considered as a replacement for SLUB. 

The memory needed for compressed pages is not pre-allocated; it grows and shrinks on demand. On initialization, ramzswap creates an xvmalloc memory pool. When the pool does not have enough memory to satisfy an allocation request, it grows by allocating single (0-order) pages from kernel page allocator. When an object is freed, xvmalloc merges it with adjacent free blocks in the same page. If the resulting free block size is equal to `PAGE_SIZE`, i.e. the page no longer contains any object; we release the page back to the kernel. 

This allocation and freeing of objects can lead to fragmentation of the ramzswap memory. Consider the case where a lot of objects are freed in a short period of time and, subsequently, there are very few swap write requests. In that case, the xvmalloc pool can end up with a lot of partially filled pages, each containing only a small number of live objects. To handle this case, some sort of xvmalloc memory defragmentation scheme would need to be implemented; this could be done by relocating objects from almost-empty pages to other pages in the xvmalloc pool. However, it should be noted that, practically, after months of use on several desktop machines, waste due to xvmalloc memory fragmentation never exceeded 7%. 

#### Swap limitations and and tools

Being a block device, ramzswap can never know when a compressed page is no longer required -- say, when the owning process has exited. Such stale (compressed) pages simply waste memory. But with recent "[swap discard](<http://lwn.net/Articles/297695/>)" support, this is no longer as much of a problem. Swap discard sends BIO_RW_DISCARD bio request when it finds a free swap cluster during swap allocation. Although compcache does not get the callback immediately after a page becomes stale, it is still better than just keeping those pages in memory until they are overwritten by another page. Support for the swap discard mechanism was added in compcache-0.5. 

In general, the discard request comes a long time after a page has become stale. Consider a case where a memory-intensive workload terminates and there is no further swapping activity. In those cases, ramzswap will end up having lots of stale pages. No discard requests will come to ramzswap since no further swap allocations are being done. Once swapping activity starts again, it is expected that discard requests will be received for some of these stale pages. So, to make ramzswap more effective, changes are required in the kernel (not yet done) to scan the swap bitmap more aggressively to find any freed swap clusters -- at least in the case of RAM backed swap devices. Also, an adaptive compressed cache resizing policy would be useful -- monitor accesses to the compressed cache and move relatively unused pages to a physical swap device. Currently, ramzswap can simply forward uncompressible pages to a backing swap disk, but it cannot swap out memory allocated by xvmalloc. 

Another interesting sub-project is the [SwapReplay](<http://code.google.com/p/compcache/wiki/SwapReplay>) infrastructure. This tool is meant to easily test memory allocator behavior under actual swapping conditions. It is a kernel module and a set of userspace tools to replay swap events in userspace. The kernel module stacks a pseudo block device (/dev/sr_relay) over a physical swap device. When kernel swaps over this pseudo device, it dumps a <sector number, R/W bit, compress length> tuple to userspace and then forwards the I/O request to the backing swap device (provided as a swap_replay module parameter). This data can then be parsed using a parser library which provides a callback interface for swap events. Clients using this library can provide any action for these events -- show compressed length histograms, simulate ramzswap behavior etc. No kernel patching is required for this functionality. 

The swap replay infrastructure has been very useful throughout ramzswap development. The ability to replay swap traces allows for easy and consistent simulation of any workload without the need to set it up and run it again and again. So, if a user is suffering from high memory fragmentation under some workloads, he could simply send me swap trace for his workload and I have all the data needed to reproduce the condition on my side -- without the need to set up the same workload. 

Clients for the parser library were written to simulate ramzswap behavior over traces from a variety of workloads leading to easier evaluation of different memory allocators and, ultimately, development and enhancement of the xvmalloc allocator. In the future, it will also help testing variety of eviction policies to support adaptive compressed cache resizing. 

#### Conclusion

The compcache project is currently under active development; some of the additional features planned are: adaptive compression cache resizing, allow swapping of xvmalloc memory to physical swap disk, memory defragmentation by relocating compressed chunks within memory and compressed swapping to disk (4-5 pages swapped out with single disk I/O). Later, it might be extended to compress page-cache pages too (as earlier patches did) -- for now, it just includes the ramzswap component to handle anonymous memory compression. 

Last time the ramzswap patches were submitted for review, only LTSP performance data was provided as a justification for this feature. Andrew Morton was [not satisfied](<http://lkml.org/lkml/2009/4/1/514>) with this data. However, now there is a lot more data uploaded to the performance page on the project wiki that shows performance improvements with ramzswap. Andrew also pointed out lack of data for cases where ramzswap can cause performance _loss_ : 

We would also be interested in seeing the performance _loss_ from these patches. There must be some cost somewhere. Find a worstish-case test case and run it and include its results in the changelog too, so we better understand the tradeoffs involved here. 

The project still lacks data for such cases. However, it should be available by the 2.6.32 time frame, when these patches will be posted again for possible inclusion in mainline.
