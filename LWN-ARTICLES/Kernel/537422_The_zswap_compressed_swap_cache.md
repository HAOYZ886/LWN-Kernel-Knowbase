---
title: The zswap compressed swap cache
url: https://lwn.net/Articles/537422/
date: "February 12, 2013"
category: "Memory management-Swapping; Transcendent memory; zswap"
author: "February 12, 2013 This article was contributed by Seth Jennings"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

February 12, 2013

This article was contributed by Seth Jennings

Swapping is one of the biggest threats to performance. The latency gap between RAM and swap, even on a fast SSD, can be four orders of magnitude. The throughput gap is two orders of magnitude. In addition to the speed gap, storage on which a swap area resides is becoming more shared and virtualized, which can cause additional I/O latency and nondeterministic workload performance. The zswap subsystem exists to mitigate these undesirable effects of swapping through a reduction in I/O activity. 

Zswap is a lightweight, write-behind compressed cache for swap pages. It takes pages that are in the process of being swapped out and attempts to compress them into a dynamically allocated RAM-based memory pool. If this process is successful, the writeback to the swap device is deferred and, in many cases, avoided completely. This results in a significant I/O reduction and performance gains for systems that are swapping. 

#### Zswap basics

Zswap intercepts pages in the middle of swap writeback and caches them using the frontswap API. Frontswap has been in the kernel since v3.5 and has been [covered by LWN](<https://lwn.net/Articles/386090/>) before. It allows a backend driver, like zswap, to intercept both swap page writeback and the page faults for swapped out pages. Zswap also makes use of the "zsmalloc" allocator (discussed below) for compressed page storage. 

Zswap seeks to be as simple as possible in its structure and operation. There are two primary data structures. The first is the `zswap_entry` structure, which contains information about a single compressed page stored in zswap: 

```
struct zswap_entry {
    	struct rb_node rbnode;
    	int refcount;
    	pgoff_t offset;
    	unsigned long handle; /* zsmalloc allocation */
    	unsigned int length;
    	/* ... */
        };
```

The second is the `zswap_tree` structure which contains a red-black tree of zswap entries indexed by the `offset` value: 

```
struct zswap_tree {
    	struct rb_root rbroot;
    	struct list_head lru;
    	spinlock_t lock;
    	struct zs_pool *pool;
        };
```

At the highest level, there is an array of `zswap_tree` structures indexed by the swap device number. 

There is a single lock per `zswap_tree` to protect the tree structure during lookups and modifications. The higher-level swap code provides certain protections that simplify the zswap implementation by not having to design for concurrent store, load, and invalidate operations on the same swap entry. While this single-lock design might seem like a likely source for contention, actual execution [demonstrates](<https://lkml.org/lkml/2013/1/8/269/>) that the swap path is largely bottlenecked by other locks at higher levels, such as the `anon_vma` mutex or `swap_lock`. In comparison, the `zswap_tree` lock is very lightly contended. Writeback support, covered in the next section, also led to this single-lock design. 

For page compression, zswap uses compressor modules provided by the kernel's cryptographic API. This allows users to select the compressor dynamically at boot time, and gives easy access to hardware compression accelerators or any other future compression engines. 

A zswap store operation occurs when a page is selected for swapping by the reclaim system and frontswap intercepts the page in `swap_writepage()`. The operation begins by compressing the page into a per-CPU temporary buffer. Compressing into the temporary buffer is required because the compressed size, and thus the size of the permanent allocation needed to hold it, isn't known until the compression is actually done. Once the compressed size is known, an object is allocated and the temporary buffer is copied into the object. Lastly, a `zswap_entry` structure is allocated, populated, and inserted into the tree for that swap device. 

If the store fails for any reason, most likely because of an object allocation failure, zswap returns an error which is propagated up through frontswap into `swap_writepage()`. The page is then swapped out to the swap device as usual. 

A load operation occurs when a program page faults on a page table entry (PTE) that contains a swap entry and is intercepted by frontswap in `swap_readpage()`. The swap entry contains the device and offset information needed to look up the zswap entry in the appropriate tree. Once the entry is located, the data is decompressed directly into the page allocated by the page fault code. The entry is not removed from the tree during a load; it remains up-to-date until the entry is invalidated. 

An invalidate operation occurs when the reference count for a particular swap offset becomes zero in `swap_entry_free()`. In this case, the zswap entry is removed from the appropriate tree, and the entry and the zsmalloc allocation that it references are freed. 

To be preemption-friendly, interrupts are never disabled. Preemption is only disabled during compression while accessing the per-cpu temporary buffer page, and during decompression while accessing a mapped zsmalloc allocation. 

#### Zswap writeback

To operate optimally as a cache, zswap should hold the most recently used pages. With frontswap, there is, unfortunately, a real potential for an inverse least recently used (LRU) condition in which the cache fills with older pages, and newer pages are forced out to the slower swap device. To address this, zswap is designed with "resumed" writeback in mind. 

As background, the process for swapping pages follows these steps: 

  1. First, an anonymous memory page is selected for swapping and a slot is allocated in the swap device. 

  2. Next, the page is unmapped from all processes using that page. The PTEs referencing that page are filled with the swap entry that consists of the swap type and offset where the page can be found. 

  3. Lastly, the page is scheduled for writeback to the swap device. 

When `frontswap_store()` in `swap_writepage()` is successful, the writeback step is not performed. However, the slot in the swap device has been allocated and is still reserved for the page even though the page only resides in the frontswap backend. Resumed writeback in zswap forces pages out of the compressed cache into their previously reserved swap slots in the swap device. Currently, the policy is basic and forces pages out from the cache in two cases: (1) when the cache has reached its maximum size according to the `max_pool_percent` sysfs tunable or, (2) when zswap is unable to allocate new space for the compressed pool. 

During resumed writeback, zswap decompresses the page, adds it back to the swap cache, and schedules writeback into the swap slot that was previously reserved. By splitting `swap_writepage()` into two functions after `frontswap_store()` is called, zswap can resume writeback from the point where the initial writeback terminated in frontswap. The new function is called `__swap_writepage()`. 

Freeing zswap entries becomes more complex with writeback. Without writeback, pages would only be freed during invalidate operations (`zswap_frontswap_invalidate page()`). With writeback, pages can also be freed in `zswap_writeback_pages()`. These invalidate and writeback functions can run concurrently for the same zswap entry. To guarantee that entries are not freed while being accessed by another thread, a reference count field (called `refcount`) is used the zswap_entry structure. 

#### Zsmalloc rationale

One really can't talk about zswap without mentioning zsmalloc, the allocator it uses for compressed page storage, which currently resides in the Linux Staging tree. 

Zsmalloc is a slab-based allocator used by zswap; it provides more reliable allocation of large objects in a memory constrained environment than does the kernel slab allocator. Zsmalloc has already been [discussed on LWN](<https://lwn.net/Articles/477067/>), so this section will focus more on the need for zsmalloc in the presence of the kernel slab allocator. 

The objects that zswap stores are compressed pages. The default compressor is lzo1x-1, which is known for speed, but not so much for high compression. As a result, zswap objects can frequently be large relative to typical slab objects (>1/8th `PAGE_SIZE`). This is a problem for the kernel slab allocator under memory pressure. 

The kernel slab allocator requires high-order page allocations to back slabs for large objects. For example, on a system with a 4K page size, the `kmalloc-512` cache has slabs that are backed by two contiguous pages. `kmalloc-2048` requires eight contiguous pages per slab. These high-order page allocations are very likely to fail when the system is under memory pressure. 

Zsmalloc addresses this problem by allowing the pages backing a slab (or “size class” in zsmalloc terms) to be both non-contiguous and variable in number. They are variable in number because zsmalloc allows a slab to be composed of less than the target number of backing pages. A set of non-contiguous pages backing a slab are stitched together using fields of `struct page` to create a “zspage”. This allows zsmalloc to service large object allocations, up to `PAGE_SIZE`, without requiring high-order page allocations. 

Additionally, the kernel slab allocator does not allow objects that are less than a page in size to span a page boundary. This means that if an object is `PAGE_SIZE/2 + 1` bytes in size, it effectively uses an entire page, resulting in ~50% waste. Hence there are no `kmalloc()` cache sizes between `PAGE_SIZE/2` and `PAGE_SIZE`. Zswap frequently needs allocations in this range, however. Using the kernel slab allocator causes the memory savings achieved through compression to be lost in fragmentation. 

In order to satisfy these larger allocations while not wasting an entire page, zsmalloc allows objects to span page boundaries at the cost of having to map the allocations before accessing them. This mapping is needed because the object might be contained in two non-contiguous pages. For example, in a zsmalloc size class for objects that are 2/3 of PAGE_SIZE, three objects could be stored in a zspage with two non-contiguous backing pages with no waste. The object stored in the second of the three object positions in the zspage would be split between two different pages. 

Zsmalloc is a good fit for zswap. Zswap was evaluated using the kernel slab allocator and these issues did have a significant impact on the `frontswap_store()` success rate. This was due to `kmalloc()` allocation failures and a need to reject pages that compressed to sizes greater than `PAGE_SIZE/2`. 

#### Performance

In order to produce a performance comparison, kernel builds were conducted with an increasing number of threads per run in a constant and constrained amount of memory. The results indicate a runtime reduction of 53% and an I/O reduction of 76% with zswap compared to normal swapping. The testing system was configured with: 

  * Gentoo running v3.7-rc7 
  * Quad-core i5-2500 @ 3.3GHz 
  * 512MB DDR3 1600MHz (limited with mem=512m on boot) 
  * Filesystem and swap on 80GB HDD (about 58MB/s with hdparm -t) 

The table below summarizes the test runs. 

> | Baseline| zswap| Change  
> ---|---|---|---  
> N| pswpin| pswpout| majflt| I/O sum| pswpin| pswpout| majflt| I/O sum| %I/O| MB  
> 8| 1| 335| 291| 627| 0| 0| 249| 249| -60%| 1  
> 12| 3688| 14315| 5290| 23293| 123| 860| 5954| 6937| -70%| 64  
> 16| 12711| 46179| 16803| 75693| 2936| 7390| 46092| 56418| -25%| 75  
> 20| 42178| 133781| 49898| 225857| 9460| 28382| 92951| 130793| -42%| 371  
> 24| 96079| 357280| 105242| 558601| 7719| 18484| 109309| 135512| -76%| 1653  
  
The 'N' column indicates the maximum number of concurrent threads for the kernel build (`make -jN`) for each run. The next four columns are the statistics for the baseline run without zswap, followed by the same for the zswap run. The I/O sum column for each run is a sum of pswpin (pages swapped in), pswpout (pages swapped out), and majflt (major page faults). The difference between the baseline and zswap runs is shown both in relative terms, as a percentage of I/O reduction, and in absolute terms, as a reduction of X megabytes of I/O related to swapping activity. 

A compressed swap cache reduces the efficiency of the page reclaim process. For any store operation, the cache may allocate some pages to store the compressed page. This results in an reduction of overall page reclaim efficiency. This reduction in efficiency results in additional shrinking pressure on the page cache causing an increase in major page faults where pages must be re-read from disk. In order to have a complete picture of the I/O impact, the major page faults must be considered in the sum of I/O. 

The next table shows the total runtime of the kernel builds: 

> Runtime (in seconds)  
> ---  
> N| base| zswap| %change  
> 8| 107| 107| 0%  
> 12| 128| 110| -14%  
> 16| 191| 179| -6%  
> 20| 371| 240| -35%  
> 24| 570| 267| -53%  
  
The runtime impact of swap activity is decreased when comparing runs with the same number of threads. The rate of degradation is reduced for increasingly constrained runs when comparing baseline and zswap. 

The measurements of average CPU utilization during the builds are: 

> %CPU utilization (out of 400% on 4 cpus)  
> ---  
> N| base| zswap| %change  
> 8| 317| 319| 1%  
> 12| 267| 311| 16%  
> 16| 179| 191| 7%  
> 20| 94| 143| 52%  
> 24| 60| 128| 113%  
  
The CPU utilization table shows that with zswap, the kernel build is able to make more productive use of the CPUs, as is expected from the runtime results. 

Additional performance testing was performed using SPECjbb. Metrics regarding the performance improvements and I/O reductions that can be achieved using zswap on both x86 and Power7+ (with and without hardware compression acceleration), can be found on [this page](<http://ibm.co/WUnOkG>). 

#### Conclusion

Zswap is a compressed swap cache, able to evict pages from the compressed cache, on an LRU basis, to the backing swap device when the compressed pool reaches it size limit or the pool is unable to obtain additional pages from the buddy allocator. Its approach trades CPU cycles for reduced swap I/O. This trade-off can result in a significant performance improvement as reads to and writes from to the compressed cache are almost always faster that reading from a swap device which incurs the latency of an asynchronous block I/O read.
