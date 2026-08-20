---
title: "Video4Linux2 part 6b: Streaming I/O"
url: https://lwn.net/Articles/240667/
date: "July 5, 2007"
category: "Device drivers-Video4Linux2; Video4Linux2"
author: 
---

The [LWN.net Video4Linux2 API series](<http://lwn.net/Articles/203924/>).   
---  
The [previous installment](<http://lwn.net/Articles/235023/>) in this series discussed how to transfer video frames with the `read()` and `write()` system calls. Such an implementation can get the basic job done, but it is not normally the preferred method for performing video I/O. For the highest performance and the best information transfer, video drivers should support the V4L2 streaming I/O API. 

With the `read()` and `write()` methods, each video frame is copied between user and kernel space as part of the I/O operation. When streaming I/O is being used, instead, this copying does not happen; instead, the application and the driver exchange pointers to buffers. These buffers will be mapped into the application's address space, making it possible to perform zero-copy frame I/O. There are two different types of streaming I/O buffers: 

  * Memory-mapped buffers (type `V4L2_MEMORY_MMAP`) are allocated in kernel space; the application maps them into its address space with the `mmap()` system call. The buffers can be large, contiguous DMA buffers, virtual buffers created with `vmalloc()`, or, if the hardware supports it, they can be located directly in the video device's I/O memory. 

  * User-space buffers (`V4L2_MEMORY_USERPTR`) are allocated by the application in user space. Clearly, in this situation, no `mmap()` call is required, but the driver may have to work harder to support efficient I/O to user-space buffers. 

Note that drivers are not required to support streaming I/O, and, if they do support streaming, they do not have to handle both buffer types. A driver which is more flexible will support more applications; in practice, it seems that most applications are written to use memory-mapped buffers. It is not possible to use both types of buffer simultaneously. 

We will now delve into the numerous grungy details involved in supporting streaming I/O. Any Video4Linux2 driver writer will need to understand this API; it is worth noting, however, that there is a higher-level API which can help in the writing of streaming drivers. That layer (called video-buf) can make life easier when the underlying device can support scatter/gather I/O. The video-buf API will be discussed in a future installment. 

Drivers which support streaming I/O should inform the application of that fact by setting the `V4L2_CAP_STREAMING` flag in their `vidioc_querycap()` method. Note that there is no way to describe which buffer types are supported; that comes later. 

### The v4l2_buffer structure

When streaming I/O is active, frames are passed between the application and the driver in the form of `struct v4l2_buffer`. This structure is a complicated beast which will take a while to describe. A good starting point is to note that there are three fundamental states that a buffer can be in: 

  * In the driver's incoming queue. Buffers are placed in this queue by the application in the expectation that the driver will do something useful with them. For a video capture device, buffers in the incoming queue will be empty, waiting for the driver to fill them with video data. For an output device, these buffers will have frame data to be sent to the device. 

  * In the driver's outgoing queue. These buffers have been processed by the driver and are waiting for the application to claim them. For capture devices, outgoing buffers will have new frame data; for output devices, these buffers are empty. 

  * In neither queue. In this state, the buffer is owned by user space and will not normally be touched by the driver. This is the only time that the application should do anything with the buffer. We'll call this the "user space" state. 

These states, and the operations which cause transitions between them, come together as shown in the diagram below: 

> ![\[Buffer states\]](https://static.lwn.net/images/ns/kernel/v4l2_buffers.png)

The actual `v4l2_buffer` structure looks like this: 

```
struct v4l2_buffer
        {
    	__u32			index;
    	enum v4l2_buf_type      type;
    	__u32			bytesused;
    	__u32			flags;
    	enum v4l2_field		field;
    	struct timeval		timestamp;
    	struct v4l2_timecode	timecode;
    	__u32			sequence;
    
    	/* memory location */
    	enum v4l2_memory        memory;
    	union {
    		__u32           offset;
    		unsigned long   userptr;
    	} m;
    	__u32			length;
    	__u32			input;
    	__u32			reserved;
        };
```

The `index` field is a sequence number identifying the buffer; it is only used with memory-mapped buffers. Like other objects which can be enumerated in the V4L2 interface, memory-mapped buffers start with index 0 and go up sequentially from there. The `type` field describes the type of the buffer, usually `V4L2_BUF_TYPE_VIDEO_CAPTURE` or `V4L2_BUF_TYPE_VIDEO_OUTPUT`. 

The size of the buffer is given by `length`, which is in bytes. The size of the image data contained within the buffer is found in `bytesused`; obviously `bytesused <= length`. For capture devices, the driver will set `bytesused`; for output devices the application must set this field. 

`field` describes which field of an image is stored in the buffer; fields were discussed in [part 5a](<http://lwn.net/Articles/218798/>) of this series. 

The `timestamp` field, for input devices, tells when the frame was captured. For output devices, the driver should not send the frame out before the time found in this field; a `timestamp` of zero means "as soon as possible." The driver will set `timestamp` to the time that the first byte of the frame was transferred to the device - or as close to that time as it can get. `timecode` can be used to hold a timecode value, useful for video editing applications; see [this table](<http://v4l2spec.bytesex.org/spec/x5953.htm#V4L2-TIMECODE>) for details on timecodes. 

The driver maintains a incrementing count of frames passing through the device; it stores the current sequence number in `sequence` as each frame is transferred. For input devices, the application can watch this field to detect dropped frames. 

`memory` tells whether the buffer is memory-mapped or user-space. For memory-mapped buffers, `m.offset` describes where the buffer is to be found. The specification describes it as ""the offset of the buffer from the start of the device memory"", but the truth of the matter is that it is simply a magic cookie that the application can pass to `mmap()` to specify which buffer is being mapped. For user-space buffers, instead, `m.userptr` is the user-space address of the buffer. 

The `input` field can be used to quickly switch between inputs on a capture device - assuming the device supports quick switching between frames. The `reserved` field should be set to zero. 

Finally, there are several `flags` defined: 

  * `V4L2_BUF_FLAG_MAPPED` indicates that the buffer has been mapped into user space. It is only applicable to memory-mapped buffers. 

  * `V4L2_BUF_FLAG_QUEUED`: the buffer is in the driver's incoming queue. 

  * `V4L2_BUF_FLAG_DONE`: the buffer is in the driver's outgoing queue. 

  * `V4L2_BUF_FLAG_KEYFRAME`: the buffer holds a key frame - useful in compressed streams. 

  * `V4L2_BUF_FLAG_PFRAME` and `V4L2_BUF_FLAG_BFRAME` are also used with compressed streams; they indicated predicted or difference frames. 

  * `V4L2_BUF_FLAG_TIMECODE`: the `timecode` field is valid. 

  * `V4L2_BUF_FLAG_INPUT`: the `input` field is valid. 

### Buffer setup

Once a streaming application has performed its basic setup, it will turn to the task of organizing its I/O buffers. The first step is to establish a set of buffers with the `VIDIOC_REQBUFS` `ioctl()`, which is turned by V4L2 into a call to the driver's `vidioc_reqbufs()` method: 

```
int (*vidioc_reqbufs) (struct file *file, void *private_data, 
    			   struct v4l2_requestbuffers *req);
```

Everything of interest will be in the `v4l2_requestbuffers` structure, which looks like this: 

```
struct v4l2_requestbuffers
        {
    	__u32			count;
    	enum v4l2_buf_type      type;
    	enum v4l2_memory        memory;
    	__u32			reserved[2];
        };
```

The `type` field describes the type of I/O to be done; it will usually be either `V4L2_BUF_TYPE_VIDEO_CAPTURE` for a video acquisition device or `V4L2_BUF_TYPE_VIDEO_OUTPUT` for an output device. There are other types, but they are beyond the scope of this article. 

If the application wants to use memory-mapped buffers, it will set `memory` to `V4L2_MEMORY_MMAP` and `count` to the number of buffers it wants to use. If the driver does not support memory-mapped buffers, it should return `-EINVAL`. Otherwise, it should allocate the requested buffers internally and return zero. On return, the application will expect the buffers to exist, so any part of the task which could fail (memory allocation, for example) should be done at this stage. 

Note that the driver is not required to allocate exactly the requested number of buffers. In many cases there is a minimum number of buffers which makes sense; if the application requests fewer than the minimum, it may actually get more buffers than it asked for. In your editor's experience, for example, the `mplayer` application will request two buffers, which makes it susceptible to overruns (and thus lost frames) if things slow down in user space. By enforcing a higher minimum buffer count (adjustable with a module parameter), the cafe_ccic driver is able to make the streaming I/O path a little more robust. The `count` field should be set to the number of buffers actually allocated before the method returns. 

Setting `count` to zero is a way for the application to request that all existing buffers be released. In this case, the driver must stop any DMA operations before freeing the buffers or terrible things could happen. It is also not possible to free buffers if they are current mapped into user space. 

If, instead, user-space buffers are to be used, the only fields which matter are the buffer `type` and a value of `V4L2_MEMORY_USERPTR` in the `memory` field. The application need not specify the number of buffers that it intends to use; since the allocation will be happening in user space, the driver need not care. If the driver supports user-space buffers, it need only note that the application will be using this feature and return zero; otherwise the usual `-EINVAL` return is called for. 

The `VIDIOC_REQBUFS` command is the only way for an application to discover which types of streaming I/O buffer are supported by a given driver. 

### Mapping buffers into user space

If user-space buffers are being used, the driver will not see any more buffer-related calls until the application starts putting buffers on the incoming queue. Memory-mapped buffers require more setup, though. The application will typically step through each allocated buffer and map it into its address space. The first stop is the `VIDIOC_QUERYBUF` command, which becomes a call to the driver's `vidioc_querybuf()` method: 

```
int (*vidioc_querybuf)(struct file *file, void *private_data, 
                               struct v4l2_buffer *buf);
```

On entry to this method, the only fields of `buf` which will be set are `type` (which should be checked against the type specified when the buffers were allocated) and `index`, which identifies the specific buffer. The driver should make sure that `index` makes sense and fill in the rest of the fields in `buf`. Typically drivers store an array of `v4l2_buffer` structures internally, so the core of a `vidioc_querybuf()` method is just a structure assignment. 

The only way for an application to access memory-mapped buffers is to map them into their address space, so a `vidioc_querybuf()` call will typically be followed by a call to the driver's `mmap()` method - this method, remember, is stored in the `fops` field of the `video_device` structure associated with this device. How the driver handles `mmap()` will depend on just how the buffers are set up in the kernel. If the buffer can be mapped up front with `remap_pfn_range()` or `remap_vmalloc_range()`, that should be done at this time. For buffers in kernel space, pages can also be mapped individually at page-fault time by setting up a `nopage()` method in the usual way. A good discussion of handling `mmap()` can be found in [Linux Device Drivers](<https://lwn.net/Kernel/LDD3/>) for those who need it. 

When `mmap()` is called, the VMA structure passed in should have the address of one of your buffers in the `vm_pgoff` field - right-shifted by `PAGE_SHIFT`, of course. It should, in particular, be the `offset` value that your driver returned in response to a `VIDIOC_QUERYBUF` call. Please iterate through your list of buffers and be sure that the incoming address matches one of them; video drivers should not be a means by which hostile programs can map arbitrary regions of memory. 

The `offset` value you provide can be almost anything, incidentally. Some drivers just return `(index<<PAGE_SHIFT)`, meaning that the incoming `vm_pgoff` field should just be the buffer index. The one thing you should _not_ do is store the actual kernel-space address of the buffer in `offset`; leaking kernel addresses into user space is never a good idea. 

When user space maps a buffer, the driver should set the `V4L2_BUF_FLAG_MAPPED` flag in the associated `v4l2_buffer` structure. It must also set up `open()` and `close()` VMA operations so that it can track the number of processes which have the buffer mapped. As long as this buffer remains mapped somewhere, it cannot be released back to the kernel. If the mapping count of one or more buffers drops to zero, the driver should also stop any in-progress I/O, as there will be no process which can make use of it. 

### Streaming I/O

So far we have looked at a lot of setup without the transfer of a single frame. We're getting closer, but there is one more step which must happen first. When the application obtains buffers with `VIDIOC_REQBUFS`, those buffers are all in the user-space state; if they are user-space buffers, they do not really even exist yet. Before the application can start streaming I/O, it must put at least one buffer into the driver's incoming queue; for an output device, of course, those buffers should also be filled with valid frame data. 

To enqueue a buffer, the application will issue a `VIDIOC_QBUF` `ioctl()`, which the V4L2 maps into a call to the driver's `vidioc_qbuf()` method: 

```
int (*vidioc_qbuf) (struct file *file, void *private_data, 
                            struct v4l2_buffer *buf);
```

For memory-mapped buffers, once again, only the `type` and `index` fields of `buf` are valid. The driver can just perform the obvious checks (`type` and `index` make sense, the buffer is not already on one of the driver's queues, the buffer is mapped, etc.), put the buffer on its incoming queue (setting the `V4L2_BUF_FLAG_QUEUED` flag), and return. 

User-space buffers can be more complicated at this point, because the driver will have never seen this buffer before. When using this method, applications are allowed to pass a different address every time they enqueue a buffer, so the driver can do no setup ahead of time. If your driver is bouncing frames through a kernel-space buffer, it need only make a note of the user-space address provided by the application. If you are trying to DMA the data directly into user-space, however, life is significantly more challenging. 

To ship data directly into user space, the driver must first fault in all of the pages of the buffer and lock them into place; `get_user_pages()` is the tool to use for this job. Note that this function can perform significant amounts of memory allocation and disk I/O \- it could block for a long time. You will need to take care to ensure that important driver functions do not stall while `get_user_pages()`, which can block for long enough for many video frames to go by, does its thing. 

Then there is the matter of telling the device to transfer image data to (or from) the user-space buffer. This buffer will not be contiguous in physical memory - it will, instead, be broken up into a large number of separate 4096-byte pages (on most architectures). Clearly, the device will have to be able to do scatter/gather DMA operations. If the device transfers full video frames at once, it will need to accept a scatterlist which holds a great many pages; a VGA-resolution image in a 16-bit format requires 150 pages. As the image size grows, so will the size of the scatterlist. The V4L2 specification says: 

If required by the hardware the driver swaps memory pages within physical memory to create a continuous area of memory. This happens transparently to the application in the virtual memory subsystem of the kernel. 

Your editor, however, is unwilling to recommend that driver writers attempt this kind of deep virtual memory trickery. A more promising approach could be to require user-space buffers to be located in hugetlb pages, but no drivers do that now. 

If your device transfers images in smaller pieces (a USB camera, for example), direct DMA to user space may be easier to set up. In any case, when faced with the challenges of supporting direct I/O to user-space buffers, the driver writer should (1) be sure that it is worth the trouble, given that applications tend to expect to use memory-mapped buffers anyway, and (2) make use of the video-buf layer, which can handle some of the pain for you. 

Once streaming I/O starts, the driver will grab buffers from its incoming queue, have the device perform the requested transfer, then move the buffer to the outgoing queue. The buffer flags should be adjusted accordingly when this transition happens; fields like the sequence number and time stamp should also be filled in at this time. Eventually the application will want to claim buffers in the outgoing queue, returning them to the user-space state. That is the job of `VIDIOC_DQBUF`, which becomes a call to: 

```
int (*vidioc_dqbuf) (struct file *file, void *private_data, 
                             struct v4l2_buffer *buf);
```

Here, the driver will remove the first buffer from the outgoing queue, storing the relevant information in `*buf`. Normally, if the outgoing queue is empty, this call should block until a buffer becomes available. V4L2 drivers are expected to handle non-blocking I/O, though, so if the video device has been opened with `O_NONBLOCK`, the driver should return `-EAGAIN` in the empty-queue case. Needless to say, this requirement also implies that the driver must support `poll()` for streaming I/O. 

The only remaining step is to actually tell the device to start performing streaming I/O. The Video4Linux2 driver methods for this task are: 

```
int (*vidioc_streamon) (struct file *file, void *private_data, 
                                enum v4l2_buf_type type);
        int (*vidioc_streamoff)(struct file *file, void *private_data, 
        	                    enum v4l2_buf_type type);
```

The call to `vidioc_streamon()` should start the device after checking that `type` makes sense. The driver can, if need be, require that a certain number of buffers be in the incoming queue before streaming can be started. 

When the application is done it should generate a call to `vidioc_streamoff()`, which must stop the device. The driver should also remove all buffers from both the incoming and outgoing queues, leaving them all in the user-space state. Of course, the driver must be prepared for the application to simply close the device without stopping streaming first.
