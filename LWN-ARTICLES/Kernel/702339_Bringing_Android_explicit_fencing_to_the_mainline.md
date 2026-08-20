---
title: Bringing Android explicit fencing to the mainline
url: https://lwn.net/Articles/702339/
date: "October 5, 2016"
category: Android
author: "By Jake Edge October 5, 2016 X.Org Developers Conference"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
October 5, 2016

* * *

[X.Org Developers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-X.Org_Developers_Conference>)

Synchronization between graphics drivers and the underlying GPU hardware is typically done using a [fence](<https://en.wikipedia.org/wiki/Memory_barrier>) (i.e. a kind of memory barrier). The fence signals when a buffer is no longer being used by one component so it can be operated on or reused by another. Gustavo Padovan spoke at the [2016 X.Org Developers Conference](<https://www.x.org/wiki/Events/XDC2016/>) (XDC) on efforts to add an [explicit fencing mechanism to the mainline](<https://lwn.net/Articles/685049/>). 

Padovan works for Collabora, which is a consulting company that helps its clients bring their code to upstream communities, mostly in the areas of graphics and multimedia. He has been working on the kernel for the last seven years or so. Over the last two years, he has mostly focused on the graphics subsystem in the kernel. 

[ ![\[Gustavo Padovan\]](https://static.lwn.net/images/2016/xdc-padovan-sm.jpg) ](<https://lwn.net/Articles/702565/>)

The idea behind fencing is to ensure ordering between operations—in particular to synchronize buffer sharing between the GPU and display drivers. It will make sure that the GPU does not write to a buffer that is still being displayed and that buffers won't be displayed while the GPU is still rendering to them. It allows the display driver and the GPU to operate asynchronously. 

The current kernel has support for implicit fencing, which means that user space has no knowledge of the fences, nor can it use or interact with them. That allows for simple compositors, but there are situations where the compositor might freeze the whole desktop. Padovan's example was a buffer ("C") that is composed of two other buffers ("A" and "B"). A and B are rendered in parallel and the compositor only gets notified when both are complete. If A is rendered quickly, but B is slow, C is blocked waiting for both to complete and the entire desktop can freeze. 

Explicit fencing, where user space can use fences to control the synchronization, can help with that problem. It allows the compositor to make its own decisions. It could, for example, composite C using the new A and the old B, which will avoid blocking the whole desktop waiting on B, he said. 

With explicit fencing, user space would get fences from both the Direct Rendering Manager (DRM) subsystem and from the GPU. It can wait on those fences to ensure that rendering and display operations have completed. It also brings other advantages: explicit fencing is better for tracing and debugging and is also needed for the new [Vulkan 3D API](<https://en.wikipedia.org/wiki/Vulkan_\(API\)>). 

#### Android sync framework

The Android [synchronization framework](<https://source.android.com/devices/graphics/index.html#synchronization_framework>) is an implementation of explicit fencing. It uses file descriptors to pass fences to user space, he said. It has three main elements: a sync timeline to control ordering, a sync point to represent a fence, and a sync fence for passing file descriptors. 

The sync timeline is a monotonically increasing counter; there is normally one timeline per driver context. A sync point is the fence itself and represents a value on the timeline. Sync points have three states: active, signaled, and error. The sync fence object wraps a sync point into a file, from which a file descriptor can be created. Those are shared via user-space file-descriptor passing. Sync fences have an active and signaled state. In addition, two sync fences can be merged into a new sync fence with its own file descriptor to allow waiting for multiple sync points. 

The user-space interface to the framework is based on `ioctl()` commands, but the libsync library provides three helper functions: `sync_wait()` to wait on a file descriptor with a timeout, `sync_merge()` to merge two sync fence file descriptors into a new one, and `sync_fence_info()` to retrieve information about the sync fence and the sync points inside that fence. 

#### Mainline fencing

The mainline effort started by modifying the Android sync framework. That resulted in the fence synchronization mechanism that Maarten Lankhorst got merged two or three years ago, Padovan said. It is a generic way to do buffer synchronization between drivers, but only inside the kernel. It uses a `struct fence` that has a `context` field that carries the same information as the sync timeline. There are several calls that can be used to operate on those fences within the kernel: `fence_signal()`, `fence_wait()`, and `fence_add_callback()`. 

The Android sync framework was added to the staging tree in 2013 and is now in the "de-staging" process. Most of it is not needed any more, he said. The sync timeline and sync point parts are handled by the fencing mechanism in the mainline. The missing piece is a sync fence that can be accessed from user space. 

To that end, sync fence has been renamed to sync file and the `ioctl()` API has been changed. There were no mainline users of sync fences and a patch has been provided for Android libsync to use the new interface. Most of the internal kernel API has been removed but, for file descriptor passing, the kernel can use `sync_file_create()` to get a sync file from a fence and the inverse, `sync_file_get_fence()`, to get a fence from a file descriptor. 

There is also a new `struct fence_array` that is a subclass of `struct fence`, Padovan said. As the name would imply, it is meant to hold multiple fences that come from merging sync files. It hides that complexity from the drivers, though the `fence_is_array()` call can be used to determine if it is a `fence_array`; there is a way to extract individual fences from the array as well. 

#### Driver support

So that covered the infrastructure needed in the kernel to support explicit fencing; once that was done, driver support was next up, both for DRM/kernel mode setting (KMS) drivers as well as for the GPU drivers. Explicit fencing is only supported for [atomic mode setting](<https://lwn.net/Articles/653071/>) drivers; the legacy API will not be supported. The work was done entirely in the DRM core, so KMS drivers do not need to do anything to support explicit fencing. The atomic mode setting `ioctl()` was extended to handle sending fences to the kernel and receiving them from the kernel. The fences that are sent from user space are called "in-fences", while those that are sent to user space are "out-fences". 

For in-fences, there is a `FENCE_FD` property associated with each DRM plane. It carries the sync file descriptors that correspond to the fences. The `drm_atomic_helper_wait_for_fences()` helper can be used to wait for all fences on all planes. There is one out-fence per DRM CRTC (which stands for "cathode ray tube controller", but that is historical; it represents a display pipeline these days). That fence will get signaled when the CRTC scanout occurs and means that the previous buffer can be reused; it is like the page-flip signal, he said. User space has to request an out-fence using the `DRM_MODE_ATOMIC_OUT_FENCE` flag, though a helper function for libdrm has been proposed. 

On the DRM rendering side, things are done similarly to what is done for KMS, Padovan said. The difference is that every GPU driver needs to change to support sync file and fences. The execbuffer `ioctl()` commands needed to be extended for that support. There is work in progress on the freedreno, i915, and virgl drivers. 

Mesa will support the same extensions as Android, he said. There are interfaces to create a fence file descriptor and to make the GPU wait for a specific fence to signal. Rob Clark is working on that, though [kmscube](<https://github.com/robclark/kmscube>) is currently working with explicit fencing, which serves as a proof of concept that everything works. 

The drm_hwcomposer2 (or HWC2) component was recently released by Google. It supports the semantics of DRM fences and can be used as an example user-space implementation that is needed in order to get the explicit fencing work into the mainline kernel. There were some initial patches to drm_hwcomposer by Sean Paul to support the fencing work, which have been picked up and extended by Robert Foss. 

Padovan concluded his talk with an overview of the current status. Some of the early pieces, such as de-staging sync files and adding the `fence_array` support, has been done. The DRM pieces are works in progress with a possible target of the 4.10 kernel. The Mesa and drm_hwcomposer2 support are also currently being worked on. Further out, Wayland support and adding explicit fencing for Video4Linux2 are both planned. 

Audience members asked about the reliance on file descriptors. There was concern that for a single process, file descriptors could become a scarce resource and that Vulkan might need a way to use fences that doesn't burn file descriptors. Padovan acknowledged that concern and suggested that it was something that could be looked at down the road. Those looking for more details may want to consult the [YouTube video](<https://www.youtube.com/watch?v=KIHrjgZJHZA&t=7590s>) and the [PDF slides](<https://www.x.org/wiki/Events/XDC2016/Program/XDC-Explicit-Fencing_Talk.pdf>) from the talk. 

[I would like to thank the X.Org Foundation for sponsoring my travel to Helsinki for XDC.]
