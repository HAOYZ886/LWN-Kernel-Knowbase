---
title: "The slab and protected-memory allocators"
url: https://lwn.net/Articles/753154/
date: "May 1, 2018"
category: "Memory management-Slab allocators; Security-Kernel hardening"
author: "By Jonathan Corbet May 1, 2018 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 1, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

One of the core jobs of the memory-management subsystem is to make memory available to other parts of the kernel when the need arises. The memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit hosted a pair of sessions on new or improved allocation functions for the kernel covering the slab allocators and protectable memory.
