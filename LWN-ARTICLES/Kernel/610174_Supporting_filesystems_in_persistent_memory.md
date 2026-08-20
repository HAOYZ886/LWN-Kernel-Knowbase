---
title: Supporting filesystems in persistent memory
url: https://lwn.net/Articles/610174/
date: "September 2, 2014"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jonathan Corbet September 2, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 2, 2014

For a few years now, we have been told that upcoming non-volatile memory (NVM) devices are going to change how we use our systems. These devices provide large amounts (possibly terabytes) of memory that is persistent and that can be accessed at RAM speeds. Just what we will do with so much persistent memory is not entirely clear, but it is starting to come into focus. It seems that we'll run ordinary filesystems on it — but those filesystems will have to be tweaked to allow users to get full performance from NVM. 

It is easy enough to wrap a block device driver around an NVM device and make it look like any other storage device. Doing so, though, forces all data on that device to be copied to and from the kernel's page cache. Given that the data could be accessed directly, this copying is inefficient at best. Performance-conscious users would rather avoid use of the page cache whenever possible so that they can get full-speed performance out of NVM devices. 

The kernel has actually had some support for direct access to non-volatile memory since 2005, when [execute-in-place (XIP)](<https://lwn.net/Articles/135472/>) support was added to the ext2 filesystem. This code allows files from a directly-addressable device to be mapped into user space, allowing file data to be accessed without going through the page cache. The XIP code has apparently seen little use, though, and has not been improved in some years; it does not work with current filesystems. 

Last year, Matthew Wilcox began work on improving the XIP code and integrating it into the ext4 filesystem. Along the way, he found that it was not well suited to the needs of contemporary filesystems; there are a number of unpleasant race conditions in the code as well. So over time, his work shifted from enhancing XIP to replacing it. That work, currently [a 21-part patch set](<https://lwn.net/Articles/609652/>), is getting closer to being ready for merging into the mainline, so it is beginning to get a bit more attention. 

Those patches replace the XIP code with a new subsystem called DAX (for "direct access," apparently). At the block device level, it replaces the existing `direct_access()` function in `struct block_device_operations` with one that looks like this: 

```
long (*direct_access)(struct block_device *dev, sector_t sector,
    			  void **addr, unsigned long *pfn, long size);
```

This function accepts a `sector` number and a `size` value saying how many bytes the caller wishes to access. If the given space is directly addressable, the base (kernel) address should be returned in `addr` and the appropriate page frame number goes into `pfn`. The page frame number is meant to be used in page tables when arranging direct user-space access to the memory. 

The use of page frame numbers and addresses may seem a bit strange; most of the kernel deals with memory at this level via `struct page`. That cannot be done here, though, for one simple reason: non-volatile memory is not ordinary RAM and has no `page` structures associated with it. Those missing `page` structures have a number of consequences; perhaps most significant is the fact that NVM cannot be passed to other devices for DMA operations. That rules out, for example, zero-copy network I/O to or from a file stored on an NVM device. Boaz Harrosh is working on [a patch set](<https://lwn.net/Articles/608538/>) allowing `page` structures to be used with NVM, but that work is in a relatively early state. 

Moving up the stack, quite a bit of effort has gone into pushing NVM support into the virtual filesystem layer so that it can be used by all filesystems. Various generic helpers have been set up for common operations (reading, writing, truncating, memory-mapping, etc.). For the most part, the filesystem need only mark DAX-capable inodes with the new `S_DAX` flag and call the helper functions in the right places; see [the documentation in the patch set](<https://lwn.net/Articles/610316/>) for (a little) more information. The patch set finishes by adding the requisite support to ext4. 

Andrew Morton [expressed some skepticism](<https://lwn.net/Articles/610182/>) about this work, though. At the top of his list of questions was: why not use a ""suitably modified"" version of an in-memory filesystem (ramfs or tmpfs, for example) instead? It seems like a reasonable question; those filesystems are already designed for directly-addressable memory and have the necessary optimizations. But RAM-based filesystems are designed for RAM; it turns out that they are not all that well suited to the NVM case. 

For the details of why that is, [this message from Dave Chinner](<https://lwn.net/Articles/610184/>) is well worth reading in its entirety. To a great extent, it comes down to this: the RAM-based filesystems have not been designed to deal with persistence. They start fresh at each boot and need never cope with something left over from a previous run of the system. Data stored in NVM, instead, is expected to persist over reboots, be robust in the face of crashes, not go away when the kernel is upgraded, etc. That adds a whole set of requirements that RAM-based filesystems do not have to satisfy. 

So, for example, NVM filesystems need all the tools that traditional filesystems have to recognize filesystems on disk, check them, deal with corruption, etc. They need all of the techniques used by filesystems to ensure that the filesystem image in persistent storage is in a consistent state at all times; metadata operations must be carefully ordered and protected with barriers, for example. Since compatibility with different kernels is important, no in-kernel data structures can be directly stored in the filesystem; they must be translated to and from an on-disk format. Ordinary filesystems do these things; RAM-based filesystems do not. 

Then, as Dave explained, there is the little issue of scalability: 

Further, it's going to need to scale to very large amounts of storage. We're talking about machines with *tens of TB* of NVDIMM capacity in the immediate future and so free space management and concurrency of allocation and freeing of used space is going to be fundamental to the performance of the persistent NVRAM filesystem. So, you end up with block/allocation groups to subdivide the space. Looking a lot like ext4 or XFS at this point. 

And now you have to scale to indexing tens of millions of everything. At least tens of millions - hundreds of millions to billions is more likely, because storing tens of terabytes of small files is going to require indexing billions of files. And because there is no performance penalty for doing this, people will use the filesystem as a great big database. So now you have to have a scalable posix compatible directory structures, scalable freespace indexation, dynamic, scalable inode allocation, freeing, etc. Oh, and it also needs to be highly concurrent to handle machines with hundreds of CPU cores. 

Dave concluded by pointing out that the kernel already has a couple of "persistent storage implementations" that can handle those needs: the XFS and ext4 filesystems (though he couldn't resist poking at the scalability of ext4). Both of them will work now on a block device based on persistent memory. The biggest thing that is missing is a way to allow users to directly address all of that data without copying it through the page cache; that is what the DAX code is meant to provide. 

There are groups working on filesystems designed for NVM from the beginning. But most of that work is in an early stage; none has been posted to the kernel mailing lists, much less proposed for merging. So users wanting to get full performance out of NVM will find little help in that direction for some years yet. It is thus not unreasonable to conclude that there will be some real demand for the ability to use today's filesystems with NVM systems. 

The path toward that capability would appear to be DAX. All that is needed is to get the patch set reviewed to the point that the relevant subsystem maintainers are comfortable merging it. That review has been somewhat slow in coming; the patch set is complex and touches a number of different subsystems. Still, the code has changed considerably in response to the reviews that have come in and appears to be getting close to a final state. Perhaps this functionality will find its way into the mainline in a near-future development cycle.
