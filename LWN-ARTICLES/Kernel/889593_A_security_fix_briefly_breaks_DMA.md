---
title: A security fix briefly breaks DMA
url: https://lwn.net/Articles/889593/
date: "April 1, 2022"
category: Direct memory access; swiotlb
author: "By Jonathan Corbet April 1, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 1, 2022

In theory, direct memory access (DMA) operations are simple to understand; a device transfers data directly to or from a memory buffer managed by the CPU. Almost all contemporary devices perform DMA, since it would not be possible to obtain the needed performance without it. Like so many things, DMA turns out to be a bit more complicated in practice. That complexity led to an erroneous patch, intended to improve security, breaking DMA for some devices in 5.17 and some stable kernels. 

The simple model of a DMA transfer looks something like this: 

> ![\[DMA model\]](https://static.lwn.net/images/2022/bounce-buffer1.png)

There is a buffer in memory that can be accessed — for either reading or writing — by both the CPU and the peripheral device, and either side can access the buffer at will. Some systems actually work this way but, in most others, various complications come into play. For example, there is almost certainly a set of caches between the CPU and the memory buffer. If the CPU writes data to the buffer with the intent of transferring it to the device, that data may be resident in the cache for some time before being flushed to main memory. If the device is instructed to read the data from the buffer, it may not see the data in cache, resulting in incorrect (corrupted) data being written to the device. 

Similarly, on many architectures, the CPU cache may be entirely unaware of data written to the DMA buffer by the device. If the CPU tries to read that data, it could instead get stale data from the cache, once again resulting in corrupted data. Long experience in the kernel-development community has shown that users have a certain tendency to become irate when that happens. 

#### Ownership, direction, and bounce buffers

The kernel's DMA-support layer has grown a set of mechanisms designed to prevent data corruption and the unhappiness that follows from it. In particular, the DMA API can avoid the pitfalls that come with systems that are not [cache-coherent](<https://en.wikipedia.org/wiki/Cache_coherence>) — where copies of data in different locations may not be synchronized with each other. When dealing with non-cache-coherent (or "streaming") mappings, driver code must keep track of two important attributes: the ownership of the buffer and the direction that the data is moving. Ownership describes which side (the CPU or the device) is able to access the buffer at any given time; while data direction describes what the owner will be doing with the buffer. There are two functions (among others) that manage these attributes for a given DMA buffer: 

```
void dma_sync_single_for_device(struct device *dev, dma_addr_t addr,
    				    size_t size, enum dma_data_direction dir);
        void dma_sync_single_for_cpu(struct device *dev, dma_addr_t addr, size_t size,
    				 enum dma_data_direction dir);
```

When a DMA buffer is first mapped, the ownership belongs to the CPU. A call to `dma_sync_single_for_device()` sets the ownership to the device, while `dma_sync_single_for_cpu()` brings the ownership back to the CPU. The `dir` argument to both functions describes the direction that data is moving; it is one of `DMA_NONE`, `DMA_TO_DEVICE`, `DMA_FROM_DEVICE`, or `DMA_BIDIRECTIONAL`. Note that terms like "read" and "write" are not used here, since in almost any DMA transaction one side is reading from the buffer while the other is writing to it. 

The combination of the ownership and the direction tells the DMA layer what needs to be done at any given time. If the CPU is passing ownership to the device, and the direction is `DMA_TO_DEVICE`, then the DMA layer must take care to flush any data from the CPU caches to the buffer so that the device sees the correct data. When taking ownership back from the device, with a direction of `DMA_FROM_DEVICE`, the DMA layer should instead invalidate any cached data, since it may well be incorrect. As long as the driver is careful to set the ownership and direction of the buffer properly at all times, there should be no problems with data corruption. 

There is one other complication to cover in order to understand what went wrong. There are times when the device can perform DMA, but it cannot directly access the buffer that the CPU is using. The buffer may be in a region of memory that the device cannot reach, it may be split into too many discontiguous pieces, or there may be an I/O memory-management unit (IOMMU) complicating the picture. When this happens, a separate DMA-layer module (called the "software I/O translation lookaside buffer" or SWIOTLB) allocates a "bounce buffer" that is accessible to the device: 

> ![\[DMA model\]](https://static.lwn.net/images/2022/bounce-buffer2.png)

In this situation, data must often be copied between the two buffers. If the CPU is sending data to the device, for example, data must be copied ("bounced") from the CPU's buffer to the bounce buffer, and the address of the bounce buffer handed to the device. Data will be copied in the opposite direction when data is being received from the device. The good news is that the DMA API is able to handle this case transparently as long as the ownership calls described above are properly used. Device drivers need not know whether a bounce buffer is in use or not. 

#### A problem with bounce buffers

There is a potential security problem with bounce buffers, though. If data is being transferred from the device to the CPU, the DMA code will, when ownership of the DMA buffer returns to the CPU, copy the contents of the bounce buffer back to the CPU's buffer. If, however, the device did not write a full buffer's worth of data, then some of the data copied out of the bounce buffer will have originated from somewhere else. It may come from a previous I/O operation, or from an entirely unrelated kernel subsystem. The device driver is then likely to copy this data back to user space; indeed, the buffer might be directly mapped into user space to begin with. The end result is that data is leaked from the kernel to user space. 

It makes sense to zero the bounce buffer before setting it up, but DMA buffers are often used many times, and it is not normal to zero them between operations. So even a bounce buffer that was zeroed at the beginning may accumulate unrelated data and expose it to user space. A real-world example of this problem was deemed [CVE-2018-1000204](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-1000204>) and [fixed](<https://github.com/torvalds/linux/commit/a45b599ad808c3c982fdcdc12b0b8611c2f92824>) in 2018, but that fix is only partial, since it only zeroes the buffer at allocation time. 

Halil Pasic wrote a more complete fix that was [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ddbd89deb7d3>) in February; a [followup fix](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=aa6f8dcbab47>) was added in early March. It works by changing the behavior of `dma_sync_single_for_device()`. If the data direction is `DMA_TO_DEVICE`, the contents of the CPU's buffer must be copied to the bounce buffer, and the DMA API has always done that. Pasic's patch caused that same copy to happen for all operations, even when the direction is `DMA_FROM_DEVICE`. 

Normally, it would not make sense to copy data into the bounce buffer for `DMA_FROM_DEVICE`; the device is just going to overwrite it anyway. But this copy ensures that, if the device only writes part of the bounce buffer, the remainder of the CPU's buffer will not be overwritten by random data when the bounce buffer is copied back; instead, it will get back a copy of the data that was already there. In a sense, it makes the end result reflect what the device has actually done, in that only data written by the device will change in the CPU's buffer. 

The latter fix was merged for the 5.17-rc8 development kernel — less than two weeks before the final 5.17 release — and quickly found its way into the 5.16.15 and 5.15.29 stable updates. 

#### Regression

On March 23, Oleksandr Natalenko [reported](<https://lwn.net/ml/linux-kernel/1812355.tdWV9SEqCh@natalenko.name/>) that this change broke the ath9k wireless driver. There followed an extended discussion where it took a while to figure out what was really going on. Robin Murphy was initially [incredulous](<https://lwn.net/ml/linux-kernel/27b5a287-7a33-9a8b-ad6d-04746735fb0c@arm.com/>): 

> I'm extremely puzzled how any driver could somehow be dependent on non-device-written data getting replaced with random crap, given that it wouldn't happen with a real IOMMU, or if SWIOTLB just didn't need to bounce, and the data would hardly be deterministic either. 

The source of the problem was eventually narrowed down to some code in [`ath_edma_get_buffers()`](<https://elixir.bootlin.com/linux/v5.17.1/source/drivers/net/wireless/ath/ath9k/recv.c#L627>). This code executes a sequence that looks like this: 

```
dma_sync_single_for_cpu(..., DMA_FROM_DEVICE);
        if (! a_packet_is_ready()) {
            dma_sync_single_for_device(..., DMA_FROM_DEVICE);
    	return false;
        }
```

The code is taking ownership of the buffer, checking to see if the device has put a packet there, then returning ownership to the device if there is no packet available. The key point here is that, while this is happening, the device is still writing to the buffer; that packet could be arriving while the driver is checking for it. This procedure worked just fine until the change went in; with the new behavior, the `dma_sync_single_for_device()` call copies the CPU buffer into the bounce buffer, potentially overwriting data that was just placed there by the device. This happens often enough, it seems, to reliably break the device. 

Developers like Christoph Hellwig initially [saw the ath9k behavior as a bug](<https://lwn.net/ml/linux-kernel/20220325163204.GB16426@lst.de/>), and felt that the problem should be fixed there. Murphy [described](<https://lwn.net/ml/linux-kernel/e077b229-c92b-c9a6-3581-61329c4b4a4b@arm.com/>) this behavior as ""a violation of the DMA API"". The problem was that the device was allowed to keep writing to the DMA buffer even though the ownership had shifted to the CPU. It seemed, for a moment, that the ath9k driver could be fixed and the bounce-buffering change could remain. 

Linus Torvalds [disagreed strongly](<https://lwn.net/ml/linux-kernel/CAHk-=wippum+MksdY7ixMfa3i1sZ+nxYPWLLpVMNyXCgmiHbBQ@mail.gmail.com/>), though, for a few different reasons. He noted that ath9k might not be the only driver that shows this kind of problem; it just showed up first there because those adapters are widely used. If other drivers do similar things, users and developers could end up chasing bugs for a long time. Some of those bugs might well make it into production kernel releases before being noticed. 

More to the point, though, he stated that the ath9k driver's behavior was correct, and the bounce-buffer change was not. Specifically, he pointed out that the `dma_sync_single_to_device()` call specified `DMA_FROM_DEVICE`. In that situation, he claimed, the bounce-buffer implementation should do nothing at all; the data is coming from the device, so the CPU has no business copying data into the bounce buffer. So the only right thing to do is to revert the patch. 

In the end, reverting the commit is exactly what happened, though it was later [partly reinstated](<https://git.kernel.org/linus/901c7280ca0d>) to preserve the parts of the patch that were not problematic. So the ath9k driver works again, and potential bugs in an unknown number of other drivers have been avoided. The problem of leaking random data out of the bounce buffer remains unsolved; a different approach will need to be found to resolve that one.
