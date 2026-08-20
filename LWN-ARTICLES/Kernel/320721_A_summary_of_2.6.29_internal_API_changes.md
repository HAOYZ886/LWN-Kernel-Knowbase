---
title: A summary of 2.6.29 internal API changes
url: https://lwn.net/Articles/320721/
date: "March 3, 2009"
category: "Releases-2.6.29"
author: "By Jonathan Corbet March 3, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 3, 2009

As the 2.6.29 kernel development cycle draws toward its eventual close, it is appropriate to look back at the internal API changes which have been made. The following list cannot possibly be exhaustive, but, hopefully, it captures the major points. 

  * The massive [task credentials patch set](<http://lwn.net/Articles/251469/>) has been merged. This code reorganizes the handling of process credentials (user ID, capabilities, etc.). One of the immediate implications of this change is direct references to credential-oriented fields in the task structure need to be changed; for example, `current->user->uid` becomes `current_uid()`. See [Documentation/credentials.txt](<https://lwn.net/Articles/313616/>) for a description of the new API. 

  * The ftrace code has seen a lot of internal changes. The function tracing feature has seen a number of improvements, and the developers have added mechanisms to profile the behavior of `if` statements, provide function call graphs, obtain user-space stack traces, and follow CPU power-state transitions. 

  * Most of the callback functions/methods associated with the `net_device` structure have been moved out of that structure and into the new `struct net_device_ops`. In-tree drivers have been converted to the new API. 

  * The `priv` field has been removed from `struct net_device`; drivers should use `netdev_priv()` instead. 

  * The generic PHY layer now has power management support. To that end, two new methods - `suspend()` and `resume()` \- have been added to `struct phy_driver`. 

  * The networking layer now supports [large receive offload](<http://lwn.net/Articles/243949/>) (or "generic receive offload") operation. 

  * The NAPI API has been cleaned up somewhat; in particular, functions like `netif_rx_schedule()`, `netif_rx_schedule_prep()`, and `netif_rx_complete()` have lost the unneeded `struct net_device` parameter. 

  * The `poll()` file operation is now allowed to sleep; see [this article](<http://lwn.net/Articles/308426/>) for more information on this change. 

  * The CPU mask mechanism, used to represent sets of processors in the system, is in the middle of being massively reworked. The problem is that CPU masks were often put on the stack, but, as the number of processors grows, the stack lacks room for the mask. The new API is designed to get these masks off the stack, and to guard against anybody ever trying to put one back. See [this posting by Rusty Russell](<http://ozlabs.org/~rusty/index.cgi/2009/01/07#2009-01-07>) for details on this work. 

  * An [infrastructure for asynchronous function calls](<http://lwn.net/Articles/314808/>) has been merged. This code is still a work in progress, though, and, for 2.6.29, it will not be activated in the absence of the `fastboot` command-line parameter. 

  * The [exclusive I/O memory allocation functions](<http://lwn.net/Articles/308426/>) have been merged. 

  * There is a new synchronous hash interface called "shash." It simplifies the use of synchronous hash operations while allowing the same tfm to be used simultaneously in different threads. All in-tree users have been switched to the new API. 

  * The hrtimer code has been simplified with the removal of variable modes for callback functions. All processing is now done in hardirq context. 

  * A new set of LSM hooks has been added; these support pathname-based security operations. With the merging of these hooks, one major obstacle to the inclusion of security modules like AppArmor and TOMOYO has been removed. 

  * The kernel will now refuse to build with GCC 4.1.0 or 4.1.1; those versions have unfortunate bugs which prevent the building of a working kernel. Versions 3.0 and 3.1 have also been deemed to be too old and will not be supported in 2.6.29. 

  * Video4Linux drivers now use a separate `v4l2_file_operations` structure to hold their VFS-like callbacks. The prototypes of a number of these functions have been changed to remove the `inode` argument. 

  * Video4Linux2 has also acquired a new "subdevice" concept, meant to reflect the fact that video "devices" tend to be, in reality, a set of cooperating devices. See [the new document](<https://lwn.net/Articles/313784/>) for a description of how this mechanism works. 

  * Two new functions - `stop_machine_create()` and `stop_machine_destroy()` \- allow the independent creation of the threads used by `stop_machine()`. That, in turn, lets those threads be created before trying to actually stop the machine, making that operation more resistant to failure. 

  * The exports for a number of SUNRPC functions have been changed to GPL-only. 

  * The internal MTD (memory technology device) API has seen significant changes aimed at supporting larger devices (those requiring 64-bit sizes). 

Developers interested in the history of kernel API changes can look at [the LWN 2.6 API changes page](<https://lwn.net/Articles/2.6-kernel-api/>). After a period of unfortunate neglect, this page has been made current once again; your editor promises to be a bit more diligent about maintaining this page in the future.
