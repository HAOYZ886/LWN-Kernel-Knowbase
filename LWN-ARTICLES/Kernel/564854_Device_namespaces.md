---
title: Device namespaces
url: https://lwn.net/Articles/564854/
date: "August 28, 2013"
category: "Namespaces-Device namespaces"
author: "By Jake Edge August 28, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
August 28, 2013

Lightweight virtualization using containers is a technique that has finally come together for Linux, though there are still some rough edges that may need filing down. Containers are created by using two separate kernel features: control groups (cgroups) and namespaces. Cgroups are in the process of [being revamped](<https://lwn.net/Articles/484251/>), while there may still need to be more namespaces added to the [varieties currently available](<https://lwn.net/Articles/531114/>). For example, there is no way to separate most devices into their own namespace. That's a hole that Oren Laadan would like to see filled, so he put out an [RFC on device namespaces](<https://lwn.net/Articles/564977/>) recently. 

Namespaces partition global system resources so that different sets of processes have their own view of those resources. For example, mount namespaces partition the mounted filesystems into different views, with the result that processes in one namespace are unable to see or interact with filesystems that are only mounted in another. Similarly, [PID namespaces](<https://lwn.net/Articles/531419/>) give each namespace its own set of process IDs (PIDs). Certain devices are currently handled by their own namespace or similar functionality: network namespaces for network devices and the `devpts` pseudo-filesystem for pseudo-terminals (i.e. pty). But there is no way to partition the view of all devices in the system, which is what [device namespaces](<https://github.com/Cellrox/devns-patches/wiki/DeviceNamespace>) would do. 

The motivation for the feature is to allow multiple virtual phones on a single physical phone. For example, one could have two complete Android systems running on the phone, with one slated for work purposes, and the other for personal uses. Each system would run in its own container that would be isolated from the other. That isolation would allow companies to control the apps and security of the "company half" of a phone, while allowing the user to keep their personal information separate. A [video](<http://vimeo.com/60113683>) gives an overview of the idea. Much of that separation can be done today, but there is a missing piece: virtualizing the devices (e.g. frame buffer, touchscreen, buttons). 

The proposal adds the concept of an "active" device namespace, which is the one that the user is currently interacting with. The upshot is that a user could switch between the phone personalities (or personas) as easily as they switch between apps today. Each personality would have access to all of the capabilities of the phone while it was the active namespace, but none while it was the inactive (or background) namespace. 

Setting up a device namespace is done in the [normal way](<https://lwn.net/Articles/531381/>), using the `clone()`, `setns()`, or `unshare()` system calls. One surprise is that there is no new `CLONE_*` flag added for device namespaces, and the `CLONE_NEWPID` flag is overloaded. A comment in the 
```
(<https://github.com/Cellrox/devns-patches/blob/master/dev_ns__introduce_device_namespaces#L419>) explains why: 
[code]
        /*
         * Couple device namespace semantics with pid-namespace.
         * It's convenient, and we ran out of clone flags anyway.
         */
```

While coupling PID and device namespaces may work, it does seem like some kind of long-term solution to the clone flag problem is required. Once a process has been put into a device namespace, any `open()` of a namespace-aware device will restrict that device to the namespace. 

At some level, adding device namespaces is simply a matter of virtualizing the major and minor device numbers so that each namespace has its own set of them. The major/minor numbers in a namespace would correspond to the driver loaded for that namespace. Drivers that might be available to multiple namespaces would need to be changed to be namespace-aware. For some kinds of drivers, for example those without any real state (e.g. for Android, the [LED subsystem](<https://github.com/Cellrox/devns-patches/blob/master/dev_ns__adopt_the_led_subsystem>) or the [backlight/LCD subsystem](<https://github.com/Cellrox/devns-patches/blob/master/dev_ns__adopt_the_backlight-lcd_subsystem>)), the changes would be minimal—essentially just a test. If the namespace that contains the device is the active one, proceed, otherwise, ignore any requested changes. 

Devices, though, are sometimes stateful. One can't suddenly switch sending frame buffer data mid-stream (or mix two streams) and expect the screen contents to stay coherent. So, drivers and subsystems will need to handle the switching behavior. For example, the framebuffer device should only reflect changes to the screen from the active namespace, but it should buffer changes from the background namespace so that those changes will be reflected in the display after a switch. 

Laadan and his colleagues at [Cellrox](<http://www.cellrox.com/>) have put together a [set of patches](<https://github.com/Cellrox/devns-patches/wiki/DeviceNamespace%3APatches>) based on the 3.4 kernel for the Android emulator (goldfish). There is also a fairly detailed description of the patches and the changes made for both stateless and stateful devices. An [Android-based demo](<https://github.com/Cellrox/devns-demo>) that switches between a running phone and an app that displays a changing color palette has also been created. 

So far, there hasn't been much in the way of discussion of the idea on the containers and lxc-devel mailing lists that the RFC was posted to. On one hand, it makes sense to be able to virtualize all of the devices in a system, but on the other that means there are a _lot_ of drivers that might need to change. There may be some "routing" issues to resolve, as well—when the phone rings, which namespace handles it? The existing proof-of-concept API for switching the active namespace would also likely need some work. 

While it may be a worthwhile feature, it could also lead to a large ripple effect of driver changes. How device namespaces fare in terms of moving toward the mainline may well hinge on others stepping forward with additional use cases. In the end, though, the [core changes](<https://github.com/Cellrox/devns-patches/blob/master/dev_ns__introduce_device_namespaces>) to support the feature are fairly small, so the phone personality use case might be enough all on its own.
