---
title: Sharing buffers between devices
url: https://lwn.net/Articles/454389/
date: "August 15, 2011"
category: "Device drivers-Support APIs"
author: "By Jonathan Corbet August 15, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 15, 2011

CPUs may not have gotten hugely faster in recent years, but they have gained in other ways; a typical system-on-chip (SoC) device now has a number of peripherals which would qualify as reasonably powerful CPUs in their own right. More powerful devices with direct access to the memory bus can take on more demanding tasks. For example, an image frame captured from a camera device can often be passed directly to the graphics processor for display without all of the user-space processing that was once necessary. Increasingly, the CPU's job looks like that of a shop foreman whose main concern is keeping all of the other processors busy. 

The foreman's job will be easier if the various devices under its control can communicate easily with each other. One useful addition in this area might be the [buffer sharing patch set](<https://lwn.net/Articles/455098/>) recently posted by Marek Szyprowski. The idea here is to make it possible for multiple kernel subsystems to share buffers under the control of user space. With this type of feature, applications could wire kernel subsystems together in problem-specific ways then get out of the way, letting the devices involved process the data as it passes through. 

There are (at least) a couple of challenges which must be dealt with to make this kind of functionality safe to export to applications. One is that the application should not be able to "create" buffers at arbitrary kernel addresses. Indeed, kernel-space addresses should not be visible to user space at all, so the kernel must provide some other way for an application to refer to a specific buffer. The other is that shared buffers must not go away until all users have let go of it. A buffer may be created by a specific device driver, but it must persist, even if the device is closed, until nobody else expects it to be there. 

The mechanism added in this patch set (this part in particular is credited to Tomasz Stanislawski) is relatively simple - though it will probably get more complex in the future. Kernel code wanting to make a buffer available to other parts of the kernel via user space starts by filling in one of these structures: 

```
struct shrbuf {
        	void (*get)(struct shrbuf *);
        	void (*put)(struct shrbuf *);
        	unsigned long dma_addr;
        	unsigned long size;
        };
```

One could immediately raise a number of complaints about this structure: the address should be a `dma_addr_t`, there's no reason not to put the kernel virtual address there, only physically-contiguous buffers are allowed, etc. It also seems like there could be value in the ability to annotate the state of the buffer (filled or empty, for example) and possibly signal another thread when that state changes. But it's worth remembering that this is an explicitly proof-of-concept patch posting and a lot of things will change. In particular, the eventual plan is to pass a scatterlist around instead of a single physical address. 

The `get()` and `put()` functions are important: they manage reference counts to the buffer, which must continue to exist until that count goes to zero. Any subsystem depending on a buffer's continued existence should hold a reference to that buffer. The `put()` function should release the buffer when the last reference is dropped. 

Once this structure exists, it can be passed to: 

```
int shrbuf_export(struct shrbuf *sb);
```

The return value (if all goes well) will be an integer file descriptor which can be handed to user space. This file descriptor embodies a reference to the buffer, which now will not be released before the file descriptor is closed. Other than closing it, there is very little that the application can do with the descriptor other than give it to another kernel subsystem; attempts to read from or write to it will fail, for example. 

If a kernel subsystem receives a file descriptor which is purported to represent a kernel buffer, it can pass that descriptor to: 

```
struct shrbuf *shrbuf_import(int fd);
```

The return value will be the same `shrbuf` structure (or an `ERR_PTR()` error value for a file descriptor of the wrong type). A reference is taken on the structure before returning it, so the recipient should call `put()` at some future time to release it. 

The patch set includes a new Video4Linux2 `ioctl()` command (`VIDIOC_EXPBUF`) enabling the exporting of buffers as file descriptors; a couple of capture drivers have been augmented to support this functionality. No examples of the other side (importing a buffer) have been posted yet. 

There has been relatively little commentary on the patch set so far, possibly because it was posted to a couple of relatively obscure mailing lists. It has the look of functionality that could be useful beyond one or two kernel subsystems, though. It would probably make sense for the next iteration, which presumably will have more of the anticipated functionality built into it, to be distributed more widely for review.
