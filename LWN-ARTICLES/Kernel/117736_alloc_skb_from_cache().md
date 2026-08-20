---
title: alloc_skb_from_cache()
url: https://lwn.net/Articles/117736/
date: "January 4, 2005"
category: Networking; Xen; alloc skb from cache
author: 
---

The post-2.6.10 mainline kernel contains a set of patches designed to help with the merging of the Xen virtual architecture. One of them is an enhancement to the networking API which could have uses beyond Xen. 

The "socket buffer" (SKB) is the core kernel data structure used to represent packets as they pass through the system. The SKB API has been described for 2.4 in [LDD2](<http://lwn.net/Kernel/LDD2/ch14.lwn#t9>); this interface has changed little since then. SKB structures are allocated in various ways by the networking layer; the Xen patches add a new way: 

```
struct sk_buff *alloc_skb_from_cache(kmem_cache_t *cache,
                                             unsigned int size, int gfp_mask);
```

This function will allocate an SKB of the given `size` from the slab `cache` provided. It assumes that the cache will provide a chunk of memory of sufficient size for the buffer - and various bits of overhead imposed by the SKB structure itself. 

The new allocation function might speed things slightly for network drivers which allocate large numbers of buffers of the same size - though the existing allocation interfaces are already pretty fast. Xen has an interesting use for this capability, however: fast networking between virtual machines. By using the slab cache, Xen can ensure that every packet is allocated a one-page buffer. When that packet is sent to another virtual machine, the associated page can be unmapped from the source system and mapped into the address space of the destination. It is, in other words, a fairly straightforward zero-copy networking scheme. As a side benefit, the Xen monitor benefits from the knowledge that the pages in question have been used for network packets - since the contents of the packet could be read by third parties while it is in transit, there is no real point in worrying about zeroing out the data afterward.
