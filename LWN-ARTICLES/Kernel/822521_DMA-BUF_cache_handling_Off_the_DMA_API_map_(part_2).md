---
title: "DMA-BUF cache handling: Off the DMA API map (part 2)"
url: https://lwn.net/Articles/822521/
date: "June 11, 2020"
category: "Device drivers-Support APIs; Direct memory access"
author: "June 11, 2020 This article was contributed by John Stultz"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

June 11, 2020

This article was contributed by John Stultz

[Part 1 of this series](<https://lwn.net/Articles/822052/>), covered some background on ION, DMA-BUF heaps, the DMA API, and the concept of "ownership" when it comes to handling CPU-cache maintenance, finally ending on a conventional DMA API view of how DMA-BUF cache handling should be done. The article concluded with a discussion of why the traditional DMA APIs can perform poorly on contemporary systems. This article completes the series with an exploration of some of the approaches that DMA-BUF exporters can use to avoid unnecessary cache operations along with some rough proposals for how we might improve things. 

The first part left off with the DMA API's viewpoint that the ownership of a DMA-BUF is transferred to the device domain with a call to `dma_buf_map_attachment()` and transfers back to the CPU with `dma_buf_unmap_attachment()`; cache-maintenance operations are done on each call. This sequence provides correctness with regard to CPU-cache handling but, for buffer pipelines that involve many devices where the CPU doesn't actually touch the buffer, these cache operations on each map and unmap call add up and can cause dramatic performance problems. 

#### Who owns the buffer?

To avoid these extraneous cache operations, the DMA-BUF interface allows one aspect of the DMA-API rules to be turned upside down. The DMA API, remember, assumes that the CPU is the natural owner of all memory and that it is only during DMA transactions — where buffer ownership has been clearly transferred to the device — that careful rules are needed. The DMA-BUF interfaces, instead, require the CPU to call [`dma_buf_begin_cpu_access()`](<https://elixir.bootlin.com/linux/v5.7.1/source/drivers/dma-buf/dma-buf.c#L1064>) before accessing a DMA-BUF. After it is done, the CPU calls [`dma_buf_end_cpu_access()`](<https://elixir.bootlin.com/linux/v5.7.1/source/drivers/dma-buf/dma-buf.c#L1100>). If the buffer is accessed from user space, these transitions are effected using the `DMA_BUF_IOCTL_SYNC` `ioctl()` command instead. 

Specifically: 

> `dma_buf_begin_cpu_access()`
>      Allows the exporter to ensure that the memory is actually available for CPU access; the exporter might need to allocate or swap-in and pin the backing storage. The exporter also needs to ensure that CPU access is coherent for the requested access direction. 
> 
> `dma_buf_end_cpu_access()`
>      Called when the importer is done accessing the CPU. The exporter can use this to flush caches and unpin any resources pinned in `dma_buf_begin_cpu_access()`. 

This approach allows one to look at the DMA-BUF memory as not being owned by the CPU by default; it can be considered to be in the device domain by default instead. CPU-cache handling can thus be done primarily in those calls to ensure the CPU gets a consistent view of the DMA-BUF and may allow for the avoidance of costly cache operations in cases where we pass, map, and access the DMA-BUF only between multiple devices. 

However, this inconsistency with the DMA API causes confusion, and not all DMA-BUF exporters take the same approach. Some exporters try to abide by the DMA API, flushing and invalidating the CPU cache on every map and unmap operation, some may leave those cache operations to the exporter's begin and end hooks, and others do both. 

While DMA-BUFs were designed to share buffers between user space and multiple devices, the first DMA-BUF exporters were tied to specific drivers, often superseding custom, driver-specific buffer-allocation code. Consider, for example, a GPU driver that allocates a buffer, performs some rendering into it, and provides a handle back to user space. The user-space process could then pass that buffer, and others along with it, back to the GPU to compose the web-browser window with the other windows that were open in a desktop environment. Switching to a DMA-BUF provided a more generic handle type, so it made sense to use even if the buffers weren't really being shared between multiple devices. 

But, knowing that the buffer was shared between just the CPU and the device, the DMA-BUF exporter could optimize some of the cache operations. For instance, some DMA-BUF exporters cache the scatter/gather table resulting from the first DMA mapping operation and, as long as the `dma_buf_map_attachment()` calls are done in the same direction, reuse that table. In this way, they can avoid expensive cache operations on each `dma_buf_map_attachment()` and `dma_buf_unmap_attachment()` call, finally releasing the mapping in `dma_buf_detach()`. These optimizations work because the exporters are tied to the device, so the buffers aren't really being shared, or the devices the buffers are shared with are cache coherent, so the cache maintenance is unnecessary. 

This scheme is efficient, but it has resulted in the dozen or so DMA-BUF exporters upstream having different cache-handling and usage semantics. So when we start to look at how to implement generic DMA-BUF exporters to support multiple-device pipelines in some sort of performant way, the rules of the road are not clearly established. 

#### Handling ownership with multiple mappings

While the DMA API provides good guidance for specifying ownership using the map and unmap calls, getting good performance on mobile devices often requires that multiple devices and the CPU all have active mappings to a buffer at the same time. That makes the concept of buffer ownership quite a bit more subtle. For instance, it's common in graphics to have a buffer mapped both by the GPU and a display at the same time. To do this, the system has to be able to share a frame buffer between multiple devices before the frame is finished drawing. This allows the GPU to write to the buffer directly and then signal the display driver, which then can immediately show the buffer. 

For this specific use case, [DMA-BUF fences](<https://www.kernel.org/doc/html/v5.6/driver-api/dma-buf.html#dma-fences>) built on the [explicit fence infrastructure](<https://lwn.net/Articles/702339/>) were added, providing a mechanism for a driver (or user space) to wait on a fence that is specific to a buffer. Another driver will eventually signal that fence, initiating the transfer of ownership. However, supporting these parallel mappings requires careful cache handling. Usually this is left to the drivers to do explicitly using the DMA-API synchronization calls. When one is working on a single integrated device with a custom vendor kernel, it is possible to know which driver passes buffers to which and, thus, to be able to add the optimal and correct cache handling. But outside of that controlled environment, it's more complicated. 

So we are already seeing two different styles of ownership tracking being handled here. **Implicit handling** means that the ownership of a DMA-BUF is transferred when the buffer is mapped to (or unmapped from) a device. **Explicit handling** , instead, is where buffers are already mapped on two (or more devices) devices and ownership is effectively transferred by DMA-BUF fences. 

DMA-BUF exporters are normally responsible for handling the cache operations for buffers as the ownership of the buffer is passed around. They can do so properly in the implicit context of `dma_buf_map_attachment()` and `dma_buf_unmap_attachment()` calls or, alternatively, in the `dma_buf_begin_cpu_access()` and `dma_buf_end_cpu_access()` calls. However, in the explicit case, the DMA-BUF exporter has no hooks for DMA-BUF fence signals, so the exporter cannot do any cache management in response to the transfer of ownership. This creates a difficult situation, where the responsibility for the buffer cache management is split between the DMA-BUF exporter and the drivers using the buffer. To do it correctly, each driver must each understand its place in a buffer pipeline to know the coherency of the device that comes after it. 

More problematically, even if the DMA-BUF exporter did have a hook for the DMA fence signals, it has no way of knowing which of the multiple styles of ownership tracking is being used. Do we perform cache operations on map and unmap calls, assuming explicit usage with default CPU ownership? Or on `dma_buf_begin_cpu_access()` and `dma_buf_end_cpu_access()`, assuming implicit usage with default device ownership? Or do we avoid that extra overhead and assume drivers will do explicit fence signaling to transfer ownership? These choices may leave us with an implementation that is either too slow to use, or potentially incompatible with some drivers. This is quite contrary to the goal of DMA-BUFs functioning as a generic interchange mechanism. 

So for someone trying to write a DMA-BUF exporter, this all starts to feel like a level of 10 (""Read the documentation and you'll get it wrong"") or 11 (""Follow common convention and you'll get it wrong"") on Rusty Russell's [classic API scale](<https://ozlabs.org/~rusty/ols-2003-keynote/img56.html>), especially if you care about performance. This poses a large problem for the goal of having generic DMA-BUF heaps that can be shared between vendors. 

#### Potential solutions

I feel like this situation could be improved, and have a few ideas we might consider. Since the DMA-BUF interface already strays from the DMA API, I think we should establish some explicit conventions for how DMA-BUFs should be used. Better documentation could help, so that both DMA-BUF exporter authors and DMA-BUF users have a solid sense of the model to follow. In particular, we should try to: 

  * Create a formal sense of ownership for DMA-BUF objects outside of the implicit map/unmap method that the DMA API specifies. 
  * Work to provide some mechanism to formally track that ownership. These hooks could be added to the `dma_buf_ops` structure so the exporter can be informed of these ownership changes. 
  * Deprecate implicit handling and move drivers to use this mechanism to mark the explicit ownership transfers. 
  * Add some state tracking for DMA-BUFs so that we know their cache state and we can correctly perform cache operations only on transitions of ownership that make them necessary. 

It may be that most of the above can be achieved with documentation and consolidating the current DMA-BUF exporter implementation semantics. The `dma_buf_begin_cpu_access()` and `dma_buf_end_cpu_access()` calls are sufficient to handle device-to-CPU and CPU-to-device transitions. But their proper use needs to be explicitly defined as such and consistently implemented by DMA-BUF exporters, formalizing the concept that buffers are device-owned by default. This would allow for safely implementing pre-flushed buffers and skipping unnecessary synchronization operations. 

However, a drawback with this approach alone is that, for multiple uses by the CPU in a series, there would be unnecessary flushes on each call. Additionally, there is some concern that, with a mix of CPU-coherent and non-coherent devices, we may need to do CPU-cache handling when transferring ownership between such devices. Both of these situations might make it useful to have some device-usage bracketing calls along with some state tracking so that ownership transfer (and not just usage) could be determined. 

This concept of ownership will also need to consider future work to support partial cache flushes, allowing both the CPU and device to be carefully working on the same buffer at the same time. Thus, ownership (and the respective cache operations) would be managed on the granularity of a single cache line, rather than the entire buffer, possibly looking more like advisory range locks on a file. 

The DMA-BUF heaps interface (along with ION that came before it) concedes that, in some cases, user space knows more about how a buffer will be used than the kernel does. Thus, it can be optimal to let user space choose the allocation type for a given pipeline. The DMA-BUF design, which allows the rules and policy for a buffer to be left to the exporter implementations, provides a lot of useful flexibility, which I don't want to eliminate. However, I do think that, as vendors start their ION migration efforts, having clear, established conventions that don't have large pits to fall into will be important to avoid a collection of unnecessarily incompatible heaps and users. Hopefully this stirs some further discussion. 

#### Thanks

Thanks so much to Rob Clark, Robert Foss, Sumit Semwal, Azam Sadiq Pasha Kapatrala Syed, Daniel Vetter, and Linus Walleij for their early reviews and feedback on this series.
