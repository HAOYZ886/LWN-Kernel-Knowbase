---
title: Optimizing VMA caching
url: https://lwn.net/Articles/589475/
date: "March 5, 2014"
category: Memory management
author: "By Jonathan Corbet March 5, 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 5, 2014

The kernel divides each process's address space into virtual memory areas (VMAs), each of which describes where the associated range of addresses has its backing store, its protections, and more. A mapping created by `mmap()`, for example, will be represented by a single VMA, while mapping an executable file into memory may require several VMAs; the list of VMAs for any process can be seen by looking at `/proc/_PID_ /maps`. Finding the VMA associated with a specific virtual address is a common operation in the memory management subsystem; it must be done for every page fault, for example. It is thus not surprising that this mapping is highly optimized; what may be surprising is the fact that it can be optimized further. 

The VMAs for each address space are stored in a [red-black tree](<https://lwn.net/Articles/184495/>), which enables a specific VMA to be looked up in logarithmic time. These trees scale well, which is important; some processes can have hundreds of VMAs (or more) to sort through. But it still takes time to walk down to a leaf in a red-black tree; it would be nice to avoid that work at least occasionally if it were possible. Current kernels work toward that goal by caching the results of the last VMA lookup in each address space. For workloads with any sort of locality, this simple cache can be quite effective, with hit rates of 50% or more. 

But Davidlohr Bueso thought it should be possible to do better. Last November, he posted [a patch](<https://lwn.net/Articles/572629/>) adding a second cache holding a pointer to the largest VMA in each address space. The logic was that the VMA with the most addresses would see the most lookups, and his results seemed to bear that out; with the largest-VMA cache in place, hit rates went to over 60% for some workloads. It was a good improvement, but the patch did not make it into the mainline. Looking at the discussion, one can quickly come up with a useful tip for aspiring kernel developers: if Linus [responds](<http://thread.gmane.org/gmane.linux.kernel/1589230>) by saying ""This patch makes me angry"", the chances of it being merged are relatively small. 

Linus's complaint was that caching the largest VMA seemed ""way too ad-hoc"" and wouldn't be suitable for a lot of workloads. He suggested caching a small number of recently used VMAs instead. Additionally, he noted that maintaining a single cache per address space, as current kernels do, might not be a good idea. In situations where multiple threads are running in the same address space, it is likely that each thread will be working with a different set of VMAs. So making the cache per-thread, he said, might yield much better results. 

A few iterations later, Davidlohr has posted [a VMA-caching patch set](<https://lwn.net/Articles/588807/>) that appears to be about ready to go upstream. Following Linus's suggestion, the single-VMA cache (`mmap_cache` in `struct mm_struct`) has been replaced by a small array called `vmacache` in `struct task_struct`, making it per-thread. On systems with a memory management unit (almost all systems), that array holds four entries. There are also new sequence numbers stored in both `struct mm_struct` (one per address space) and in `struct task_struct` (one per thread). 

The purpose of the sequence numbers is to ensure that the cache does not return stale results. Any change to the address space (the addition or removal of a VMA, for example) causes the per-address-space sequence number to be incremented. Every attempt to look up an address in the per-thread cache first checks the sequence numbers; if they do not match, the cache is deemed to be invalid and will be reset. Address-space changes are relatively rare in most workloads, so the invalidation of the cache should not happen too often. 

Every call to `find_vma()` (the function that locates the VMA for a virtual address) first does a linear search through the cache to see if the needed VMA is there. Should the VMA be found, the work is done; otherwise, a traversal of the red-black tree will be required. In this case, the result of the lookup will be stored back into the cache. That is done by overwriting the entry indexed by the lowest bits of the page-frame number associated with the original virtual address. It is, thus, a random replacement policy for all practical purposes. The caching mechanism is meant to be fast so there would probably be no benefit from trying to implement a more elaborate replacement policy. 

How well does the new scheme work? It depends on the workload, of course. For system boot, where almost everything running is single-threaded, Davidlohr reports that the cache hit rate went from 51% to 73%. Kernel builds, unsurprisingly, already work quite well with the current scheme with a hit rate of 75%, but, even in this case, improvement is possible: that rate goes to 88% with Davidlohr's patch applied. The real benefit, though, can be seen with benchmarks like [ebizzy](<http://sourceforge.net/projects/ebizzy/>), which is designed to simulate a multithreaded web server workload. Current kernels find a cached VMA in a mere 1% of lookup attempts; patched kernels, instead, show a 99.97% hit rate. 

With numbers like that, it is hard to find arguments for keeping this patch out of the mainline. At this point, the stream of suggestions and comments has come to a halt. Barring surprises, a new VMA lookup caching mechanism seems likely to find its way into the 3.15 kernel.
