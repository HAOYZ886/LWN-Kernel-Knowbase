---
title: Integrating the ION memory allocator
url: https://lwn.net/Articles/565469/
date: "September 4, 2013"
category: "Android; ION; Memory management-Large allocations"
author: "September 4, 2013 This article was contributed by John Stultz"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

September 4, 2013

This article was contributed by John Stultz

As part of the Android + Graphics micro-conference at the [2013 Linux Plumbers Conference](<http://www.linuxplumbersconf.org/2013/>), we'll be discussing the [ION memory allocator](<https://lwn.net/Articles/480055/>) and how its functionality might be upstreamed to the mainline kernel. Since time will be limited, I wanted to create some background documentation to try to provide context to the issues we will discuss and try to resolve at the micro-conference. 

#### ION overview

The main goal of Android's ION subsystem is to allow for allocating and sharing of buffers between hardware devices and user space in order to enable zero-copy memory sharing between devices. This sounds simple enough, but in practice it's a difficult problem. On system-on-chip (SoC) hardware, there are usually many different devices that have direct memory access (DMA). These devices, however, may have different capabilities and can view and access memory with different constraints. For example, some devices may handle scatter-gather lists, while others may be able to only access physically contiguous pages in memory. Some devices may have access to all of memory, while others may only access a smaller portion of memory. Finally, some devices might sit behind an I/O memory management unit (IOMMU), which may require configuration to give the device access to specific pages in memory. 

If you have a buffer that you want to share with a device, and the buffer isn't allocated in memory that the device can access, you have to use bounce buffers to copy the contents of that memory over to a location where the other devices can access it. This can be expensive and greatly hurt performance. So the ability to allocate a buffer in a location accessible by all the devices using the buffer is important. 

Thus ION provides an interface that allows for centralized allocation of different "types" of memory (or "heaps"). In current kernels without ION, if you're trying to share memory between a DRM graphics device and a video4linux (V4L) camera, you need to be sure to allocate the memory using the subsystem that manages the most-constrained device. Thus, if the camera is the most constrained device, you need to do your allocations via the V4L kernel interfaces, while if the graphics is the most constrained device, you have to do the allocations via the Graphics Execution Manager (GEM) interfaces. ION instead provides one single centralized interface that allows applications to allocate memory that satisfies the required constraints. 

One thing that ION doesn't provide, though, is a method for determining what type of memory satisfies the constraints of the relevant hardware. This is instead a problem left to the device-specific user-space implementations doing the allocation ("Gralloc," in the case of Android). This hard-coded constraint solving isn't ideal, but there are not any better mainline solutions for allocating buffers with GEM and V4L. User space just has to know what is the most-constrained device. On mostly static hardware devices, like phones and tablets, this information is known ahead of time, but this limitation makes ION less suitable for upstream adoption in its current form. 

To share these buffers, ION exports a file descriptor which is linked to a specific buffer. These file descriptors can be then passed between applications and to ION-enabled drivers. Initially these were ION-specific file descriptors, but ION has since been reworked to utilize [dma-buf](<https://lwn.net/Articles/474819/>) structures for sharing. One caveat is that while ION can export dma-bufs it won't import dma-bufs exported from other drivers. 

#### ION cache management

Another major role that ION plays as a central buffer allocator and manager is handling cache maintenance for DMA. Since many devices maintain their own memory caches, it's important that, when serializing device and CPU access to shared memory, those devices and CPUs flush their private caches before letting other devices access the buffers. Providing a full background on caching would be out of the scope of this article, so I'll instead point folks to [this LWN article](<https://lwn.net/Articles/282250/>) if they are interested in learning more. 

ION allows for buffer users to set a flag describing the needed cache behavior on allocations. This allows those users to specify if mappings to the buffer should be cached with ION doing the cache maintenance, if the buffers will be uncached but use write-combining (see [this article](<https://lwn.net/Articles/440221/>) for details), or if the buffers will be uncached and managed explicitly via ION's synchronization `ioctl()`. 

In the case where the buffers are cached and ION performs cache maintenance, ION further tries to allow for optimizations by delaying the creation of any mappings at `mmap()` time. Instead, it provides a fault handler so pages are mapped in only when they are accessed. This method allows ION to keep track of the changed pages and only flush pages that were actually touched. 

Also, when ION allocates memory for uncached buffers, it is managing physical pages which aren't mapped into kernel space yet. Since these buffers may be used by DMA before they are mapped into kernel space, it is not correct to flush them at mapping time; that could result in data corruption. So these buffers have to be pre-flushed for DMA when they are allocated. So another performance optimization ION provides is that it pre-flushes pools of pages for DMA. On some systems, flushing memory for DMA on frequent small buffer allocations is a major performance penalty. Thus ION uses a page pool, which allows a large pool of uncached pages to be pre-allocated and flushed all at once, then when smaller allocations are made they just pick pages from the pool. 

Unfortunately both of these optimizations are somewhat problematic from an upstream perspective. 

Delayed mapping creation is problematic because the DMA API uses either scatter-gather lists or larger contiguous DMA areas; there isn't a generic interface to flush a single page. Because of this, when ION tries to flush only the pages that have been touched, it ends up using the ARM-specific `__dma_page_cpu_to_dev()` function, as it was too costly to iterate across the scatter-gather lists to find the faulted page. The use of this interface makes ION only buildable on 32-bit ARM systems. 

The pre-flushed page pools are also problematic: since these pools of memory are allocated ahead of time, it's not necessarily clear which device is going to be using them. Normally, when flushing pages for DMA, one must specify the device which will access the memory next, so in the case of a device behind an IOMMU, that IOMMU can be set up so the device can access those pages. ION gets away with this again by using the 32-bit ARM-specific `__dma_page_cpu_to_dev()` interface, which does not take a device argument. Thus this further limits ION's ability to function in more generic environments where IOMMUs are more common. 

For Android's uses, this limitation isn't problematic. 32-bit ARM is its main target, and, on Intel systems there is coherent memory and fewer device-specific constraints, so ION isn't needed there. Further, for Android's use cases, IOMMUs can be statically configured to specific heaps (boot-time reserved carve-out memory, for example) so it's not necessary to dynamically reconfigure the IOMMUs. But these limitations are problematic for getting ION upstream. The problem is that without these optimizations the performance penalty will be too high, so Android is unlikely to make use of more upstream-friendly approaches that leave out these optimizations. 

#### Other ION details

Since ION is a centralized allocator, it has to be somewhat flexible in order to handle all the various types of hardware. So ION allows implementations to define their own heaps beyond the common heaps provided by default. Also, since many devices can have quirky allocation rules, such as allocating on specific DIMM banks, ION allows some of the allocation flags to be defined by the heap implementation. 

It also provides an `ION_IOC_CUSTOM` `ioctl()` multiplexer which allows ION implementations to implement their own buffer operations, such as finer-grained cache management or special allocators. However, the downside to this is that it makes the ION interface actually quite hardware-specific — in some cases, specific devices require fairly large changes to the ION core. As a result, user-space applications that use the ION interface must be customized to use the specific ION implementation for the hardware they are running on. Again, this isn't really a problem for embedded devices where kernels and user space are delivered together, so strict ABI consistency isn't required, but is an issue for merging upstream. 

This hardware- and implementation-specific nature of ION also brings into question the viability of the centralized allocator approach ION uses. In order to enable the various features of all the different hardware, it basically has hardware-specific interfaces, forcing the writing of hardware-specific user-space applications. This removes some of the conceptual benefit of having a centralized allocator rather than using device specific allocators. However, the Android developers have reasoned that, by having a ION be a centralized memory manager, they can reduce the amount of complex code each device driver has to implement and allows for optimizations to be made once in the core, rather than over and over to various drivers of differing quality. 

To summarize the issues around ION: 

  * It does not provide a method to discover device constraints. 

  * The interface exposes hardware-specific heap IDs to user space. 

  * The centralized interface isn't sufficiently generic for all devices, so it exposes an `ioctl()` multiplexer for device-specific options. 

  * ION only imports dma-bufs from itself. 

  * It doesn't properly use the DMA API, failing to specify a device when flushing caches for DMA. 

  * ION only builds on 32-bit ARM systems. 

#### ION compared to current upstream solutions

In some ways GEM is a similar memory allocation and sharing system. It provides an API for allocating graphics buffers that can be used by an application to communicate with graphics drivers. Additionally, GEM provides a way for an application to pass an allocated buffer to another process. To do this one uses the `DRM_IOCTL_GEM_FLINK` operation, which provides a GEM-specific reference that is conceptually similar to a file descriptor that can be passed to another process over a socket. One drawback with this is that these GEM-specific "flink" references are just a global 32-bit value, and thus can be guessed by applications which otherwise should not have access to them. Another problem with GEM-allocated buffers is that they are specific to the device they were allocated for. Thus, while GEM buffers could be shared between applications, there is no way to share GEM buffers between different devices. 

With the advent of hybrid graphics implementations (usually discrete NVIDIA GPUs combined with integrated Intel GPUs), the need for sharing buffers between devices arose and dma-bufs and PRIME (a GEM-specific mechanism for sharing buffers between devices) were created. 

For the most part, dma-bufs can be considered to be marshaling structures for buffers. The dma-buf system doesn't provide any method for allocation, but provides a generic structure that can be used to to share buffers between a number of different devices and applications. The dma-buf structures are shared to user space using a file descriptor, which avoids the potential security issues with GEM flink IDs. 

The DRM PRIME infrastructure allows drivers to share GEM buffers via dma-bufs, which allows for things like having the Nouveau driver be able to render directly into a buffer that the Intel driver will display to the screen. In this way GEM and PRIME together provide functionality similar to that of ION, allowing for the type of buffer sharing (both utilizing dma-bufs) that ION enables on SoCs on more conventional desktop machines. However PRIME does not handle any information about what kind of memory the device can access, it just allows for GEM drivers to utilize dma-buf sharing, assuming all the devices sharing the buffer can access it. 

The V4L subsystem, which is used for cameras and video recorders, also has integrated dma-buf functionality, allowing camera buffers to be shared with graphics cards and other devices. It provides its own allocation interfaces but, like GEM, these allocation interfaces only make sure the buffer being allocated works with the device that the driver manages, and are unaware of what constraints other drivers the buffer might be shared with have. 

So with the current upstream approach, in order to share buffers between devices, user space must know which devices will share the buffer and which device has the most restrictive constraints; it must then allocate the buffer using the API that most-constrained driver uses. Again, much like in the ION case, user space has no method available in order to determine which device is most-constrained. 

The upstream issues can thus be summarized this way: 

  * There is no existing solution for constraint-solving for sharing buffers between devices. 

  * There are different allocation APIs for different devices, so, once users determine the most constrained device, they have to then do allocation with the matching API for that device. 

  * The IOMMU and DMA API interfaces do not currently allow for the DMA optimizations used in ION. 

#### Possible solutions

Previously, when ION has been discussed in the community, there have been a few potential approaches proposed. Here are the ones I'm aware of. 

One idea would be to try to just merge a centralized ION-like allocator upstream, keeping a similar interface. To address the problematic constraint discoverability issue, devices would export a opaque heap cookie via sysfs and/or via an `ioctl()`, depending on the device's needs (devices could have different requirements depending on device-specific configuration). The meaning of the bits would not be defined to user space, but could be ANDed together by a user-space application and passed to the allocator, much as the heap mask is currently with ION. This provides a way for user space to do the constraint solving but avoids the problem of fixing heap types into the ABI; it also allows the kernel to define which bits mean which heap for a given machine, making the interface more flexible and extensible. This, however, is a more complicated interface for user space to use, and many do not like the idea of exposing the constraint information to user space, even in the form of an opaque cookie. 

Another possible solution is to allow dma-buf exporters to not allocate the backing buffers immediately. This would allow multiple drivers to attach to a dma-buf before the allocation occurs. Then, when the buffer is first used, the allocation is done; at that time, the allocator could scan the list of attached drivers and be able to determine the constraints of the attached devices and allocate memory accordingly. This would allow user space to not have to deal with any constraint solving. 

While this approach was planned for when dma-bufs were originally designed, much needed infrastructure is still missing and no drivers yet use this solution. The Android developers have raised the concern that this sort of delayed allocation could cause non-deterministic latency in application hot-paths, though, without an implementation, this has not yet been quantified. Another downside is that this delayed allocation isn't required of all dma-buf exporters, so it would only work with drivers that actually implement this feature. Since the possibility exists that not all of the drivers one would want to share a buffer with would support delayed allocation, applications would have to somehow detect the functionality and make sure to allocate memory to be shared using the dma-buf exporter that does support this delayed allocation functionality. This approach also requires the exporter driver allocators to each handle this constraint solving individually (though common helper functions may be something that could be provided). 

Another possible approach could be to prototype the dma-buf late-allocation constraint solving using a generic dma-buf exporter. This in some ways would be ION-like in that it would be a centralized exporter, but would not expose heap IDs to user space. Then the buffer would be attached to the various hardware drivers, and, on the first use, the exporter would determine the attached constraints and allocate the buffer. This would provide a testing ground for the delayed-allocation approach above while having some conceptual parallels to ION. The downside to this approach would be that the centralized interface would likely not be able to address the more intricate hardware-specific allocation flags that possibly could be needed. 

Finally, none of these proposals address the non-generic caching optimizations ION uses, so those issues will have to be discussed further. 

#### Conference Discussion

I suspect at the Linux Plumbers Android + Graphics mini-conference, we won't find a magic or easy solution on how to get Android's ION functionality upstream. But I hope that, by having key developers from both the Android team and the upstream kernel community able to discuss their needs and constraints and be able to listen to each other, we might be able to get a sense of which subproblems have to be addressed and what direction forward we might be able to take. To this end I've created a few questions for folks to think about and discuss so we can hopefully come up with answers for during the discussion: 

  * Current upstream dma-buf sharing uses, such as PRIME, seem focused on x86 use cases (such as two devices sharing buffers). Will these interfaces really scale to ARM-style uses cases (many devices sharing buffers) in a generic fashion? As non-centralized allocation requires exporters to manage more logic and understand device constraints, and there seems to be a potential that this approach will eventually be unmaintainable. 

  * ION's centralized allocation style is problematic in many cases, but also provides significant performance gains. Is this too major of an impasse or is there a way forward? 

  * What other potential solutions haven't yet been considered? 

  * If a centralized dma-buf allocation API is the way forward, what would be the best approach (ie: heap-cookies, vs post-attach allocation)? 

  * Is there any way to implement some of the caching optimizations ION uses in a way that is also more generically applicable? Possibly extending IOMMU/DMA-API? 

  * Given Android's needs, what next steps could be done to converge on a solution? How can we test to see if attach-time solving will be usable for Android developers? What would it miss that ION still provides? 

  * How do Android developers plan to deal with IOMMUs and non-32-bit ARM architecture issues? 

Some general reference links follow: 

  * [General graphics overview [PDF]](<http://people.freedesktop.org/~marcheu/linuxgraphicsdrivers.pdf>)
  * [Caching overview](<https://lwn.net/Articles/282250/>)
  * [Write combining](<http://en.wikipedia.org/wiki/Write-combining>)
  * ION overviews: [1](<https://lwn.net/Articles/480055/>), [2](<http://groleo.wordpress.com/2012/07/24/ion-buffer-sharing-mechanism/>), [3](<https://git.linaro.org/gitweb?p=people/jstultz/android.git;a=history;f=drivers/gpu/ion;hb=refs/heads/upstream/android-3.4>)
  * [DRM overview](<http://free-electrons.com/kerneldoc/latest/DocBook/drm/index.html>)
  * GEM: [1](<http://blog.ffwll.ch/2011/05/gem-overview.html>), [2](<http://blog.ffwll.ch/2012/10/i915gem-crashcourse.html>)
  * [DMA-buf](<https://lwn.net/Articles/474819/>)
  * [PRIME](<http://airlied.livejournal.com/71734.html>) 

#### Credits

Thanks to Laurent Pinchart, Jesse Barker, Benjamin Gaignard and Dave Hansen for reviewing and providing feedback on early drafts of this document, and many thanks to Jon Corbet for his careful editing.
