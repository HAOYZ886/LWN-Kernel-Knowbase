---
title: Making use of persistent memory
url: https://lwn.net/Articles/674752/
date: "February 10, 2016"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet February 10, 2016 linux.conf.au 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 10, 2016

* * *

[linux.conf.au 2016](<https://lwn.net/Archives/ConferenceByYear/#2016-linux.conf.au>)

Persistent memory holds a lot of promise: what's not to like about vast amounts of directly-attached memory that remembers its contents over a power cycle? For some years we have been told that large persistent-memory arrays are coming; now it seems that they are about to arrive. According to Matthew Wilcox, who spoke on the topic at [linux.conf.au](<http://linux.conf.au/>) 2016, Intel wants persistent memory to become a regular platform feature. It will start by shipping server platforms supporting 6TB arrays in 2017. Matthew's question was: how will we make use of that persistent memory? 

There are a lot of ideas out there, he said, many of which have been promoted by academics — most of whom are not greatly concerned about practicality. Start, for example, with the idea of total system persistence, where the entire system can be turned off at any time and, when powered back on, will simply pick up where it left off. The problem here is that the CPU caches are _not_ persistent, and there is no easy [![\[Matthew Wilcox\]](https://static.lwn.net/images/conf/2016/lca/MatthewWilcox-sm.jpg)](<https://lwn.net/Articles/674879/>) way to know when all writes to main memory have completed. Whole-system persistence is a "delightful" idea, but it is not something that can be done with today's hardware. 

Stepping back a bit, one can try for application persistence — using persistent memory to make cheap application snapshots. Unfortunately, the cache problem exists here as well. 

Perhaps what needs to be done is to completely redesign the operating system, creating a new system designed around persistent memory from the beginning. The "new" ideas that proponents of this idea bring up tend to have a familiar ring to them: microkernels, nanokernels, unikernels, etc. Matthew suggested that some people see new technology as an opportunity to push the same ideas they have been promoting for years. He wishes them the best, but is looking for something that will work today. 

Some developers at Intel created a new filesystem for persistent memory called [pmfs](<https://github.com/linux-pmfs/pmfs>). They are, he said, smart people, but they are not Linux kernel developers. As a result, the work they did is not suitable for production settings. 

What _can_ be done today is to package up a persistent-memory array and export it as a fast block device. We [have support for that](<https://lwn.net/Articles/640113/>) in the kernel now, but, he said, it doesn't feel like a general-purpose solution. To get to that more general solution, he set out to make some small modifications to existing filesystems so that they could make use of persistent memory; he allowed that it was perhaps good that Dave Chinner was unable to attend the conference, since Dave might just quibble with Matthew's notion of "small". In any case, Matthew started with the existing execute-in-place support, rewrote it, and ended up with [the subsystem now known as DAX](<https://lwn.net/Articles/610174/>). 

#### Beyond filesystems

There is more to proper persistent-memory support than getting the filesystems to work, though. Down at the CPU level, Intel's designers have created a new set of instructions for use with persistent memory. The intent is to allow developers to initiate the flushing of specific cache lines; it is not possible to know when a write completes, but the operation can at least be started when needed. The `CLFLUSH` instruction has existed for years; its job is to remove a line of data from the cache. It is not optimal for persistent-memory applications, though, because it serializes the instruction stream, hurting performance. 

That shortcoming will be addressed with the `CLFLUSHOPT` instruction, which does not perform that serialization. In the future, there will also be `CLWB`, which starts writing out the cache line but does not remove the data from the cache. Finally, the `PCOMMIT` instruction will ensure that all data written prior to the last store fence is persistent. It may not be written to a specific persistent-memory array, but it will be in persistent storage somewhere. 

There is still the question of how application programmers should make use of persistent memory. One option would be to create a special-purpose programming language that natively understands persistence, but that, he said, is not particularly interesting. A bit more practical might be to modify the runtime virtual machines for language like Python to get them to issue the new instructions when needed. That would avoid the need for code changes in general, but is still not entirely interesting to "dinosaurs" like him, who want to program in languages like C. 

For such people, there is [a whole set of new libraries](<http://pmem.io/nvml/>) available. At the lowest level, `libpmem` simply provides easy access to the new instructions, but adds little otherwise. The `libvmmalloc` library supplies a replacement for `malloc()` that can allocate from persistent memory. It can also be used for non-persistent applications; indeed, applications can be linked against this library and be unaware that they are using persistent memory at all. For developers who are willing to code specific persistent-memory awareness into their applications, there is `libvmem`. It provides better performance on persistent memory, but is still mainly expected to be used with memory used as if it were volatile. 

Developers wanting utilities to help with the storage of persistent memory can use `libpmemblk`, which provides access to atomic blocks of memory. Log-oriented applications, which append data to an existing structure, can use `libpmlog` to manage logs easily in persistent memory. But the interesting one, he said, is `libpmemobj`, which provides a transactional object store on top of persistent memory. It provides locking that persists over system boots, type safety, and data structures like doubly linked lists and a key-value store. It can handle replication of data across files. And, for those who are so inclined, C++ support has been added in recent months. 

Quite a bit of functionality has been built on these libraries, he said. There is, for example, a MySQL storage engine that uses it. Interested developers can go to [pmem.io](<http://pmem.io/>), which hosts a blog, pointers to the source, and other information. There is also [intel.com/nvm](<http://intel.com/nvm/>), which has mostly marketing material about the upcoming persistent-memory hardware. It appears that, after years of hype, the hardware will soon be available, and the software support will be there for it as well. 

The [video of this talk](<http://mirror.linux.org.au/linux.conf.au/2016/04_Thursday/Costa_Hall/Using_Persistent_Memory_for_Fun_and_Profit.webm>) is available for those wanting more information. 

[Your editor thanks LCA for assisting with his travel expenses.]
