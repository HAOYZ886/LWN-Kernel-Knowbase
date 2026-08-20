---
title: More IOPS with BIO caching
url: https://lwn.net/Articles/868070/
date: "September 6, 2021"
category: "Block layer-Scalability"
author: "By Jonathan Corbet September 6, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 6, 2021

Once upon a time, block storage devices were slow, to the point that they often limited the speed of the system as a whole. A great deal of effort went into carefully ordering requests to get the best performance out of the storage device; achieving that goal was well worth expending some CPU time. But then storage devices got much faster and the equation changed. Fancy I/O-scheduling mechanisms have fallen by the wayside and effort is now focused on optimizing code so that the CPU can keep up with its storage. A block-layer change that was merged for the 5.15 kernel shows the kinds of tradeoffs that must be made to get the best performance from current hardware. 

Within the block layer, an I/O operation is represented by [`struct bio`](<https://elixir.bootlin.com/linux/v5.14/source/include/linux/blk_types.h#L215>); an instance of this structure is usually just called a "BIO". Contained within a BIO are a pointer to the relevant block device, a description of the buffer(s) to be transferred, a pointer to a function to call when the operation completes, and a surprising amount of ancillary information. A BIO must be allocated, managed, and eventually freed for every I/O operation executed by the system. Given that a large, busy system with fast block devices can generate millions of I/O operations per second (IOPS), huge numbers of BIOs will be going through this life cycle in a constant stream. 

The kernel's slab allocator is optimized for the task of repeatedly allocating and freeing structures of a uniform size; it seems like it should be well suited as a source of BIOs for the block subsystem. It turns out, though, that the slab allocator is not fast enough; it has become a bottleneck slowing down block I/O. So block maintainer Jens Axboe has put together [a set of patches](<https://lwn.net/ml/linux-block/20210812154149.1061502-1-axboe@kernel.dk/>) to circumvent the problem. 

The result is a simple cache of BIO structures. It is built as a set of linked lists, one for each CPU in the system. When a new BIO is needed (and when some other conditions are met — see below), the linked list for the current CPU is checked; if a free BIO is found there, it can be removed from the list and used without having to call into the slab allocator. If the list is empty, a slab call must be made as usual, of course. When the time comes to free a BIO, it is put onto the current CPU's list. Should the list grow too large (more than 576 cached BIOs), 64 BIOs will be handed back to the slab allocator. 

It is a simple mechanism, which is the source of its speed. Rather than calling into the slab allocator, the block layer can just grab an available BIO directly off of the appropriate per-CPU list without any function calls at all. The use of a per-CPU list eliminates the need for locking, speeding things further. The lists are managed like a stack, maximizing the chance that an allocated BIO will already be present in the CPU cache. The end result is a significant improvement in performance. 

At least, that is the case for some workloads. The BIO cache is, as noted, simple; one of the things it doesn't bother with is interrupt safety. A per-CPU data structure is only safe for lockless access if the kernel cannot be preempted while executing the critical section; interrupts, being the definitive form of preemption, violate that rule. If a block-driver interrupt handler tries to allocate or free a BIO while some other kernel code is doing the same, the results are likely to be unpleasant and users will be remarkably unappreciative of the improved performance. 

The BIO cache could be made interrupt-safe, of course, and someday that might just have to happen. But disabling interrupts has a performance cost as well, so there are good reasons for avoiding it. The cost associated with leaving interrupts enabled is that the BIO cache can only be used in situations where concurrent access in interrupt handlers is not a possibility. The good news is that one such situation is well defined: when [block-layer I/O polling](<https://lwn.net/Articles/663879/>) is in use. Polling turns off interrupts from the storage device in favor of simply looping until an I/O request is completed; this can actually be a reasonable thing to do with fast devices. In settings where getting the highest I/O rates possible is important, administrators are likely to have polling enabled anyway; targeting this additional performance improvement at that use case thus makes some sense. 

Getting the slab allocator out of the loop improves performance considerably, but there is one other bottleneck to overcome. The block layer has a function called [`bio_init()`](<https://elixir.bootlin.com/linux/v5.14/source/block/bio.c#L240>), the core of which reads: 

```
memset(bio, 0, sizeof(*bio));
```

One might think that `memset()` would be the fastest way to initialize a moderately sized structure like this, but that turns out not to be the case. So Axboe added [a patch](<https://lwn.net/ml/linux-block/20210812154149.1061502-2-axboe@kernel.dk/>) that replaces the `memset()` call with a series of statements explicitly setting each BIO field to zero. The changelog notes that this change halves the time it takes to allocate and initialize a BIO (when using the BIO cache, of course). 

With these changes in place, Axboe said, the block layer's performance increased by about 10%; it can now execute over 3.5 million IOPS on each CPU core on his test system. That is a lot of blocks moving back and forth, which will surely please the managers of storage servers. This series shows what can be (and must be) done to optimize I/O throughput on current hardware; it also suggests, though, that it may be time to put some more optimization effort into the (already highly optimized) slab allocator. If the kernel starts to fill up with per-subsystem object caches as a way of bypassing the allocator, performance overall will suffer. Meanwhile, though, few are likely to argue that this effort to improve block-I/O performance is anything but well placed.
