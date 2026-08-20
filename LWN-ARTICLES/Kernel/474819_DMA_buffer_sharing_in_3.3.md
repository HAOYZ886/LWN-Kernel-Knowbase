---
title: DMA buffer sharing in 3.3
url: https://lwn.net/Articles/474819/
date: "January 11, 2012"
category: "Device drivers-Support APIs; Direct memory access; Memory management"
author: "By Jonathan Corbet January 11, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 11, 2012

Back in August 2011, LWN [looked at](<https://lwn.net/Articles/454389/>) the DMA buffer sharing patch set posted by Marek Szyprowski. Since then, that patch has been picked up by Sumit Semwal, who modified it considerably in response to comments from a number of developers. The version of this patch that was merged for 3.3 differs enough from its predecessors that it merits another look here. 

The core idea remains the same, though: this mechanism allows DMA buffers to be shared between drivers that might otherwise be unaware of each other. The initial target use is sharing buffers between producers and consumers of video streams; a camera device, for example, could acquire a stream of frames into a series of buffers that are shared with the graphics adapter, enabling the capture and display of the data with no copying in the kernel. 

In the 3.3 sharing scheme, one driver will set itself up as an exporter of sharable buffers. That requires providing a set of callbacks to the buffer sharing code: 

```
struct dma_buf_ops {
    	int (*attach)(struct dma_buf *buf, struct device *dev,
    		      struct dma_buf_attachment *dma_attach);
    	void (*detach)(struct dma_buf *buf, struct dma_buf_attachment *dma_attach);
    	struct sg_table *(*map_dma_buf)(struct dma_buf_attachment *dma_attach,
    					enum dma_data_direction dir);
    	void (*unmap_dma_buf)(struct dma_buf_attachment *dma_attach, struct sg_table *sg);
    	void (*release)(struct dma_buf *);
        };
```

Briefly, `attach()` and `detach()` inform the exporting driver when others take or release references to the buffer. The `map_dma_buf()` and `unmap_dma_buf()` callbacks, instead, cause the buffer to be prepared (or unprepared) for DMA and pass ownership between drivers. A call to `release()` will be made when the last reference to the buffer is released. 

The exporting driver makes the buffer available with a call to: 

```
struct dma_buf *dma_buf_export(void *priv, struct dma_buf_ops *ops,
    			           size_t size, int flags);
```

Note that the `size` of the buffer is specified here, but there is no pointer to the buffer itself. In fact, the current version of the interface never passes around CPU-accessible buffer pointers at all. One of the actions performed by `dma_buf_export()` is the creation of an anonymous file to represent the buffer; `flags` is used to set the mode bits on that file. 

Since the file is anonymous, it is not visible to the rest of the kernel (or user space) in any useful way. Truly exporting the buffer, instead, requires obtaining a file descriptor for it and making that descriptor available to user space. The descriptor can be had with: 

```
int dma_buf_fd(struct dma_buf *dmabuf);
```

There is no standardized mechanism for passing that file descriptor to user space, so it seems likely that any subsystem implementing this functionality will add its own special `ioctl()` operation to get a buffer's file descriptor. The same is true for the act of passing a file descriptor to drivers that will share this buffer; it is something that will happen outside of the buffer-sharing API. 

A driver wishing to share a DMA buffer has to go through a series of calls after obtaining the corresponding file descriptor, the first of which is: 

```
struct dma_buf *dma_buf_get(int fd);
```

This function obtains a reference to the buffer and returns a `dma_buf` structure pointer that can be used with the other API calls to refer to the buffer. When the driver is finished with the buffer, it should be returned with a call to `dma_buf_put()`. 

The next step is to "attach" to the buffer with: 

```
struct dma_buf_attachment *dma_buf_attach(struct dma_buf *dmabuf,
    					      struct device *dev);
```

This function will allocate and fill in yet another structure: 

```
struct dma_buf_attachment {
    	struct dma_buf *dmabuf;
    	struct device *dev;
    	struct list_head node;
    	void *priv;
        };
```

That structure will then be passed to the exporting driver's `attach()` callback. There seems to be a couple of reasons for the existence of this step, the first of which is simply to let the exporting driver know about the consumers of the buffer. Beyond that, the `device` structure passed by the calling driver can contain a pointer (in its `dma_params` field) to one of these structures: 

```
struct device_dma_parameters {
    	unsigned int max_segment_size;
    	unsigned long segment_boundary_mask;
        };
```

The exporting driver should look at these constraints and ensure that the buffer it is exporting can satisfy them; if not, the `attach()` call should fail. If multiple drivers attach to the buffer, the exporting driver will need to allocate the buffer in a way that satisfies all of their constraints. 

The final step is to map the buffer for DMA: 

```
struct sg_table *dma_buf_map_attachment(struct dma_buf_attachment *attach,
    					    enum dma_data_direction direction);
```

This call turns into a call to the exporting driver's `map_dma_buf()` callback. If this call succeeds, the return value will be a scatterlist that can be used to program the DMA operation into the device. A successful return also means that the calling driver's device owns the buffer; it should not be touched by the CPU during this time. 

Note that mapping a buffer is an operation that can block for a number of reasons; if the buffer is busy elsewhere, for example. Also worth noting is that, until this call is made, the buffer need not necessarily be allocated anywhere. The exporting driver can wait until others have attached to the buffer so that it can see their DMA constraints and allocate the buffer accordingly. Of course, if the buffer lives in device memory or is otherwise constrained on the exporting side, it can be allocated sooner. 

After the DMA operation is completed, the sharing driver should unmap the buffer with: 

```
void dma_buf_unmap_attachment(struct dma_buf_attachment *attach,
    				  struct sg_table *sg_table);
```

That will, in turn, generate a call to the exporting driver's `unmap_dma_buf()` function. Detaching from the buffer (when it is no longer needed) can be done with: 

```
void dma_buf_detach(struct dma_buf *dmabuf, struct dma_buf_attachment *attach);
```

As might be expected, this function will call the exporting driver's `detach()` callback. 

As of 3.3, there are no users for this interface in the mainline kernel. There seems to be a fair amount of interest in using it, though, so Dave Airlie [pushed it into the mainline](<https://lwn.net/Articles/474828/>) with the idea that it would make the development of users easier. Some of those users can be seen (in an early form) in [Dave's drm-prime repository](<http://cgit.freedesktop.org/~airlied/linux/log/?h=drm-prime-dmabuf>) and [Rob Clark's OMAP4 tree](<https://github.com/robclark/kernel-omap4/commit/0cb96b221cc28da2dea586279a5a1b7d696573a1>).
