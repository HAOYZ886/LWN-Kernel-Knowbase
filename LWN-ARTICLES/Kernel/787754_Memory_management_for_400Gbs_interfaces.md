---
title: Memory management for 400Gb/s interfaces
url: https://lwn.net/Articles/787754/
date: "May 8, 2019"
category: "Networking-Performance"
author: "By Jonathan Corbet May 8, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Christoph Lameter has spent years improving Linux for high-performance computing tasks. During the memory-management track of the 2019 Linux Storage, Filesystem, and Memory-Management Summit, he talked about the problem of keeping up with a 400Gb/s network interface. At that speed, there simply is no time for the system to get its work done. Some ways of improving the situation are in sight, but it's a hard problem overall and, despite some progress, the situation is getting worse. 

The problem is that, at those data rates, the kernel's page cache is overwhelmed and simply cannot keep up. That is not entirely the kernel's fault; there is an increasing mismatch between interface speeds and memory speeds. As a result, sites have stopped upgrading their Infiniband fabrics; there is no point in making the fabric go any faster. A PCIe 3 bus can manage 1GB/s in each lane; x86 systems have 44 lanes, all of which must be used together to keep up with a 400Gb/s interface. So extra capacity on the fabric side is not useful. 

PCIe 4 offers a bit of relief in the form of a doubled transfer rate but, Lameter said, that effort is currently stalled. Meanwhile latencies are high. The whole Intel computing paradigm is in trouble, he said; it is no [![\[Christoph
Lameter\]](https://static.lwn.net/images/conf/2019/lsfmm/ChristophLameter-sm.jpg)](<https://lwn.net/Articles/787759/>) longer suitable for high-performance computing. The [OpenCAPI](<https://web.archive.org/web/20190620074439/https://opencapi.org/>) architecture is somewhat faster than PCIe, but it is only available on POWER9 systems. The fastest interlink available currently is NVIDIA's [NVLink](<https://www.nvidia.com/en-us/data-center/nvlink/>), which can attain 300GB/s; that too is only available on POWER9. 

In the area of memory bandwidth, processor vendors are adding memory channels; Intel has six of them now, AMD has eight. But that adds more pins and complicates routing. These systems can move 20GB/s in each channel, which puts an upper bound on what any individual thread can do; a single thread cannot keep up with even a 100Gb/s network interface. So multiple cores are needed to get the job done. There is some potential in [GDDR](<https://en.wikipedia.org/wiki/GDDR_SDRAM>) and [HBM](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>) memory; those, combined with NVLink, show that it is possible to do better than current systems do. 

Jesper Brouer has done [a lot of work](<https://lwn.net/Articles/629155/>) improving the performance of the kernel's network stack; he was able to get up to a rate of 10Gb/s. But when the data rate is raised to 100Gb/s, there are only 120ns available to process each packet; the system cannot take even a single cache miss and keep up. So that kind of network processing must be done in hardware. The development of the express data path (XDP) mechanism is another sign that you just cannot use the network stack at those rates. Moving some functions, such as checksums and timestamps, to the interfaces can help somewhat. 

Then, there are problems with direct I/O in the kernel; it works with arrays of pointers to 4KB pages, meaning there is little opportunity for batching. 1GB transfers are thus relatively slow. The 5.1 kernel has improved the situation by allowing for larger chunks of data to be managed; that results in lower cache use, fewer memory allocations, and less out-of-band data to communicate to devices — and, thus, higher performance. But this is a new feature that will not make its way into the major distributions for some time. 

The kernel's page cache, Lameter said, simply does not scale. The fact that it can't work with large pages makes things worse; users have to use direct I/O or bypass the kernel entirely, which should not be necessary. That said, there has been some progress. The XArray data structure enables handling multiple page sizes in the page cache. The [slab movable objects](<https://lwn.net/Articles/784964/>) work can help to address fragmentation. Work is being done to [avoid acquiring the `mmap_sem` lock](<https://lwn.net/Articles/730531/>) while handling page faults, and support for huge pages is being added to filesystems. One option that has not been pursued, he said, is to create a kernel that uses 2MB as its base page size or increasing the base size to an intermediate value by grouping 4KB pages. 

There is some value in persistent memory, which is attached to the memory channels and is thus fast. The DAX mechanism can be used to avoid the page cache altogether. This storage is currently limited in size, though, and cannot be used with RDMA due to the [well-discussed problems](<https://lwn.net/Articles/787636/>) with `get_user_pages()`. 

In the future, he said, kernel developers need to be thinking about terabit streams. There is 3D hologram streaming on the horizon, he said. We increasingly need to move massive amounts of data, but everybody is busy trying to avoid the kernel's limitations to get this work done. Part of the solution, eventually, will be new hardware architectures for high-performance computing. 

It would be nice, he concluded, if the memory-management subsystem had a road map showing how it plans to meet these challenges. In the brief moment before the session ended, Matthew Wilcox said that not having a road map is not necessarily a bad thing. The development community is indeed working on these problems; each developer has taken on one piece of it. Coordinating all of this work is what LSFMM is all about; he now knows what others need from the subsystem.
