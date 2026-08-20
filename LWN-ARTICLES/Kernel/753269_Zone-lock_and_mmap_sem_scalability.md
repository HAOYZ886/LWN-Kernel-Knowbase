---
title: "Zone-lock and mmap_sem scalability"
url: https://lwn.net/Articles/753269/
date: "May 3, 2018"
category: "Memory management-mmap sem"
author: "By Jonathan Corbet May 3, 2018 LSFMM"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 3, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The memory-management subsystem is a central point that handles all of the system's memory, so it is naturally subject to scalability problems as systems grow larger. Two sessions during the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit looked at specific contention points: the zone locks and the `mmap_sem` semaphore.
