---
title: Firmware loading and suspend/resume
url: https://lwn.net/Articles/511105/
date: "August 15, 2012"
category: "Device drivers-Firmware loading; Firmware"
author: "By Jonathan Corbet August 15, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 15, 2012

Many devices are unable to function until the host system has loaded them with their operating firmware. Runtime-loadable firmware has some real advantages: the hardware can be a little cheaper to make, and the firmware is easily upgraded after the hardware has been sold. But it also poses some problems, especially when combined with other features. Properly handling firmware loading over suspend/resume cycles has been a challenge for the kernel for some time, but a new set of patches may be poised to make things work better with little or no need for changes to drivers. 

The obvious issue with suspend/resume is that any given device may lose its firmware while the system is suspended. The whole point of suspending the system is to reduce its power consumption to a minimum, so that operation may well power down peripheral devices entirely. Loss of firmware during suspend doesn't seem like it should be a big problem; the driver can just load the firmware again at resume time. But firmware tends to live on disk, and the actual firmware loading operation involves the running of a helper process in user space. Neither the disk nor user space are guaranteed to be available at the point in the resume process when a given device wants its firmware back; drivers that attempt to obtain firmware at such times may fail badly. The result is resume failures; they may be of the intermittent, developer-never-sees-it variety that can be so frustrating to track down. So the search has been on for a more robust solution for some time. 

In July, Ming Lei tried to address this problem with [a patch](<https://lwn.net/Articles/511114/>) integrating firmware loading with the deferred driver probing mechanism. In short, if a firmware load fails, the whole driver initialization process would be put on the deferred queue to be retried later on. So, a driver that is unable to load its firmware at resume time will be put on hold and retried at a later point when, hopefully, the resources required to complete the firmware load will be available. That, Ming hoped, would resolve a lot of resume-time failures without requiring changes to lots of drivers. 

Linus, however, [disagreed](<https://lwn.net/Articles/511115/>): 

Sure, for a lot of devices it's fine to load the firmware later. But some devices may be part of the resume sequence in very critical ways, and deferring the firmware loading will just mean that the resume will fail. 

Deferring firmware loading in this manner, he thought, would just serve to hide problems from developers but leave them to burn users later on. It is much better, he thought, to force driver writers to deal with the problem explicitly. 

The classic way for a driver writer to handle this problem is to just keep the firmware around after it is loaded at system boot time. Permanently cached firmware will always be available when it is needed, so firmware loading at resume time should be robust. The problem with that approach is that the firmware blobs loaded into some devices can be quite large; keeping them around forever can waste a fair amount of kernel-space memory. To make things worse, these blobs are loaded into `vmalloc()` memory (so that they appear to be contiguous in memory); that memory can be in short supply on 32-bit systems. Permanently caching the firmware is, thus, not an ideal solution, but that is what a number of drivers do now. 

After the discussion with Linus, Ming thought for a while and came back with [a new proposal](<https://lwn.net/Articles/509827/>): cache firmware blobs, but only during the actual suspend/resume cycle. Drivers can, of course, do that now; they can request a copy of the firmware while suspending their devices, and release that copy once it's no longer needed at resume time. But that is a chunk of boilerplate code that would need to be added to each driver. Ming's patch, instead, makes this process automatic and transparent. 

In particular, `request_firmware()` is changed to make a note of the name of every firmware blob it is asked to load. This information is reference-counted and tied to the devices that needed the firmware; it can thus be discarded if all such devices disappear. The result is a simple data structure tracking all of the firmware blobs that may be needed by the hardware currently present in the system. 

At system suspend time, the code simply goes and loads every piece of firmware that it thinks may be needed. That data then sits in memory while the system is suspended. At resume time, those cached blobs are available to any driver, with no need for filesystem access or user-space involvement, via the usual `request_firmware()` interface. Once the resume process is complete, the firmware loader will, after a small delay, release all of those cached firmware images, freeing the associated memory and address space for other uses. 

The patch seems close to an ideal solution. Firmware loading at resume time becomes more robust, there is no need for drivers to be concerned with how it works, and wasted memory is minimized. Even Linus [said](<https://lwn.net/Articles/511120/>) ""Nothing in this patchset made me go 'Eww'"", which, from him, can be seen as reasonably high praise. It doesn't solve every problem; there are, for example, [some strange devices](<http://marc.info/?l=linux-usb&m=132554118928398&w=2>) that retain firmware over a reboot but not over suspend, so the system may not know that a specific firmware image is needed until resume time, when it's too late. But such hardware is probably best handled as a special case. For the rest, we may be close to a solution that simply works—and that brings an end to the recurring "firmware at resume time" discussions on the mailing lists.
