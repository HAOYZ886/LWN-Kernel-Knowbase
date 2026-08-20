---
title: The chained scatterlist API
url: https://lwn.net/Articles/256368/
date: "October 29, 2007"
category: Direct memory access; Scattergather chaining
author: "By Jonathan Corbet October 29, 2007"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 29, 2007

When asked which of the changes in 2.6.24 was most likely to create problems, an informed observer might well point at the i386/x86_64 merger. As it happens, that large patch set has gone in with relatively few hitches, but a rather smaller change has created quite a bit of fallout. The change in question is the updated API for the management of scatterlists, which are used in scatter/gather I/O. This work broke a number of in-tree drivers, so it seems likely to affect a lot of out-of-tree code as well. 

Scatter/gather I/O allows the system to perform DMA I/O operations on buffers which are scattered throughout physical memory. Consider, for example, the case of a large (multi-page) buffer created in user space. The application sees a continuous range of virtual addresses, but the physical pages behind those addresses will almost certainly not be adjacent to each other. If that buffer is to be written to a device in a single I/O operation, one of two things must be done: (1) the data must be copied into a physically-contiguous buffer, or (2) the device must be able to work with a list of physical addresses and lengths, grabbing the right amount of data from each segment. Scatter/gather I/O, by eliminating the need to copy data into contiguous buffers, can greatly increase the efficiency of I/O operations while simultaneously getting around the problem that the creation of large, physically-contiguous buffers can be problematic in the first place. 

Within the kernel, a buffer to be used in a scatter/gather DMA operation is represented by an array of one or more `scatterlist` structures, defined in `<linux/scatterlist.h>`. This array has traditionally been constrained to fit within a single page, which imposes a maximum length on scatter/gather operations. That limit has proved to be a bottleneck on high-end systems, which could otherwise benefit from transferring very large buffers (usually to and from disk devices). As a result, there has been a search for ways to get around that limit; the large block size patches which occasionally surface on the mailing lists are one approach. But the solution which has made it into the 2.6.24 kernel is to remove the limit on the length of scatter/gather lists by allowing them to be chained. 

A chained scatter/gather list can be made up of more than one page, and those pages, too, are likely to be scattered throughout physical memory. When this chaining is done, a couple of low-order bits in the buffer pointer are used to mark chain entries and the end of the list. This usage is not something which driver code needs to worry about, but the existence of special bits and chain pointers forces some changes to how drivers work with scatterlists. 

Drivers which do not perform chaining will allocate their `scatterlist` arrays in the usual way - usually through a call to `kcalloc()` or some such. Prior to 2.6.23, there was no initialization step required, beyond, perhaps, zeroing the entire array. That has changed, however; drivers should now initialize a `scatterlist` array with: 

```
void sg_init_table(struct scatterlist *sg, unsigned int nents);
```

Here, `sg` points to the allocated array, and `nents` is the number of allocated scatter/gather entries. 

As before, a driver should loop through the segments of the buffer, setting one `scatterlist` entry for each. It is no longer possible to set the `page` pointer directly, however: that pointer does not exist in 2.6.24. Instead, the usual way to set a scatterlist entry will be with one of: 

```
void sg_set_page(struct scatterlist *sg, struct page *page,
    		     unsigned int len, unsigned int offset);
    
        void sg_set_buf(struct scatterlist *sg, const void *buf,
    	      	    unsigned int buflen);
```

2.6.24 scatterlists also require that the end of the list be explicitly marked. This marking is performed when `sg_init_table()` is called, so drivers will not normally have to mark the end explicitly. Should the I/O operation not use all of the entries which were allocated in the list, though, the driver should mark the final segment with: 

```
void sg_mark_end(struct scatterlist *sg, unsigned int nents);
```

Where `nents` is the number of valid entries in the scatterlist. 

After the scatterlist has been mapped (with a function like `dma_map_sg()`), the driver will need to program the resulting DMA addresses into the hardware. The old approach of just stepping through the array will no longer work; instead, a driver should move on to the next entry in a scatterlist with: 

```
struct scatterlist *sg_next(struct scatterlist *sg);
```

The return value will be the next entry to process - or `NULL` if the end of the list has been reached. There is also a `for_each_sg()` macro which can be used to iterate through an entire scatterlist; it will typically be used in code which looks like: 

```
int i;
        struct scatterlist *list, *sgentry;
    
        /* Fill in list and pass it to dma_map_sg().  Then... */
        for_each_sg(i, list, sgentry, nentries) {
    	program_hw(device, sg_dma_address(sgentry), sg_dma_len(sgentry));
        }
```

Drivers which wish to take advantage of the chaining feature must do just a little more work. Each piece of the scatterlist must be allocated independently, then those pieces must be chained together with: 

```
void sg_chain(struct scatterlist *prv, unsigned int prv_nents,
    		  struct scatterlist *next);
```

This call turns the scatterlist entry `prv[nents]` into a chain link to `next`. If the chaining is done while the list is being filled, `prv` should have no more than `prv_nents-1` segments stored into it. Alternatively, a driver can chain together the pieces of the list ahead of time (remembering to allocate one entry for each chain link), then use `sg_next()` to fill the list without the need to worry about where the chain links are. 

As of this writing, this API is still evolving in response to issues which have come up with in-tree drivers. It seems unlikely that any more substantial changes will be made before the 2.6.24 release, but surprises are always possible.
