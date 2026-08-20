---
title: The return of preadv2()/pwritev2()
url: https://lwn.net/Articles/670231/
date: "January 6, 2016"
category: System calls
author: "By Jonathan Corbet January 6, 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 6, 2016

Back in 2014, Milosz Tanski [tried to add a `flags` argument](<https://lwn.net/Articles/612483/>) to the `preadv()` and `pwritev()` system calls. At the time, your editor suggested that the patch set might be ready for the 3.19 development cycle. It might have been, but things stalled and the work was never actually merged. Now this idea is back, but with a different end goal in mind. 

The existing `preadv()` and `pwritev()` system calls (along with `readv()` and `writev()`, which are simpler versions of them) lack any means to pass operation-specific options into the kernel. So there is no way to change how a specific call works. Milosz's specific need was to be able to turn on non-blocking behavior for some, but not all, operations; this is a feature that appears to have a number of use cases. The idea was [reasonably well received](<https://lwn.net/Articles/636967/>) at the (March) 2015 Linux Filesystem, Storage and Memory Management Summit. Even so, work on these patches seemed to come to a halt, with [the last posted version](<https://lwn.net/Articles/636955/>) showing up in March. 

Now, however, [`preadv2()` and `pwritev2()` are back](<https://lwn.net/Articles/670018/>), though with a different use case in mind. This patch set, posted by Christoph Hellwig, still introduces the new system calls, which still look like: 

```
int preadv2(unsigned long fd, struct iovec *vec, unsigned long vlen,
                    unsigned long pos_l, unsigned long pos_h, int flags);
        int pwritev2(unsigned long fd, struct iovec *vec, unsigned long vlen,
                     unsigned long pos_l, unsigned long pos_h, int flags);
```

(Note that the system calls, as presented by the C library, would almost certainly be a little different, with `pos_l` and `pos_h` being combined into a single, 64-bit position value). 

Unlike Milosz's patch set, though, Christoph's does not provide for non-blocking operations. Instead, it provides a different flag (`RWF_HIPRI`) allowing an application to indicate a high-priority operation. The block layer can then use that flag to decide whether it should use [the new block-layer polling mechanism](<https://lwn.net/Articles/663879/>) with that request or not. Polling can, for fast devices (non-volatile memory, for example), reduce latencies significantly. But polling has its costs as well; it probably is best used only when an application is concerned about cutting latency to the bare minimum. Without a flag like `RWF_HIPRI`, the kernel can't really know if the application cares about latency or not. 

Christoph hasn't forgotten about the non-blocking read use case; he also mentions other possibilities like per-operation synchronous behavior or use of the DIF/DIX data integrity mechanism. But his first priority at this point is to get the new system calls in place, along with the polling feature. Once that has been done, there are plenty of other flags that can be added.
