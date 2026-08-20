---
title: "Toward better performance on large-memory systems"
url: https://lwn.net/Articles/753171/
date: "May 2, 2018"
category: "Memory management-Scalability"
author: "By Jonathan Corbet May 2, 2018 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 2, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Christoph Lameter works in a different computing environment than most of us; he supports high-volume trading applications that need every bit of performance that the fastest hardware can give them. Even then, it seems that isn't fast enough. In a memory-management-track session at the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Lameter described some of the problems he has encountered and approaches he is considering to address them. 

He is working with a large system that has four 10Gb/s network interfaces on it. Those interfaces run at full speed during bursts of activity, pouring data into the page cache, which sits in 1TB of main memory. That data is then archived onto a 100TB storage array. This system is vital for his company's operations; it must be possible to answer questions from regulators on exactly when any given packet arrived. It is important, he said, that the system works. 

Unfortunately, he is having problems writing data to the archive; the maximum data rates from the page cache are 4-5GB/s. The only way the system can keep up is to remove some of the network interfaces. A much higher data rate (around 10GB/s) could be obtained by using direct I/O and avoiding the page cache altogether, but he would rather not do that. There are also a number of analysis processes running on this system, and they benefit from having the data in the page cache. So he's not sure what to do. 

The problem is going to get worse; the company is upgrading to 100Gb/s interfaces and wants to achieve rates of 40GB/s to the disk. Lameter thinks that these performance issues are specific to Intel hardware; everything [![\[Christoph
Lameter\]](https://static.lwn.net/images/conf/2018/lsfmm/ChristophLameter-sm.jpg)](<https://lwn.net/Articles/753173/>) works as desired on POWER8 processors. The root of the problem is the 4KB system page size, which is quite small on such a system. Unfortunately, filesystems and block devices in Linux do not understand huge pages. 

He is considering trying to increase the base page size above 4KB. This is not a new idea; developers have been trying to make this work for years. Hugh Dickins noted that William Lee Irwin's [page clustering](<https://lwn.net/Articles/23785/>) patches worked fifteen years ago, but they proved to be too complex to be merged. A larger page size can also break some applications that expect to be able to map pages on a 4KB granularity. 

Lameter has done some work on the "order N page cache", which would be able to store pages of any order. But he never got around to implementing `mmap()`, he said, and the patch had too many fragmentation problems to be merged. An [alternative](<https://lwn.net/Articles/686690/>) is transparent huge pages in the page cache. The problem with that is that no filesystem (other than tmpfs) supports it currently. He has looked at (ab)using the DAX mechanism to get huge-page mappings in ordinary memory; one could also use DAX for real on nonvolatile memory. 

Returning to the 4KB page-size issue, Lameter pointed out that, at that size, about 2% of the system's memory is taken up by `page` structures. A system with 4TB of RAM must manage one-billion `page` structures. There are systems supporting 20TB of nonvolatile RAM coming soon, he said; the overhead is becoming unsupportable. But Dickins said that he wasn't worried about a 2% space tax. 

Almost any solution to Lameter's performance problems is going to require more reliable allocation of large, physically contiguous memory areas. He has been playing with an XArray cache for memory chunks that would allow them to be moved as needed. Simple slab allocator support has been implemented, but a lot of work is needed still; allocators need to provide callbacks to allow memory chunks to be relocated. Alternatives include reserving memory at boot for large allocations, the `MAP_CONTIG` option to `mmap()`, or Java-style garbage collection. "But we are all horrified" by that last idea, he said. 

Rik van Riel suggested working with the filesystem developers to support larger block sizes. But some filesystems (XFS, for example) already have that support; the real problem is in the page cache. Dickins suggested creating a mechanism like transparent huge pages that would opportunistically allocate larger chunks for the page cache; those chunks would be smaller than typical huge pages, though. 

From that point on, the conversation went around in circles about whether these performance issues are truly a hardware problem or not. There was no useful outcome from that discussion, though, and it petered out as beer time approached.
