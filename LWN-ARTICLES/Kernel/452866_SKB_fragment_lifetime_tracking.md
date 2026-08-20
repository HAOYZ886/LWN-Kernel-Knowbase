---
title: SKB fragment lifetime tracking
url: https://lwn.net/Articles/452866/
date: "July 25, 2011"
category: Networking
author: "By Jonathan Corbet July 25, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 25, 2011

In May, LWN [examined the "stable pages" patch](<https://lwn.net/Articles/442355/>), whose intent is to be sure that pages under I/O cannot be modified (by the kernel or user space) until the I/O completes. Block I/O is not the only context in which this kind of problem arises, though; memory which has been given to the network stack should also be kept stable until the transmission is complete. Unfortunately, it is hard to know when the network stack is truly done with a page, leaving the system open to possible data corruption problems. 

Ian Campbell [described how things can go wrong](<https://lwn.net/Articles/452868/>) in June. Imagine a page full of data to be written to a file on an NFS-mounted filesystem. The NFS code will put together a network I/O operation as represented by an `sk_buff` structure (an "SKB") and pass it into the network stack for transmission. Perhaps the server is slow or the network is noisy; one way or another the acknowledgment from the remote NFS server is slow in coming - slow enough that the network stack decides to retransmit the request. While the data sits in the retransmission queue (perhaps already handed to the interface driver), the ACK arrives from the server. The network stack will tell the NFS client that the operation has completed. The page used to contain the data to be written could then be rewritten with some other data - even though that retransmission is still waiting to go out. The result could be a (re)transmission of corrupted data. This problem is especially acute for `O_DIRECT` writes - where the application is waiting for the end of the operation - but it can come up in other situations as well. 

SKBs can have a destructor function, so one would think that it would be possible to just wait until the network stack finishes with the structure before releasing the relevant page(s). But the network stack works in strange and mysterious ways, and the fact that it has finished with an SKB does not imply that it is finished with the pages of data referenced by that SKB. The "cloning" of SKBs happens often in the network stack, and pages of data can actually move between SKBs. The networking code manages the page reference counts directly, so there is no danger of the data pages being put to some other use entirely by the system. But that is not helpful to higher layers, which have no way to know when it's safe to signal the completion of an operation. 

Fixing this problem requires [a significant set of changes](<https://lwn.net/Articles/452852/>) to the low-level SKB-handling code. Ian's patch series starts by defining a set of helper functions for the tracking of references to pages from SKBs. Current networking code calls `get_page()` and `put_page()` directly; after patching, all of those calls have been wrapped by functions like `skb_frag_ref()`. Quite a few changes are required to get the networking core and in-tree drivers to use these functions. 

Once that is done, the patch series introduces the concept of a "fragment destructor" for SKBs: 

```
struct skb_frag_destructor {
    	atomic_t ref;
    	int (*destroy)(void *data);
    	void *data;
        };
```

The low-level functions that add fragments to SKBs are modified to take an additional destructor argument. The destructor is always optional; code which does not need to use destructors can simply pass a null pointer instead. 

At this point, it's a relatively simple matter for the accessor functions added earlier in the series to increment and decrement the reference count whenever there are destructors present. When the reference count (`ref`) drops to zero, the provided `destroy()` function will be called. Putting the reference counter in the destructor is a useful optimization: in the absence of destructors, the overhead of maintaining the reference count can be skipped. Also worth noting is the fact that multiple fragments in an SKB can share the same destructor; in this case, the `destroy()` function will only be called when the networking code has finished with all of those fragments. 

One other optimization is that, in the presence of a destructor, the network code will no longer increment and decrement the reference counts associated with the pages in the fragments. In this situation, the calling code is assumed to hold a reference for the duration of the operation, so separate reference counting at that level is not needed. 

The final step is to make use of this capability. The internal `kernel_sendpage()` function gains an extra parameter to hold a pointer to the destructor, should the caller want to use one. The sunrpc code is changed to not signal completion of operations until the networking code indicates that it is done with the associated memory. And that solves the problem - for NFS at least; there should be no more troubles with pages being reused while they are still under network I/O. There are other places in the kernel which can - and presumably will - make use of this functionality in the future; this work was originally motivated by problems encountered in the implementation of zero-copy I/O for Xen clients. Ian suspects that subsystems like iSCSI could also benefit from this mechanism. 

The patch set seems to have been relatively well received. There will be another posting at some point reorganizing some of the work, but there does not appear to be a need for significant changes at this point. So this feature seems likely to appear in the 3.2 kernel.
