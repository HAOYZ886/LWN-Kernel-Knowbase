---
title: Graphics world domination may be closer than it appears
url: https://lwn.net/Articles/703574/
date: "October 18, 2016"
category: "Device drivers-Graphics"
author: "By Jonathan Corbet October 18, 2016 Kernel
Recipes"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 18, 2016

* * *

[Kernel Recipes](<https://lwn.net/Archives/ConferenceByYear/#2016-Kernel_Recipes>)

The mainline kernel has support for a wide range of hardware. One place where support has traditionally been lacking, though, is graphics adapters. As a result, a great many people are still using proprietary, out-of-tree GPU drivers. Daniel Vetter went before the crowd at [Kernel Recipes 2016](<https://kernel-recipes.org/en/2016/>) to say that the situation is not as bad as some think; indeed, he said, in this area as well as others, world domination is proceeding according to plan. 

#### The current state of affairs

The first stop on Vetter's tour of the direct rendering manager (DRM) subsystem was documentation, and, in particular, the [transition to Sphinx](<https://lwn.net/Articles/692704/>) that has unfolded over the last couple of release cycles. The new formatted documentation system for the kernel is "pretty and awesome", and makes writing the documentation fun. As a result, there's now a lot more documentation than there used to be; indeed, the [DRM documentation](<https://dri.freedesktop.org/docs/drm/gpu/index.html>) is pretty much complete. The biggest gap at this point is a top-level picture that nicely ties all the pieces together. 

Moving on to rather older work (he titled this section "dungeons and dragons"), Vetter noted that there are still some DRM1 drivers around; these are at least ten years old at this point. They feature nasty user-space APIs, root holes, and other delightful things. These drivers are built around a midlayer architecture, a design which has gone out of fashion in recent years; the idea was to make it possible to build the drivers on BSD systems. In current kernels, these drivers are hidden behind the `CONFIG_DRM_LEGACY` option. They cannot be removed outright without breaking things, though, so they will remain for a while. 

The [IGT tools from Intel](<https://01.org/linuxgraphics/gfx-docs/igt/>) have proved to be a useful test suite for the validation of DRM drivers. They are Intel-specific for now, but are being modified to be more generic. At this point, a number of drivers and continuous-integration systems are using these tests to trap regressions. See [the DRM documentation](<https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html#validating-changes-with-igt>) for information on how to validate drivers with the IGT suite. 

Recently there has been an influx of DRM developers from the ARM community; that has led to a new set of problems. The DRM subsystem is special, Vetter said, in that it [requires](<https://dri.freedesktop.org/docs/drm/gpu/drm-uapi.html#open-source-userspace-requirements>) that the user-space API for any driver be open source. Much of the code for these drivers runs in user space; the 10% that runs in the kernel is "useless" without the user-space side as well. A kernel driver without the user-space code cannot be enhanced or maintained. [![\[Daniel Vetter\]](https://static.lwn.net/images/conf/2016/recipes/DanielVetter2-sm.jpg)](<https://lwn.net/Articles/703612/>) The ARM folks were unaware of this restriction and not used to operating in this mode, so the DRM maintainers have had to start rejecting their patches. The result was some screaming, but, at this point, the ARM community understands the requirements and is starting to look at opening up the user-space code as well. 

One of the big changes in the DRM subsystem in recent years has been the switch to the [atomic mode-setting API](<https://lwn.net/Articles/653071/>). The original DRM API featured one `ioctl()` call for each operation to be done; that resulted in a lot of display flickering as applications worked through a long series of changes. The atomic API allows everything to be done with a single call, leading to flicker-free changes. An atomic change is an all-or-nothing affair; if it succeeds at all, it will succeed completely. 

This API also provides a separate call to check whether a set of changes would succeed without actually making those changes. It can be hard to know before trying; hardware often has weird restrictions that get in the way. He mentioned adapters with three video outputs but only two clocks as an example. Overlay support (the ability to directly display a video stream from another source, such as a camera, without going through user space) has been added to this API as well. Overlays went out of fashion for a while, but it turns out that a lot of power can be saved by outputting the video directly; it is a crucial feature for mobile systems. 

At this point, there are 20 drivers in the mainline with atomic mode-setting implementations; another two or three are added with each release. The adoption of this API far exceeds the rate of adoption of the original kernel mode-setting API. It helps that a lot of functionality is in common code now, so the drivers themselves have gotten smaller. The support library has been made more modular; using it is not an all-or-nothing affair like it used to be. 

Use of the atomic API is growing; one example is the [drm_hwcomposer](<https://dev.chromium.org/android/contributing-to-drm_hwcomposer>) library, written by Google for use with Android systems. The ChromeOS [Ozone](<https://www.chromium.org/developers/design-documents/ozone>) interface running on Wayland uses it, as do all the other Wayland implementations. We have, he said, "a driver API to rule them all" for the first time. 

#### Looking forward

Turning to future work, Vetter mentioned that there is interest in an interface that can allocate buffers for use with multiple devices. The [ION memory allocator](<https://lwn.net/Articles/480055/>) offers this functionality, but it remains Android-specific for now. 

The old framebuffer device (fbdev) interface has been deprecated for some time, but it still turns out to be useful in some settings. In particular, it can save memory bandwidth and power on some low-end displays — those that require manual uploading of display data. The generic fbdev "defio" interface can now be remapped onto kernel mode-setting operations, making it possible to write a full fbdev driver on top of the DRM subsystem. 

The [simple display pipeline helper](<https://01.org/linuxgraphics/gfx-docs/drm/gpu/drm-kms-helpers.html>) also makes writing simple drivers easy. For settings where there is a simple processing pipeline and a single connector, it can provide access to the atomic API without most of the complexity. With this helper, the DRM API is "now strictly better" than fbdev. 

Fences are currently an area of active development. A fence is like the kernel's `completion` structure, in that it can be used to wait for (and signal) the completion of an operation; it is intended to be used with DMA operations in particular. There are two models for fence usage. In the "implicit" model, the kernel attaches fences to I/O buffers and takes care of everything; user space never sees it. The "explicit" model, instead, has the kernel providing fences to user space, which must then manage them itself. 

The implicit model has been implemented for some time, in the form of `reservation_object` structures attached to DMA buffers. The TTM memory manager (used with the AMD and Nouveau drivers) has always supported it; other drivers are picking up support over time. This is the model preferred by the Linux desktop; both X and Wayland expect implicit fencing. 

On the other hand, the Android system wants to use explicit fencing. It provides more control to user space and reduces the need for complexity in (vendor-supplied) graphics drivers. That was the driving factor in Android's decision, Vetter said; no vendor proved able to implement implicit fences correctly. The DRM subsystem [implements an explicit fence](<https://lwn.net/Articles/702339/>) as a `sync_file` structure, which is returned to user space as a file descriptor. User-space fences will be supported in the 4.9 kernel; the MSM/freedreno driver has added support so far. 

As one might imagine, there is some tricky interaction between implicit and explicit fences. The solution that has been chosen is to use implicit fences by default, but to switch to the explicit model as soon as an application calls one of the explicit-fencing extensions. 

Google has created the "HWC2" composer that can make use of DRM's explicit-fencing support; it is not yet publicly released, Vetter said, but will hopefully show up in 4.10. More information will be available at the Linux Plumbers Conference. Sometime soon it will be possible to run Android on a mainline kernel with an open-source graphics stack, he said. 

Along those lines, what is the status of low-level GPU drivers? At this point, there are three vendor-supported open drivers in the mainline, and three more reverse-engineered ones. Of those, the Nouveau driver runs fairly well on Tegra systems. The freedreno driver is "pretty feature-complete" and is now competitive with proprietary drivers. The [etnaviv](<https://github.com/etnaviv>) driver is coming along, but still needs work on the user-space side. But, he said, there are still no vendor-supported system-on-chip drivers; that situation is "pretty dire." 

He finished up by noting that the atomic API now "rules them all." There has been a lot of progress in documentation and general cleanup; all of the major gaps for authors of display drivers have been closed. Cross-driver fencing is reaching a point of being ready for everyone, and even rendering is showing some (albeit slow) progress. Upstream graphics, he said, is finally winning.
