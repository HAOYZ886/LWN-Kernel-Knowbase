---
title: The videobuf2 API
url: https://lwn.net/Articles/447435/
date: "June 14, 2011"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: "By Jonathan Corbet June 14, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
June 14, 2011

Video4Linux2 drivers are charged with the task of acquiring video data from a sensor (via some sort of DMA controller, usually) and transferring those video frames to user space. The amount of data being moved makes performance a consideration; to that end, V4L2 has defined [a somewhat complex API](<https://lwn.net/Articles/240667/>) to handle streaming data. Implementing this API adds a certain amount of complexity to V4L2 drivers, but much of that complexity is the same from one driver to the next. To make life easier for driver writers (and their users), the "videobuf" subsystem was created to handle many of the details of streaming I/O buffer management. LWN [documented videobuf](<https://lwn.net/Articles/363349/>) toward the end of 2009; a version of that article also found its way into the kernel's documentation directory. 

This is the Linux kernel we're talking about, though, so 2009 is ancient history. The videobuf interface has since been superseded by videobuf2, which, while clearly inspired by the original, has a different way of doing things. So this article will be an attempt to get caught up with the current state of the art - an effort which will certainly inspire the creation of videobuf3 in the near future. 

Why videobuf2? The original videobuf worked well, but it turned out to be inflexible in a number of ways. The API varied considerably depending on which type of buffer was in use, and there was no real way for drivers to add their own specific memory management needs. Videobuf2 has created a more consistent API which allows for more customization in the drivers themselves. 

#### Buffers

Like the original videobuf, videobuf2 implements three basic types of buffers. **Vmalloc** buffers are allocated with `vmalloc()`, and are thus virtually contiguous in kernel space; drivers working with these buffers almost invariably end up copying the data once as it passes through the system. **Contiguous DMA** buffers are physically contiguous in memory, usually because the hardware cannot perform DMA to any other type of buffer. **S/G DMA** buffers are scattered through memory; they are the way to go if the hardware can do scatter/gather DMA. 

Depending on the type of buffer being used, the driver will need to include one of the following header files: 

```
#include <media/videobuf2-vmalloc.h>
        #include <media/videobuf2-dma-contig.h>
        #include <media/videobuf2-dma-sg.h>
```

One nice difference with videobuf2 is that a driver can be written to support more than one mode if need be. The above include files do not conflict with each other, and the videobuf2 interface is nearly the same for all three modes. 

A buffer for a video frame is represented by `struct vb2_buffer`, defined in `<media/videobuf2-core.h>`. Usually drivers will want to track per-buffer information of their own, so, in the usual style, the driver will define its own buffer type that includes a `vb2_buffer` instance. However, the videobuf2 authors didn't read Neil Brown's [Object-oriented design patterns in the kernel](<https://lwn.net/Articles/446317/>), so they don't have the driver allocate the resulting structures (in all fairness, said developers may offer lame excuses to the effect that said article had not been written at the time). That means that (1) the driver has to tell videobuf2 what the real size of the structure is, and (2) the `vb2_buffer` instance must be the first field in the driver's structure. Your editor may just post a patch to fix that in the near future. 

A videobuf2 driver must create a set of callbacks to give to the videobuf2 subsystem, five of which are specific to the management of buffers; they function in a similar (but not identical) manner to the videobuf versions. These callbacks are: 

```
int (*buf_init)(struct vb2_buffer *vb);
        int (*buf_prepare)(struct vb2_buffer *vb);
        void (*buf_queue)(struct vb2_buffer *vb);
        int (*buf_finish)(struct vb2_buffer *vb);
        void (*buf_cleanup)(struct vb2_buffer *vb);
```

Videobuf2 will call `buf_init()` for each new buffer after it has been allocated; the driver can then perform any additional initialization that needs to be done. Returning a failure code will abort the setup of the buffer queue. 

The `buf_prepare()` callback is invoked when user space queues the buffer (i.e. in response to a `VIDIOC_QBUF` operation); it should do any preparation which might be required before the buffer is used for I/O. `buf_queue()`, instead, is called to pass actual ownership of the buffer to the driver, indicating that it's time to actually start I/O on it. 

`buf_finish()` will be called just before the buffer is passed back to user space. One might question the need for this callback; the driver already knows when a buffer has a new frame for user space and, indeed, must tell videobuf2 about it. One possible answer is that the completion of I/O to the buffer is often handled in interrupt context, while `buf_finish()` will be called in process context. 

Finally, `buf_cleanup()` is called just before a buffer is freed so that the driver can do any additional cleanup work required. 

#### Other videobuf2 callbacks

In the original videobuf, the only callbacks provided by the driver had to do with buffer management. With videobuf2, there are a few others, starting with: 

```
int (*queue_setup)(struct vb2_queue *q, unsigned int *num_buffers,
    		       unsigned int *num_planes, unsigned long sizes[],
    		       void *alloc_ctxs[]);
```

This function, called in response to a `VIDIOC_REQBUFS` `ioctl()` operation, allows the driver to influence how the buffer queue is set up. The `vb2_queue` structure describes the queue as a whole; we'll see more about it shortly. The `num_buffers` argument is the number of buffers requested by the application; the driver can modify it if needed. `num_planes` contains the number of distinct video planes needed to hold a frame; for packed formats, it will be one, but it will be larger if planar formats are in use (see [this article](<https://lwn.net/Articles/218798/>) for more information on formats). The `sizes` array should contain the size (in bytes) of each plane. 

The `alloc_ctxs` field contains the "allocation context" for each plane; it is currently only used by the contiguous DMA mode. Drivers which use contiguous DMA should call: 

```
void *vb2_dma_contig_init_ctx(struct device *dev)
```

to get that context; it needs to be remembered and passed to `vb2_dma_contig_cleanup_ctx()` when the driver shuts down. 

There are two callbacks which tell the driver when to start and stop acquiring video data: 

```
int (*start_streaming)(struct vb2_queue *q);
        int (*stop_streaming)(struct vb2_queue *q);
```

Videobuf2 will call `start_streaming()` whenever user space wants to start grabbing data. That may happen in response to a `VIDIOC_STREAMON` `ioctl()`, but the videobuf2 implementation of the `read()` system call can also use it. A call to `stop_streaming()` will be made when user space no longer wants data; this callback should not return until DMA has been stopped. It's worth noting that, after the `stop_streaming()` call, videobuf2 will grab back all buffers passed to the driver; the driver should forget any references it may have to those buffers. 

#### Locking

The final two callbacks deserve a section of their own. The locking model used for videobuf2 is not documented all that well; from what your editor has been able to gather, the rules are mostly like the following. The driver needs a lock (probably a mutex) which governs access to the device as a whole. Then: 

  * Calls that the driver makes directly into videobuf2 should be made with the device lock held. 

  * Callbacks to the driver from videobuf2 should assume that the lock has already been taken. For example, a `start_streaming()` call will result from a user-space request to acquire data which will have necessarily passed through the driver before videobuf2 gets involved, so, by the time `start_streaming()` is called, the device lock will be held. 

With that context, one needs to consider one little problem: a user-space invocation of `VIDIOC_DQBUF`, meant to get a buffer full of data from the kernel, may block waiting for a buffer to become available. That, in turn, may not happen until user space (perhaps in a different thread) hands a buffer back to the kernel with `VIDIOC_QBUF`. If the first call blocks with the lock held, the application will end up waiting for a very long time. For this situation, videobuf2 provides two more callbacks: 

```
void (*wait_prepare)(struct vb2_queue *q);
        void (*wait_finish)(struct vb2_queue *q);
```

Before a `VIDIOC_DQBUF` operation blocks to wait for a buffer, it will call `wait_prepare()` to release the device lock; once it stops waiting, a call to `wait_finish()` will reacquire the lock. It might have been better to call them `release_lock()` and `reacquire_lock()`, but so it goes. 

#### Queue setup

With all of the above in place, the driver can introduce itself to videobuf2. The first step is to fill in a `vb2_ops` structure with all of the callbacks described above: 

```
static const struct vb2_ops my_special_callbacks = {
    		.queue_setup = my_special_queue_setup,
    		/* ... */
        };
```

Then, to set up a videobuf2 queue (normally done either at device registration or device open time), the driver should allocate a `vb2_queue` structure: 

```
struct vb2_queue {
    	enum v4l2_buf_type		type;
    	unsigned int			io_modes;
    	unsigned int			io_flags;
    	const struct vb2_ops		*ops;
    	const struct vb2_mem_ops	*mem_ops;
    	void				*drv_priv;
    	unsigned int			buf_struct_size;
    	/* Lots of private stuff omitted */
        };
```

The structure should be zeroed, and the above fields filled in. `type` is the buffer type, usually `V4L2_BUF_TYPE_VIDEO_CAPTURE`. `io_modes` is a bitmask describing what types of buffers can be handled: 

  * `VB2_MMAP`: buffers allocated within the kernel and accessed via `mmap()`; vmalloc and contiguous DMA buffers will usually be of this type. 

  * `VB2_USERPTR`: buffers allocated in user space. Normally, only devices which can do scatter/gather I/O can deal with user-space buffers. Interestingly, videobuf2 supports contiguous buffers allocated by user space; the only way to get those, though, is to use some sort of special mechanism like the out-of-tree Android "pmem" driver. Contiguous I/O to huge pages is _not_ supported. 

  * `VB2_READ`, `VB2_WRITE`: user-space buffers provided via the `read()` and `write()` system calls. 

The `mem_ops` field is where the driver tells videobuf2 what kind of buffers it is actually using; it should be set to one of `vb2_vmalloc_memops`, `vb2_dma_contig_memops`, or `vb2_dma_sg_memops`. If a situation arises where none of the existing modes works for a specific device, the driver author can create a custom set of `vb2_mem_ops` to meet that need; as of this writing, there are no drivers in the mainline kernel that have supplied their own memory operations. 

Finally, `drv_priv` is a place where the driver can stash a pointer of its own (usually to its device structure), and `buf_struct_size` is where the driver tells videobuf2 how big its buffer structure is. Once the structure has been filled in, it can be passed to: 

```
int vb2_queue_init(struct vb2_queue *q);
```

A call to `vb2_queue_release()` should be made when the device is shut down. 

#### Device operations

Now most of the infrastructure for videobuf2 is in place; what's left is (1) making the connection between user space operations and their implementation in videobuf2, and (2) actually performing I/O. For the first step, the driver needs to create V4L2 callbacks for the various I/O-oriented `ioctl()` calls in the usual way. Most of these callbacks, though, can simply acquire the device lock and call directly into videobuf2. There is a whole set of functions to be used in this role: 

```
int vb2_querybuf(struct vb2_queue *q, struct v4l2_buffer *b);
        int vb2_reqbufs(struct vb2_queue *q, struct v4l2_requestbuffers *req);
        int vb2_qbuf(struct vb2_queue *q, struct v4l2_buffer *b);
        int vb2_dqbuf(struct vb2_queue *q, struct v4l2_buffer *b, 
    		  bool nonblocking);
        int vb2_streamon(struct vb2_queue *q, enum v4l2_buf_type type);
        int vb2_streamoff(struct vb2_queue *q, enum v4l2_buf_type type);
```

A similar thing needs to be done with a number of entries in the driver's `file_operations` structure. To that end, videobuf2 provides: 

```
int vb2_mmap(struct vb2_queue *q, struct vm_area_struct *vma);
        unsigned int vb2_poll(struct vb2_queue *q, struct file *file, 
    			  poll_table *wait);
        size_t vb2_read(struct vb2_queue *q, char __user *data, size_t count,
    		    loff_t *ppos, int nonblock);
        size_t vb2_write(struct vb2_queue *q, char __user *data, size_t count,
    		     loff_t *ppos, int nonblock);
```

This, of course, is where the payoff happens: all the grungy details of buffer management, implementing `mmap()`, and more are handled in videobuf2 with no further mess. So the driver code is significantly shorter, the core code is known to be well debugged, and devices behave more consistently toward user space. 

There's only one little bit of work left to do: actually getting the data into the buffers. For vmalloc buffers, that task is usually accomplished with something like `memcpy()`; one useful helper function in this case is: 

```
void *vb2_plane_vaddr(struct vb2_buffer *vb, unsigned int plane_no);
```

which returns the kernel-space virtual address for the given `plane` in the buffer. 

Contiguous DMA drivers will need to get the bus address to hand to the device for I/O; that address can be had with: 

```
dma_addr_t vb2_dma_contig_plane_paddr(struct vb2_buffer *vb, 
                                              unsigned int plane_no);
```

For scatter/gather drivers, the interface is just a bit more complex: 

```
struct vb2_dma_sg_desc {
    	unsigned long		size;
    	unsigned int		num_pages;
    	struct scatterlist	*sglist;
        };
    
        struct vb2_dma_sg_desc *vb2_dma_sg_plane_desc(struct vb2_buffer *vb, 
    						  unsigned int plane_no);
```

In this case, the driver can obtain the scatterlist from videobuf2 which can then be used to program the device for I/O. 

For all three cases, the buffer will eventually be filled with frame data which needs to be passed back to user space. The `vb2_buffer` structure contains `v4l2_buffer` structure (called `v4l2_buf`) which should be filled in with the usual information: image size, sequence number, time stamp, etc. Then the buffer should be passed to: 

```
void vb2_buffer_done(struct vb2_buffer *vb, enum vb2_buffer_state state);
```

The `state` parameter should be passed as `VB2_BUF_STATE_DONE` for a normal completion, or `VB2_BUF_STATE_ERROR` if something went wrong. Happily videobuf2, unlike its predecessor, does not get upset if buffers are completed in an arbitrary order. 

And that is a fairly complete summary of the state of the art with regard to the videobuf2 API. Those wanting to see the complete interface can find it in the include files mentioned above. As always, the "virtual video" driver (in `drivers/media/video/vivi.c`) serves as a sort of showcase for almost everything Video4Linux2 can do; it uses videobuf with vmalloc-style buffers. As of this writing, there is no videobuf3 in sight, so hopefully the information found here will remain useful for a while.
