---
title: "A reworked TCP zero-copy receive API"
url: https://lwn.net/Articles/754681/
date: "May 18, 2018"
category: "Networking-Performance"
author: "By Jonathan Corbet May 18, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 18, 2018

In April, LWN [looked at the new API](<https://lwn.net/Articles/752188/>) for zero-copy reception of TCP data that had been merged into the net-next tree for the 4.18 development cycle. After that article was written, a couple of issues came to the fore that required some changes to the API for this feature. Those changes have been made and merged; read on for the details. 

This API is intended to make it possible to read data from a TCP connection without the need to copy the data between the kernel and user space. The original version was based entirely on the `mmap()` system call; once a socket had been marked for zero-copy reception, an `mmap()` call would create a mapping containing the available data — in some circumstances, anyway. The application would use `munmap()` to release that data once processing was complete; see the article linked above for the details. 

Two problems with this interface came to light after the feature had been merged. One was that this use of `mmap()` was somewhat strange; an `mmap()` call does not normally have side effects like consuming data from a socket. The author of this patch (Eric Dumazet) [was comfortable](<https://lwn.net/ml/netdev/633a84fd-ef6a-91a1-e45a-86d44ab2b9ac@gmail.com/>) with that aspect of the interface, but he had a harder time dealing with the locking problems that came with it. Calling network-layer operations from within `mmap()` inverts the normal locking order around `mmap_sem`; there was no easy way to fix that without separating the networking operations from the `mmap()` code. 

So, in the version that (barring more surprises) will be merged for 4.18, the call to `mmap()` just sets up a range of address space into which data from the network can appear via zero-copy magic. Actually getting some data into that range requires a `getsockopt()` call with the `TCP_ZEROCOPY_RECEIVE` operation. This structure is passed into that call: 

```
struct tcp_zerocopy_receive {
    	__u64 address;
    	__u32 length;
    	__u32 recv_skip_hint;
        };
```

On entry to `getsockopt()`, the `address` field contains the address of the special mapping created with `mmap()`, and `length` is the number of bytes of data to be put into that mapping. As before, `address` must be page-aligned (which will happen naturally since it must also be the address returned from the `mmap()` call), and `length` must be a multiple of the page size. On return, `length` will be set to the number of bytes actually mapped into that range. The data will remain mapped until either the range is unmapped with `munmap()` or another `getsockopt()` call replaces the data. 

In the old interface, the `mmap()` call would fail if the available data did not fill full pages or if there is pending urgent data. The new `getsockopt()` call will fail in the same way in those circumstances, but with a difference: the `recv_skip_hint` field of the `tcp_zerocopy_receive` structure will be set to the amount of data the application must consume with `recv()` before returning to the zero-copy mode. That should make it easier for applications to recover when things don't go as planned. 

The new interface should also perform better, especially in multi-threaded applications, because it is no longer necessary to call `mmap()` for each new batch of data. The implementation can also avoid making some higher-order allocations that were necessary with the old API. 

The end result is an interface that is less surprising, easier to use, and perhaps even faster for some use cases. The whole episode is a clear demonstration of the benefits of wider review of new features, especially those that have user-space API components. In this case, a number of the ideas behind the new implementation [came from Andy Lutomirski](<https://lwn.net/ml/linux-kernel/9ed6083f-d731-945c-dbcd-f977c5600b03@kernel.org/>), who seemingly only became aware of the changes once they were discussed beyond the netdev mailing list. Having many eyes on the code really does make it better in the end.
