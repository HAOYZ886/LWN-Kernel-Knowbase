---
title: A summary of 2.6.22 internal API changes
url: https://lwn.net/Articles/239336/
date: "June 24, 2007"
category: "Releases-2.6.22"
author: 
---

The 2.6.22 development cycle is slowly heading toward its conclusion, meaning that it should be safe to try to list the significant internal API changes made this time around. They include: 

  * The mac80211 (formerly "Devicescape") wireless stack has been merged, creating a whole new API for the creation of wireless drivers, especially those requiring software MAC support. 

  * The `eth_type_trans()` function now sets the `skb->dev` field, consistent with how similar functions for other link types operate. As a result, many Ethernet drivers have been changed to remove the (now) redundant assignment. 

  * The header fields in the `sk_buff` structure have been renamed and are no longer unions. Networking code and drivers can now just use `skb->transport_header`, `skb->network_header`, and `skb->skb_mac_header`. There are new functions for finding specific headers within packets: `tcp_hdr()`, `udp_hdr()`, `ipip_hdr()`, and `ipipv6_hdr()`. 

  * Also in the networking area: the packet scheduler has been reworked to use `ktime` values rather than `jiffies`. 

  * The i2c layer has seen significant new changes meant to make i2c drivers look more like drivers for other buses. There are, for example, new `probe()` and `remove()` methods for notifying devices when i2c peripherals come and go. Since i2c is not a self-describing bus, the support code still needs help to know where i2c devices might be; for many classes of device, this information can be had from the system BIOS. 

  * The crypto API has a new set of functions for use with asynchronous block ciphers. There is also a new `cryptd` kernel thread which can run any synchronous cipher in an asynchronous mode. 

  * The `subsystem` structure has been removed from the Linux device model; there never really was any need for it. Most code which was expecting a `struct subsystem` argument has been changed to use the relevant `kset` instead. 

  * There is a new version of the in-kernel rpcbind (portmapper) client which supports versions 2-4 of the rpcbind protocol. The portmapper API has changed as a result. 

  * Numerous changes to the `paravirt_ops` methods have been made. Additionally, `paravirt_ops` is no longer a GPL-only export. 

  * There is a new memory function: 

```
void *krealloc(const void *p, size_t new_size, gfp_t flags);
```

As one would expect, it changes the size of the allocated memory, moving it if need be. 

  * The [SLUB allocator](<http://lwn.net/Articles/229984/>) has been merged as an experimental (for now) alternative to the slab code. The SLUB API generally matches slab, but the handling of zero-length allocations [has changed](<http://lwn.net/Articles/236920/>) somewhat. 

  * A new macro has been added to make the creation of slab caches easier: 

```
struct kmem_cache KMEM_CACHE(struct-type, flags);
```

The result is the creation of a cache holding objects of the given `struct_type`, named after that type, and with the additional slab `flags` (if any). 

  * The `SLAB_DEBUG_INITIAL` flag has been removed, along with the associated `SLAB_CTOR_VERIFY` flag passed to constructors. The result is a set of changes which ripples through quite a few source files. The unused `SLAB_CTOR_ATOMIC` flag is also gone. 

  * The SuperH architecture has working kgdb support again. 

  * The ia64 architecture has a new tool which will inject machine check errors into a running system. Not recommended for production machines. 

  * The [deferrable timers patch](<http://lwn.net/Articles/228143/>) has been merged. There is also a new macro for initializing workqueue entries (`INIT_DELAYED_WORK_DEFERRABLE()`) which causes the job to be queued in a deferrable manner. 

  * The old `SA_`* interrupt flags have not been removed as originally scheduled, but their use will now generate warnings at compile time. 

  * There is a new `list_first_entry()` macro which, surprisingly, gets the first entry from a list. 

  * The `atomic64_t` and `local_t` types are now fully supported on a wider set of architectures. 

  * Workqueues have been reworked again. There is a new function: 

```
void cancel_work_sync(struct work_struct *work);
```

This function tries to cancel a single workqueue entry, be it on the shared (keventd) or a private workqueue. Meanwhile `run_scheduled_work()` has been removed. 

The [LWN 2.6 API changes page](<https://lwn.net/Articles/2.6-kernel-api/>) is an ongoing list of API changes in the 2.6 development series.
