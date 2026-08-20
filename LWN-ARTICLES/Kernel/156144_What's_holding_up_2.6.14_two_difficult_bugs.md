---
title: "What's holding up 2.6.14: two difficult bugs"
url: https://lwn.net/Articles/156144/
date: "October 18, 2005"
category: "Read-copy-update; swiotlb"
author: 
---

Linus was set on releasing the 2.6.14 kernel on October 17, when a little issue came up. Serge Belyshev [discovered](<https://lwn.net/Articles/156145/>) that it is easy to cause the system to stop opening files for user-space applications. He posted a program which, in essence, does the following: 

```
while (1) {
            int fd = open("/dev/null", O_RDONLY);
    	close(fd);
        }
```

After some 50,000 iterations, the open fails with a "too many open files in system" message. This behavior can be problematic in more realistic situations; it evidently can cause highly-parallel kernel builds to fail, and it also exposes the system to local denial of service attacks. So it is worth tracking down. 

The kernel places a limit on the number of files which are allowed to be open simultaneously. That limit is not normally expected to include files which have been closed, however. The problem, as it turns out, is a virtual filesystem scalability patch which was merged in September. That patch eliminates some locking around `file` structures in the kernel, and, to that end, defers certain tasks (such as file cleanup) to the [read-copy-update](<http://lwn.net/Articles/37889/>) mechanism. For this particular case, `file` structures corresponding to closed files are building up in the RCU callback list, and RCU is not getting around to freeing them in time. 

Initially, it was thought that the culprit was another patch which put a limit on the processing of the RCU callback lists. Those lists can get quite long, and lengthy callback processing was causing latency problems elsewhere in the kernel. So a "batch size" of ten was imposed; after ten callbacks have been processed, the RCU subsystem defers the rest until later. It seemed that this limit was causing the freeing of `file` structures to languish. Raising the batch limit to 10,000 seemed to improve the situation, so Linus merged a patch to that effect. 

But, in fact, the higher batch limit did not solve the problem for real. RCU callbacks cannot be called immediately after being queued. They must, instead, wait until every processor on the system has scheduled at least once. This "quiescence" requirement is the kernel's way of ensuring that no references to the freed structure remain; it's a key part of how RCU works. If a process chews through `file` structures quickly enough, they will accumulate while the kernel waits for the grace period to run out, and no changes to the batch limits will help. The only way to be able to process those callbacks - and free the associated structures - is to force every processor to schedule. 

A couple of patches have been posted in an attempt to deal with this problem. One of them simply changes the way `file` structures are accounted for - they are removed from the count of open files when the RCU callback is queued, rather than when it is executed. This patch stops programs from running into the maximum open file limit, but does nothing to stop the growth of the RCU callback queues. So the patch which got merged, instead, is [this one from Eric Dumazet](<https://lwn.net/Articles/156160/>), which keeps track of the length of the callback list. Should the list get to be too long (where "too long" is wired at 10,000 entries), a reschedule is forced so that the callbacks can be processed. This patch appears to have dealt with the problem well enough to allow 2.6.14 to come out, though more refinement may be required afterward. 

Unfortunately for those who are waiting for 2.6.14, another problem turned up. Some 64-bit architectures which lack I/O memory management units must be very careful in setting up DMA areas. A number of devices can only reliably deal with 32-bit DMA addresses, so DMA areas must be allocated in the lower part of memory. To that end, the x86-64 and ia64 architectures use a mechanism called the "software I/O translation buffer", or swiotlb. It is simply a large chunk of low memory, allocated at boot time, which is used as a bounce buffer for DMA operations involving 64-bit-challenged devices. 

It [was noted](<https://lwn.net/Articles/156162/>) that the 2.6.14-rc4 kernel can allocate the swiotlb area in high memory, which defeats the entire purpose. This revelation led to a long discussion of how swiotlb memory should be allocated. It turns out that there is no easy way of finding the low memory on the system. Once upon a time, that memory would belong to CPU 0, but on [some contemporary NUMA systems](<https://lwn.net/Articles/156163/>), the low memory might be elsewhere. So the real solution appears to iterate through all CPUs on the system, try to allocate from each of them, and test to see if the resulting memory is within the DMAable range. If not, the memory is freed and the next processor is tried. A couple of patches implementing this approach are circulating; none has been merged as of this writing.
