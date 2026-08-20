---
title: The guaranteed contiguous memory allocator
url: https://lwn.net/Articles/1015000/
date: "March 21, 2025"
category: "Contiguous memory allocator; Memory management-Large allocations"
author: "By Jonathan Corbet March 21, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 21, 2025

As a system runs and its memory becomes fragmented, allocating large, physically contiguous regions of memory becomes increasingly difficult. Much effort over the years has gone into avoiding the need to make such allocations whenever possible, but there are times when they simply cannot be avoided. The kernel's [contiguous memory allocator](<https://lwn.net/Articles/486301/>) (CMA) subsystem attempts to make such allocations possible, but it has never been a perfect solution. Suren Baghdasaryan is is trying to improve that situation with the [guaranteed contiguous memory allocator patch set](<https://lwn.net/ml/all/20250320173931.1583800-1-surenb@google.com>), which includes work from Minchan Kim as well. 

In the distant past, Dan Magenheimer introduced the concept of [transcendent memory](<https://lwn.net/Articles/340080/>) — memory that is not directly addressable, but which can be used opportunistically by the kernel for caching or other purposes. Most of the transcendent-memory work has since gone unused and been removed from the kernel, but the idea persists, and this patch series makes use of it to provide guaranteed CMA. 

Specifically, the patch set includes a subsystem called "cleancache", which is a concept that was [proposed](<https://lwn.net/Articles/545244/>) by Magenheimer in 2012. If the kernel has to dump a page of data, but would like to keep that data around if possible, it can put it into the cleancache, which will stash it aside somewhere. Should the need for that data arise, the kernel can copy it back out of the cleancache — if it is still there. Meanwhile, the page that initially contained that data can be reclaimed for other uses. 

Guaranteed CMA then builds on cleancache by allocating a region of physically contiguous memory at boot, when such allocations are relatively easy. That memory is then turned into a cleancache and made available to the kernel. Whenever the memory-management system reclaims pages of file-backed memory, it can choose to place the data from those pages into the cleancache. Should that data be needed, an attempt will be made to retrieve it from the cleancache before rereading it from disk. The memory reserved for CMA is thus available to the kernel when not allocated to a CMA user, but in a restricted way. 

At some point, some kernel subsystem will need a large, physically contiguous buffer. Requesting that buffer from the guaranteed CMA subsystem will result in an allocation from the reserved memory, after dropping any cached data that happens to be in the allocated region. This allocation can happen quickly, since that data has been cached with the explicit stipulation that it can be dropped at any time. This approach was [proposed](<https://lwn.net/Articles/619865/>) by Seongjae Park and Kim in 2014. 

This new subsystem is integrated with the existing CMA API, so CMA users need not change to make use of it. The reserved region is set up by way of a devicetree property explicitly requesting the "guaranteed" behavior. 

The end result is a version of CMA that is guaranteed to succeed as long as the total allocations do not exceed the size of the reserved area; existing CMA has a higher likelihood of failure. Since CMA usage is often restricted to a problematic device or two with known needs, sizing the reserved area for a specific system should be straightforward. 

The other advantage of guaranteed CMA is latency; if the memory is available, it can be allocated quickly. CMA in current kernels may have to migrate data out of the allocated region first, which takes time. The downside is that the memory reserved for guaranteed CMA can only be used for data that can be dropped at will; that will increase the pressure on the rest of the memory in the system. 

This patch series was posted just ahead of the [2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), where it is currently scheduled for a discussion in the memory-management track. There will probably not be a lot of comments on it ahead of that discussion. The patches are relatively small, though, and do not intrude into the memory-management subsystem on systems where CMA is not in use, so we might just see a transcendent-memory application actually go forward, some 15 years after the idea was first proposed.
