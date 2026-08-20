---
title: Memory management for graphics processors
url: https://lwn.net/Articles/257417/
date: "November 6, 2007"
category: "Device drivers-Video; Memory management-Video memory; Translation table maps TTM"
author: "By Jonathan Corbet November 6, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 6, 2007

The management of video hardware has long been an area of weakness in the Linux system (and free operating systems in general). The X Window System tends to get a lot of the blame for problems in this area, but the truth of the matter is that the problems are more widespread and the kernel has never made it easy for X to do this job properly. Graphics processors (GPUs) have gotten steadily more powerful, to the point that, by some measures, they are the fastest processor on most systems, but kernel support for the programming of GPUs has lagged behind. A lot of work is being done to remedy this situation, though, and an important component of that work has just been put forward for inclusion into the mainline kernel. 

Once upon a time, video memory comprised a simple frame buffer from which pixels were sent to the display; it was up to the system's CPU to put useful data into that frame buffer. With contemporary GPUs, the memory situation has gotten more complex; a typical GPU can work with a few different types of memory: 

  * Video RAM (VRAM) is high-speed memory installed directly on the video card. It is usually visible on the system's PCI bus, but that need not be the case. There is likely to be a frame buffer in this memory, but many other kinds of data live there as well. 

  * In many lower-end systems, the "video RAM" is actually a dedicated section of general-purpose system memory. That RAM is set aside for the use of the GPU and is not available for other purposes. Even adapters with their own VRAM may have a dedicated RAM region as well. 

  * Video adapters contain a simple memory management unit (the graphics address remapping table or GART) which can be used to map various pages of system memory into the GPU's address space. The result is that, at any time, an arbitrary (scattered) subset of the system's RAM pages are accessible to the GPU. 

Each type of video memory has different characteristics and constraints. Some are faster to work with (for the CPU or the GPU) than others. Some types of VRAM might not be directly addressable by the CPU. Memory may or may not be cache coherent - a distinction which requires careful programming to avoid data corruption and performance problems. And graphical applications may want to work with much larger amounts of video memory than can be made visible to the GPU at any given time. 

All of this presents a memory management problem which, while being similar to the management needs of system RAM, has its own special constraints. So the graphics developers have been saying for years that Linux needs a proper manager for GPU-accessible memory. But, for years, we have done without that memory manager, with the result that this management task has been performed by an ugly combination of code in the X server, the kernel, and, often, in proprietary drivers. Happily, it would appear that those days are coming to an end, thanks to the creation of the translation-table maps (TTM) module written primarily by Thomas Hellstrom, Eric Anholt, and Dave Airlie. The TTM code provides a general-purpose memory manager aimed at the needs of GPUs and graphical clients. 

The core object managed by TTM, from the point of view of user space, is the "buffer object." A buffer object is a chunk of memory allocated by an application, and possibly shared among a number of different applications. It contains a region of memory which, at some point, may be operated on by the GPU. A buffer object is guaranteed not to vanish as long as some application maintains a reference to it, but the location of that buffer is subject to change. 

Once an application creates a buffer object, it can map that object into its address space. Depending on where the buffer is currently located, this mapping may require relocating the buffer into a type of memory which is addressable by the CPU (more accurately, a page fault when the application tries to access the mapped buffer would force that move). Cache coherency issues must be handled as well, of course. 

There will come a time when this buffer must be made available to the GPU for some sort of operation. The TTM layer provides a special "validate" `ioctl()` to prepare buffers for processing; validating a buffer could, again, involve moving it or setting up a GART mapping for it. The address by which the GPU will access the buffer will not be known until it is validated; after validation, the buffer will not be moved out of the GPU's address space until it is no longer being operated on. 

That means that the kernel has to know when processing on a given buffer has completed; applications, too, need to know that. To this end, the TTM layer provides "fence" objects. A fence is a special operation which is placed into the GPU's command FIFO. When the fence is executed, it raises a signal to indicate that all instructions enqueued before the fence have now been executed, and that the GPU will no longer be accessing any associated buffers. How the signaling works is very much dependent on the GPU; it could raise an interrupt or simply write a value to a special memory location. When a fence signals, any associated buffers are marked as no longer being referenced by the GPU, and any interested user-space processes are notified. 

A busy system might feature a number of graphical applications, each of which is trying to feed buffers to the GPU at any given time. It is not at all unlikely that the demands for GPU-addressable buffers will exceed the amount of memory which the GPU can actually reach. So the TTM layer will have to move buffers around in response to incoming requests. For GART-mapped buffers, it may be a simple matter of unmapping pages from buffers which are not currently validated for GPU operations. In other cases, the contents of the buffers may have to be explicitly copied to another type of memory, possibly using the GPU's hardware to do so. In such cases, the buffers must first be invalidated in the page tables of any user-space process which has mapped it to ensure that the buffer will not be written to during the move. In other words, the TTM really does become an extension of the system's memory management code. 

The next question which is bound to come up is: what happens when graphical applications want to use more video memory than the system as a whole can provide? Normal system RAM pages which are used as video memory are locked in place (and unavailable for other uses), so there must be a clear limit on the number of such pages which can be created. The current solution to this problem is to cap the number of such pages at a fraction of the available low memory \- up to 1GB on a 4GB, 32-bit system. It would be nice to be able to extend this memory by writing unused pages to swap, but the Linux swap implementation is not meant to work with pages owned by the kernel. The long-term plan would appear to be to let the X server create a large virtual range with `mmap()`, which would then be swappable. That functionality has not yet been implemented, though. 

There is a lot more to the TTM code than has been described here; some more information can be found in [this TTM overview [PDF]](<http://www.tungstengraphics.com/mm.pdf>). For the time being, this code works with a patched version of the Intel i915 driver, with other drivers to be added in the future. [TTM has been proposed](<http://lwn.net/Articles/257230/>) for inclusion into -mm now and merger into the mainline for 2.6.25. The main issue between now and then will be the evaluation of the user-space API, which will be hard to change once this code is merged. Unfortunately, documentation for this API appears to be scarce at the moment.
