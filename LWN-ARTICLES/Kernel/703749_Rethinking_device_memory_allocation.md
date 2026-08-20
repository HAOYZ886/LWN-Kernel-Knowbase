---
title: Rethinking device memory allocation
url: https://lwn.net/Articles/703749/
date: "October 19, 2016"
category: "Device drivers-Support APIs"
author: "By Jake Edge October 19, 2016 X.Org Developers Conference"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
October 19, 2016

* * *

[X.Org Developers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-X.Org_Developers_Conference>)

James Jones started his [2016 X.Org Developers Conference](<https://www.x.org/wiki/Events/XDC2016/>) (XDC) talk by saying that he would like to make some real progress at the conference on creating a user-space API for allocating memory that is also accessible by various devices. His talk on day one of the conference set the stage for a meeting of interested developers on day two. By day three, he reported back in a lightning talk on the progress made. 

Jones has worked at NVIDIA on window system integration over the last decade or so, which originally meant X11, but now also includes other window systems. There are some existing solutions for memory allocation, but NVIDIA noticed some drawbacks to them when it tried to make them work with its drivers. So the company [proposed EGLStream](<https://lists.freedesktop.org/archives/wayland-devel/2016-March/027547.html>) as a solution, which was "not so well-received so far", but it did help identify the problems that need to be solved. 

That proposed patch added EGLStream to the Weston compositor, but it launched a discussion of [Generic Buffer Management](<https://en.wikipedia.org/wiki/Mesa_\(computer_graphics\)#Generic_Buffer_Management>) (GBM), which Weston already uses for memory allocation, versus EGLStream. Many strong views were expressed in that discussion; there has already been considerable investment in the existing APIs, both by Mesa and Wayland developers as well as by NVIDIA, so it is not surprising that there were differences of opinion. But it was nice to have a civil discussion about the memory allocation issue, he said, and many areas for improvement were identified. The discussion has died down and it was suggested that XDC would be a good venue to make some progress on the issue. 

The problem is how to allocate device-accessible surfaces (memory buffers for various kinds of graphics and video data) from user space. The devices are things like GPUs, scanout engines, and video encoders and decoders. The surfaces allocated are for textures, images, and such; there is a need for some kind of handle for the surfaces that can be securely passed between user-space processes. In addition, a way to manage the surface state (e.g. format, color parameters, compression) and its layout in memory needs to be part of the API. In order to use these buffers in different parts of the system, some kind of synchronization mechanism is required. The latter is not directly related to the allocation problem, but is something that needs to be kept in mind, he said. 

His goal is to get a consensus-based forward-looking API for surface allocation, but he has "no idea" what that API will be, at least yet. It should be agnostic with regard to window systems, kernels, and graphics vendors. So it will be able to be used for window systems like Wayland and others, by old and new Linux kernels, and by other kernels beyond Linux, as long as they are POSIX-like. It would have a "minimal but optimal driver interface" that would still be able to use "100% of the GPU's capabilities". While not directly related to surface allocation, the "final destination", he said, is to have "a completely optimized [scene graph](<https://en.wikipedia.org/wiki/Scene_graph>)" for Weston and other scene-graph compositors. 

#### Prior art

Jones then went into a review of the existing solutions to this problem, with their pros and cons—starting with GBM. At the basic level, GBM has the ability to allocate surfaces and to arbitrate the uses of a surface with a set of flags. It also provides handles to those surfaces. It is incorporated into many code bases at this point, so it is widely deployed and well tested. It has a pretty minimal API and fairly small implementation. 

But GBM does have some shortcomings. The handles are process-local; there are ways to import handles from elsewhere, but not to export them to other processes using the API. It is focused on GPU operations (texturing, rendering, and display), so there is no way to specify that a surface would be used for rendering and passed to a video encoder, for example. Related to that is that the arbitration for the capabilities needed by a surface is done only in the scope of a single device, so you can't use the API to specify surfaces that will be used with multiple devices. 

The Chrome OS Freon project attempted to add surface state management capabilities on top of GBM. There was a lot of discussion between vendors, but no consensus was reached on an optimal design, so something "not ideal" was settled on. The main point of contention was the level of abstraction in describing the transitions between various uses of a surface. 

Android's Gralloc has a similar feature set to GBM. It has support for synchronization using fence file descriptors, but passing handles between processes requires other components from an Android system as there is no direct support for it in Gralloc. It has been widely deployed and is proven in the field. It also has an allocation-time usage specification that has support for non-graphics usage (such as video encoders and decoders). 

Many of the shortcomings of Gralloc are similar to those of GBM as well. There is no explicit surface state management and the arbitration abilities are flag-based. It is open source, but the API is proprietary in some sense, since Google controls it. 

EGLStream was developed to solve the problems he described, so it is not surprising that it provides allocation, arbitration, handles that can be shared by different processes, state management, and synchronization. NVIDIA has been shipping EGLStream for quite some time for a lot of different use cases, he said. It has been ported to all of the different operating systems that the company supports and has a comprehensive feature set. 

While EGLStream is an [open standard](<https://www.khronos.org/registry/egl/extensions/KHR/EGL_KHR_stream.txt>), in practice there is only a single vendor that has implemented it. It does not have cross-device support and it is EGL-based, which may complicate things by bringing OpenGL into the picture. It has been said that EGLStream does too much encapsulation and tries to do too much extra within the API. In addition, its behavior is loosely defined, or even undefined, in some cases. 

The [DMA-BUF allocation mechanism](<https://lwn.net/Articles/474819/>) provides handles to memory allocations that can be shared between drivers; it supports non-graphics devices as well. But it does not have a centralized user-space allocation API, is Linux-only, and lacks any way to describe the content layout. It also only has a limited means to describe the planned usage of the memory at allocation time. 

The Vulkan 3D graphics and compute API is one other thing to consider, Jones said. It provides an allocation mechanism as well as the most detailed allocation-time usage specification that he knows of. It has explicit state management and has a robust synchronization mechanism as well. Vulkan is both extensible and portable, but there is no support at this point for cross-process handles or arbitration. It is also focused only on graphics, compute, and display operations. 

#### Path forward

Based on the prior art and the needs going forward, a set of features needed was identified and generally agreed upon. Whatever the new API is, it should be minimal—anything that is not needed should be eliminated. It should also be portable to multiple platforms and have support for non-graphics devices (e.g. rendering to a video encoder or texturing from a video decoder). It should also use the GPU optimally in the steady state when someone is not moving windows around on the screen; X11 already has this, so anything new should be at least as good. 

To achieve that, he believes there is a need for something like what Vulkan has in terms of an allocation-time usage specification. So when the driver is asked for an allocation, all of the different use cases for the surface can be specified. That will allow the driver to negotiate the surface capabilities based on those use cases. During transitions (such as moving a window or going from a window to full screen), the performance still needs to be good. The idea is to allow multiple uses of the surface without having to do reallocations. 

So, there are various existing APIs and a set of more-or-less consensus goals; what is the path forward? He suggested focusing on solving specific problems that occur with the existing APIs, rather than trying to pick a winner from those APIs. By solving the problems, it will become clear what the API should look like—what it is called at that point is not particularly important. 

Specifically, he suggested that the focus should be on how to create a surface that is cross-driver, cross-engine, and cross-device. Historically, that has been where everything falls apart. If agreement can be reached on that, other simpler cases will just fall out naturally. 

He presented a set of assumptions that he hoped would help simplify the initial discussions. To start with, those working on this problem should assume they are designing an ideal allocation API. That may not actually be the case, but it is a good way to think about it. Thinking in terms of the user-space API first, while keeping both API elegance and the capabilities of the hardware in mind, is also important. 

There needs to be a standard way to describe the capabilities of different devices (for example, devices have different tiling formats, but other drivers won't know anything about some of those formats). It could be similar to the [Khronos data format specification](<https://www.khronos.org/dataformat>) but cover other types of capabilities beyond pixel data formats. 

Capabilities could then be queried from each driver, though the list could become quite large, so some filtering mechanism would be needed. There would also need to be a central authority of some sort to maintain the capability namespace. That could simply be a file in a Git repository or, perhaps, a group like Khronos—it simply needs to be authoritative. The surface allocation layer would collect up and intersect the capabilities of all of the different drivers. 

There is a question of how to filter these capabilities. The API could provide a way to describe the desired usage of the surface, including things like its format, dimensions, and the operations that will be performed using it. The Khronos data format could again be used as a model for how to describe this information. Some types of data have obvious representations (e.g. width/height) and others can be indicated using Boolean flags like those in Gralloc. But there would also be capabilities that are driver-specific, so drivers would have to ignore ones that are targeted at other devices. 

Once the capabilities that are not supported by all of the involved drivers have been eliminated, there needs to be a way to choose the optimal remaining choice. Sorting the remainder depends on the implementation and usage, so it cannot be done by the common framework. His straw-man proposal was to let the application decide once the list has been narrowed down. 

After the surface has been allocated, its chosen properties must also be described. That could perhaps use the same data format as the capability information, but it must be communicated to the requester in some fashion. 

He finished the presentation by noting that all of what had been discussed thus far concerned the image-level capabilities for the requested memory. But there are also some memory-level capabilities that may come into play, notably whether the memory must be physically contiguous. He thought that the image capability concept could be generalized to cover the memory-level requirements as well. Extensibility to allow for tiling layouts or hardware compression of surfaces, for example, would also be important. 

#### Results

In Jones's lightning report of the meetings held on day two, he indicated that some good progress had been made; agreement had been reached on some key points. An allocation request will contain some basic properties like width, height, and format (others will be available via an extension mechanism) along with a list of usage descriptions (e.g. render target, video encoder input). 

The arbitration of the properties is based on intersected sets of supported capabilities along with sets of constraints that get combined together (e.g. a certain stride might constrain the alignment differently between devices). The exact merging of the constraints may not exactly be the union of them, but the merging algorithm will be baked into the library, he said. There will be a set of common capabilities, but some can be vendor-specific; constraint definitions will be shared. 

The capability sets will be reported back to the application, which can serialize them to pass to other processes to allow for incremental refinement. Processes could ask that the list be filtered for specific uses to help winnow down the choices. Once that is done, the sorting is handled by the drivers and the allocation takes place once a single capability set has been chosen. This API will be exposed via a library that has user-space driver/vendor back-ends. 

There are still plenty of things to be resolved, particularly how sorting the capabilities is actually done. There was a lot of discussion how that might be handled, but no conclusion was reached. In addition, the application may need to be able to tell the hardware when the surface is only being used as one of the use cases and when it transitions to one of the others, but how to do that has not been determined. 

How to specify format types is another unresolved piece and they did not discuss the type of handle that would be used for an allocated surface. There is a question whether devices will be enumerable using the API. Also, which kernel interface would be used for allocation has not been resolved. Essentially, Jones said, it has reached a point where folks need to go off and start doing some research and trying things out before further progress can be made. 

For more information, Jones's [PDF slides](<https://www.x.org/wiki/Events/XDC2016/Program/Unix_Device_Memory_Allocation.pdf>) from the talk are available, as is YouTube video of his [talk](<https://youtu.be/4PflCyiULO4?t=7h33m47s>) and [lightning talk report](<https://youtu.be/KIHrjgZJHZA?t=7h13m40s>). His [notes](<http://goo.gl/dv8tHV>) from the meetings are also available. He [posted](<https://lwn.net/Articles/704025/>) an update and pointer to his [GitHub repository](<https://github.com/cubanismo/allocator>) on the dri-devel mailing list on October 4\. 

[I would like to thank the X.Org Foundation for sponsoring my travel to Helsinki for XDC.]
