---
title: Shared memory mappings for devices
url: https://lwn.net/Articles/753481/
date: "May 7, 2018"
category: "Device drivers-Support APIs"
author: "By Jake Edge May 7, 2018 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 7, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

In a short filesystem-only discussion at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Jérôme Glisse wanted to talk about some (more) changes to support GPUs, FPGAs, and RDMA devices. In [other talks at LSFMM](<https://lwn.net/Articles/752564/>), he discussed changes to `struct page` in support of these kinds of devices, but here he was looking to discuss other changes to support mapping a device's memory into multiple processes. It should be noted that I had a hard time following the discussion in this session, so there may be significant gaps in what follows. 

[ ![\[Jérôme Glisse\]](https://static.lwn.net/images/2018/lsf-glisse-sm.jpg) ](<https://lwn.net/Articles/753479/>)

A device driver stores the device context in the `private_data` field of the `struct file`, Glisse said, which has worked well, but is now becoming a problem. There are new devices that developers want to be able to attach to an `mm_struct`. In addition, though, those devices are still being used by a legacy API that needs to be preserved. 

Glisse said that his first idea was to associate the device context with an `mm_struct`. That led to various developers to try to better understand the use case. Ted Ts'o summarized what came out of that. He suggested that what Glisse wanted was for every `mm_struct` to have a unique ID associated with it and to store that unique ID in the device context. Any `ioctl()` that tried to access the device would only work if the unique ID in the `mm_struct` of the caller is the same as that buried in the device context. Glisse agreed that would do what he was aiming for. 

Ts'o noted that you can't use the address of the `mm_struct` because that would vary between processes. It wouldn't necessarily be implemented as a unique ID, he said, but that is conceptually how it would work. Kent Overstreet suggested a simple global sequence number for `mm_struct`. Processes that shared them would have the same sequence number, so the `ioctl()` enforcement could be done. 

After questions about where the changes might lie, Glisse said that he had not written any code yet, but that he did not think changes to the virtual filesystem (VFS) layer would be required. VFS maintainer Al Viro did not really think it mattered where the changes would be made, his question was whether the behavior is needed. Glisse said that it is; it will allow the legacy code to continue running on GPUs, while allowing for more modern uses of the devices going forward.
