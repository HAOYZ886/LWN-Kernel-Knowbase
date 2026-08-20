---
title: Transcendent memory
url: https://lwn.net/Articles/340080/
date: "July 8, 2009"
category: "Memory management-Virtualization; Transcendent memory"
author: "By Jonathan Corbet July 8, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 8, 2009

Making the best use of available memory is one of the biggest challenges for any operating system. Throwing virtualization into the mix adds both new challenges (balancing memory use between guests, for example) and opportunities (sharing pages between guests). Developers have responded with technologies like hot-plug memory and [KSM](<http://lwn.net/Articles/330589/>), but nobody seems to think that the problem is fully solved. Transcendent memory is a new memory-management technique which, it is hoped, will improve the system's use of scarce RAM, regardless of whether virtualization is being used. 

In his [linux-kernel introduction](<http://lwn.net/Articles/338098/>), Dan Magenheimer asks: 

What if there was a class of memory that is of unknown and dynamically variable size, is addressable only indirectly by the kernel, can be configured either as persistent or as "ephemeral" (meaning it will be around for awhile, but might disappear without warning), and is still fast enough to be synchronously accessible? 

Dan (along with a list of other kernel developers) is exploring this concept, which he calls "transcendental memory." In short, transcendental memory can be thought of as a sort of RAM disk with some interesting characteristics: nobody knows how big it is, writes to the disk may not succeed, and, potentially, data written to the disk may vanish before being read back again. At a first blush, it may seem like a relatively useless sort of device, but it is hoped that transcendental memory will be able to improve performance in a few situations. 

There is [an API specification [PDF]](<http://oss.oracle.com/projects/tmem/dist/documentation/api/tmemspec-v001.pdf>) available; there is also a related C API found in the patch itself. This discussion will focus on the latter, which suffers from less EXCESSIVE CAPITAL USE and is generally easier to understand. 

Transcendental memory operates on the concept of page pools; once a pool is created, data can be stored to pages within the pool. The calls for creating and destroying pools look like this: 

```
u32 pool_id = tmem_new_pool(struct tmem_pool_uuid uuid, u32 flags)
        tmem_destroy_pool(u32 pool_id);
```

Pools are identified by the `uuid` value, though the identification really only matters for pools which might be shared among multiple users. A fair amount of information is stored in the `flags` field, including: 

  * An "ephemeral" bit, which controls whether data successfully written to the pool is allowed to disappear at a random future time. 

  * A "shared" bit indicating whether the pool is to be shared with other users. 

  * The size of pages to use in the pool, expressed as a kernel "order" value. 

  * A specification version number, used to ensure that both sides of the conversation know how to understand each other. 

While users are expected to specify an expected page size, there is no way to specify the size of the pool as a whole. Determining the proper sizing for a pool (which almost certainly changes over time) is left to the hypervisor or whatever other software component is managing the pool. 

As suggested by the above interface, transcendental memory is very much page-based. Beyond that, it also can never be referenced directly; users are required to copy data into and out of the pool explicitly. The functions used for moving data between normal and transcendental memory are: 

```
int tmem_put_page(u32 pool_id, u64 object_id, u32 page_id, unsigned long pfn);
        int tmem_get_page(u32 pool_id, u64 object_id, u32 page_id, unsigned long pfn);
```

For both of these calls, `pool_id` specifies an existing pool. The `object_id` and `page_id` values, together, form a unique identifier for the page within the pool. If the pool is being used to cache file pages, for example, the `object_id` would identify the file, while `page_id` would be the offset within the file. `pfn` (a page frame number) identifies the page which is the source of the data (for `tmem_put_page()`) or the destination (`tmem_get_page()`). 

Note that either call might fail. Since the size of the pool is not known, callers can never know in advance whether `tmem_put_page()` will succeed. So any transcendental memory user must have a backup plan ready in case the call fails. For pools marked as "ephemeral," `tmem_get_page()` is allowed to fail even if `tmem_put_page()` on the same ID succeeded; in other words, the implementation is allowed to drop pages from ephemeral pools if it decides that the memory can be put to better use elsewhere. It's also worth noting that, with private, ephemeral pools, `tmem_get_page()` will remove the indicated page from the pool. 

As an example of how this feature might be used, consider the Linux page cache, which maintains copies of pages from disk files. When memory gets tight, the page cache will start forgetting pages which are clean, but which have not been referenced in the recent past. With transcendental memory, the page cache could, before dropping the pages, attempt to store them into an ephemeral transcendental memory pool. At some future time, when one of those pages is needed again, the page cache would first attempt to fetch it from the pool. If the `tmem_get_page()` call succeeds, a disk I/O operation will have been avoided and everybody benefits; otherwise the page is read from disk as usual. 

Persistent (non-ephemeral) pools could be used as a sort of swap device. If the swapping code succeeds in writing a page to the pool, it can avoid writing it to the real swap device. The result is saved I/O at both swap-out and swap-in times. If the pool lacks space for the swapped page, it will be written to the real swap device in the usual way. 

Meanwhile, the transcendental memory implementation can try to optimize its management of the memory pools. Guests which are more active (or which have been given a higher priority) might be allowed to allocate more pages from the pools. Duplicate pages can be coalesced; KSM-like techniques could be used, but the use of object IDs could make it easier to detect duplicates in a number of situations. And so on. 

The API specifies a number of other operations. There are a couple of calls to flush pages from the pool; one of them can remove all pages with a given object ID. Sub-page-size reads and writes are supported; there is also a `tmem_xchg()` call to atomically exchange data within a transcendental memory page. See the API specification for the full list. 

A number of concerns were raised in the subsequent discussion; as a result, the above API is likely to change a bit. The biggest concern, though, appears to be security. The potential for hostile code to tap into shared pools and read out pages has developers worried; the need to guess a 128-bit UUID first has proved not to be sufficiently reassuring. Even with legitimate users only, a shared pool has the potential to contain data which should not, in reality, be shared between guests. As a result, any transcendental memory user will have to be written to take high-level security issues into account in low-level code. 

Dan seemingly doesn't see the security problems as being as worrisome as others do. Even so, he eventually [announced](<https://lwn.net/Articles/340087/>) that the next transcendental memory patch would not include support for shared pools, and, indeed, [version 2](<https://lwn.net/Articles/340409/>) lacks that feature. That feature will probably not come back until the security issues have been thought through and the concerns have been addressed. 

Beyond that, transcendental memory will need some convincing evidence that it improves performance before it can make it into the mainline. The potential for improvements is clearly there; it is essentially a way for the system to take higher-level information into account when managing its virtual memory resources. If transcendental memory is able to fulfill that potential in a secure way, there may well be a place for it in the mainline kernel.
