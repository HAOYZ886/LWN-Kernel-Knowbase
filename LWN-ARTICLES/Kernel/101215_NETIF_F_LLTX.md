---
title: NETIF_F_LLTX
url: https://lwn.net/Articles/101215/
date: "September 8, 2004"
category: "NETIF F LLTX; Networking-hard start xmit locking"
author: 
---

One of the key network driver methods is called `hard_start_xmit()`; its job is to put a network packet onto the wire (or, at least, queue it for transmission). The networking subsystem protects calls to this method with a lock (`xmit_lock`) in the `net_device` structure so that only one call will be happening at any given time. This lock also protects a few configuration operations. 

As it turns out, quite a few network drivers implement their own locking internally as well. There are contexts (such as in interrupt handlers) where the `xmit_lock` will not be held, so some other provision must be made for mutual exclusion. So the `hard_start_xmit()` method, in those drivers, is called with a redundant lock held. It all works, but it adds overhead to a performance-critical path. 

Andi Kleen has put together [a patch](<https://lwn.net/Articles/101119/>) which addresses this duplicate locking. With this patch (which appears likely to be merged), drivers which do their own transmit locking can set the `NETIF_F_LLTX` "feature" flag. When a packet is to be handed to an interface with that flag set, no additional locking is performed by the networking code. As an added feature, the driver can attempt to take its internal lock with `spin_trylock()`, and immediately return `-1` if that attempt fails; the networking subsystem will then retry the transmission later. In this way, the driver can avoid stalling the CPU while waiting for the lock; there should be, after all, no slowdown if the packet is added to the transmission ring a little bit later.
