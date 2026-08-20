---
title: Running Android on a mainline graphics stack
url: https://lwn.net/Articles/733463/
date: "September 12, 2017"
category: Android
author: "By Jonathan Corbet September 12, 2017 Open Source Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 12, 2017

* * *

[Open Source Summit](<https://lwn.net/Archives/ConferenceByYear/#2017-Open_Source_Summit_North_America>)

The Android system may be based on the Linux kernel, but its developers have famously gone their own way for many other parts of the system. That includes the graphics subsystem, which avoids user-space components like X or Wayland and has special (often binary-only) kernel drivers as well. But that picture may be about to change. As Robert Foss described in his Open Source Summit North America presentation, running Android on the mainline graphics subsystem is becoming possible and brings a number of potential benefits. 

He started the talk by addressing the question of why one might want to use mainline graphics with Android. The core of the answer was simple enough: we use open-source software because it's better, and running mainline graphics takes us toward a fully open system. With mainline graphics, there are no proprietary blobs to deal with. That, in turn, makes it easy to run current versions of the kernel and higher-level graphics software like [Mesa](<https://www.mesa3d.org/>). 

Getting the security fixes found in current kernels is worth a lot in its own right, but up-to-date kernels also bring new features, lots of bug fixes, better performance, and reduced power usage. The performance [![\[Robert Foss\]](https://static.lwn.net/images/conf/2017/ossna-lpc/RobertFoss-sm.jpg)](<https://lwn.net/Articles/733467/>) and power-consumption figures for most hardware tends to improve for years after its initial release as developers find ways to further optimize the software. Running a fully free system increases the possibilities for long-term support. Many devices have a ten-year (or longer) life span; if they are running free software, they can be supported by anybody. That is, Foss said, one of the main reasons why the GPU vendors tend _not_ to open-source their drivers. Using mainline graphics also makes it possible to support multiple vendors with a single stack, and to switch vendors at will. 

At the bottom of the Android graphics stack is the kernel, of course; but the layer above that tends to be a proprietary vendor driver. That driver, like most GPU drivers, has a substantial user-space component. Android's display manager is SurfaceFlinger; it takes graphical objects from the various apps and composes them onto the screen. The interface between SurfaceFlinger and the driver is called [HWC2](<https://source.android.com/devices/graphics/implement-hwc>); it is implemented by the user-space component of the vendor driver. Among other things, HWC2 implements common interfaces like OpenGL and Vulkan. 

The HWC2 interface is also responsible for composing objects into the final display and implementing the abstractions describing those objects. When possible, it will offload work from the GPU to a hardware-based compositor. In the end, he said, GPUs are not particularly good at composing, so offloading that work can speed it up and save power. HWC2 is found in ChromeOS as well as in Android. 

To create an open-source stack, one clearly has to replace the proprietary vendor drivers. That means providing a driver for the GPU itself and an implementation of the HWC2 API. The latter can be found in the [drm_hwc](<https://www.chromium.org/android/drm_hwcomposer-overview>) (or drm_hwcomposer) project, which was originally written at Google but which has since escaped into the wider community. It is sometimes used on Android systems now, Foss said, especially in embedded settings. The manufacturers of embedded devices are finding that their long-term support needs are well met with open-source drivers. 

So a free Android stack is built around drm_hwc. It also includes components like Mesa and [libdrm](<https://01.org/linuxgraphics/community/libdrm>), and it's all based on the kernel's direct rendering manager (DRM) layer. Finally, there is a component called [gbm_gralloc](<https://github.com/robherring/gbm_gralloc>), which handles memory allocations and associates properties (which color format is in use, for example) with video buffers. 

So what is the status of this work? There are a couple of important kernel components that were prerequisites to this support; one of those is [buffer synchronization](<https://lwn.net/Articles/702339/>), which has recently been merged. This feature allows multiple drivers to collaborate around shared buffers; it was inspired by a similar feature in the Android kernel. Some GPU drivers now have support for synchronization. The other important piece was the [atomic display API](<https://lwn.net/Articles/653071/>); it's the only API that supports synchronization. Most drivers have support for this API at this point, which is good, since HWC2 requires it. 

There are a few systems where all of this works now. The i.MX6 processor with the Vivante gc3000 GPU has complete open-source support; versions with older GPUs are not yet supported at the same level. There is support for the DragonBoard 410c with the Adreno GPU. The MinnowBoard Turbot has an Intel HD GPU which has "excellent open-source software support". Finally, the HiKey 960 is a new high-end platform; it's not supported yet but that support is "in the works". 

Foss concluded by saying that support for Android on the mainline graphics stack is now a reality for a growing number of platforms. The platforms he named are development boards and such, though, so your editor took the opportunity to ask if there was any prospect for handsets with mainline graphics support in the future. Foss answered that there are "rumors" that Google likes this work and is keeping an eye on it. Time will tell whether those rumors turn into mainstream Android devices that can run current mainline kernels with blob-free graphics support. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting your editor's travel to the Open Source Summit.]
