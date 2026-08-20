---
title: A sys_poll() ABI tweak
url: https://lwn.net/Articles/483078/
date: "February 22, 2012"
category: "Development model-User-space ABI"
author: "By Jonathan Corbet February 22, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 22, 2012

The `poll()` system call has three parameters, one of which is a timeout value specifying an upper bound (in milliseconds) for how long the process will wait. The manual page indicates that the type of this value is `int`. For reasons lost in history, though, the kernel's internal implementation of `poll()` has always expected the timeout value to be a `long` integer. And that has created a source of occasional bugs. 

Most of the time, things just work. The `int` and `long` types tend to be the same on most architectures, and, in cases where they are different, glibc sign-extends the timeout value appropriately. Things go wrong, though, when a 32-bit process is running on an x86-64 system. In that case, the 32-bit `sys_poll()` function just passes the timeout value directly to the native kernel version, without sign extension. So if the timeout value is negative (an indication that `poll()` should wait forever if need be), the kernel will eventually see a large, positive timeout instead. 

There are various ways this problem could be fixed. What Linus has chosen to do, though, is to just change the type of the timeout parameter to `int` inside the kernel. Since the timeout is now a 32-bit quantity on all systems, that particular source of confusion is removed. There is a small risk to this approach, though: it is possible that some program somewhere was actually making use of 64-bit timeouts. Doing so would require replacing or bypassing glibc (because its sign extension makes 64-bit timeouts unusable), so it's unlikely that anybody has bothered, but one never knows. If this change were to break a real application, it would have to be reverted in favor of a more complicated solution. 

Linus's [patch](<https://lwn.net/Articles/483083/>) was merged for 3.3-rc5, so anybody who objects has a few weeks to make their concerns known.
