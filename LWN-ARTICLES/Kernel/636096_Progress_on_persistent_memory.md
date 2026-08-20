---
title: Progress on persistent memory
url: https://lwn.net/Articles/636096/
date: "March 11, 2015"
category: "Memory management-Nonvolatile memory"
author: "By Jake Edge March 11, 2015 LSFMM 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
March 11, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015>)

It has been "the year of persistent memory" for several years now, Matthew Wilcox said with a chuckle to open his plenary session at the 2015 [Storage, Filesystem, and Memory Management summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>) in Boston on March 9\. Persistent memory refers to devices that can be accessed like RAM, but will permanently store any data written to them. The good news is that there are some battery-backed DIMMs already available, but those have a fairly small capacity at this point (8GB, for example). There are much larger devices coming, 400GB was mentioned, but it is not known when they will be shipping. From Wilcox's talk, it is clear that the two different classes of devices will have different use cases, so they may be handled differently by the kernel. 

It is good news that there are "exciting new memory products" in development, he said, but it may still be some time before we see them on the market. He is not sure that we will see them this year, for example. It turns out that development delays sometimes happen when companies are dealing with "new kinds of physics". 

[ ![\[Matthew Wilcox\]](https://static.lwn.net/images/2015/lsf-wilcox-sm.jpg) ](<https://lwn.net/Articles/636260/>)

Christoph Hellwig jumped in early on in the talk to ask if Wilcox's employer, Intel, would be releasing its driver for persistent memory devices anytime soon. Wilcox was obviously unhappy with the situation, but said that the driver could not be released until the ACPI specification for how the device describes itself to the system is released. That is part of the ACPI 6 process, which will be released "when ACPI gets around to it". As soon as that happens, Intel will release its driver. 

James Bottomley noted that there is a process within UEFI (which oversees ACPI) to release portions of specifications if there is general agreement by the participants to do so. He encouraged Intel to take advantage of that process. 

Another attendee asked whether it was possible to write a driver today that would work with all of the prototype devices tested but wouldn't corrupt any other of the other prototypes that had not been tested. Wilcox said no; at this point that isn't the case. "It is frustrating", he said. 

#### Persistent memory and `struct page`

He then moved on to a topic he thought would be of interest to the memory-management folks in attendance. With a 4KB page size, and a `struct page` for each page, the 400GB device he mentioned would require 6GB just to track those pages in the kernel. That is probably too much space to "waste" for those devices. But if the kernel tracks the memory with page structures, it can be treated as normal memory. Otherwise, some layer, like a block device API, will be needed to access the device. 

Wilcox has been operating under the assumption that those kinds of devices won't use `struct page`. On the other hand, Boaz Harrosh (who was not present at the summit) has been pushing [patches](<https://lwn.net/Articles/608538/>) for other, smaller devices, and those patches do use `struct page`. That makes sense for that use case, Wilcox said, but it is not the kind of device he has been targeting. 

Those larger devices have wear characteristics that are akin to those of NAND flash, but it isn't "5000 cycles and the bit is dead". The devices have wear lifetimes of 107 or 108 cycles. In terms of access times, some are even faster than DRAM, he said. 

Ted Ts'o suggested that the different capacity devices might need to be treated differently. Dave Chinner agreed, saying that the battery-backed devices are effectively RAM, while the larger devices are storage, which could be handled as block devices. 

Wilcox said he has some preliminary patches to replace calls to `get_user_pages()` for these devices with a new call, `get_user_sg()`, which gets a scatter/gather list, rather than pages. That way, there is no need to have all those page structures to handle these kinds of devices. Users can treat the device as a block device. They can put a filesystem on it and use `mmap()` for data access. 

That led to a discussion about what to do to handle a `truncate()` on a file that has been mapped with `mmap()`. Wilcox thinks that Unix, thus Linux, has the wrong behavior in that scenario. If a program accesses memory that is no longer part of the mapped file due to the truncation, it gets a `SIGSEGV`. Instead, he thinks that the `truncate()` call should be made to wait until the memory is unmapped. 

Making `truncate()` wait is trivial to implement, Peter Zijlstra said, but it certainly changes the current behavior. He suggested adding a flag to `mmap()` to request this mode of operation. That should reduce the surprise factor as it makes the behavior dependent on what is being mapped. Ts'o said that he didn't think the kernel could unconditionally block truncate operations for hours or days without breaking some applications. 

Getting back to the question of the drivers, Ts'o asked what decisions needed to be made and by when. The battery-backed devices are out there now, so patches to support them should go in soon, one attendee said. Hellwig said that it would make sense to have Harrosh's driver and the Intel driver in the kernel. People could then choose the one that made sense for their device. In general, that was agreeable, but the driver for the battery-backed devices still needs some work before it will be ready to merge. Bottomley noted that means that the group has decided to have two drivers, "one that needs cleaning up and one we haven't seen". 

#### New instructions

Wilcox turned to three new instructions that Intel has announced for its upcoming processors that can be used to better support persistent memory and other devices. The first is `clflushopt`, which adds guarantees to the cache-line flush (`clflush`) instruction. The main benefit is that it is faster than `clflush`. Cache-line writeback (`clwb`) is another, which writes the cache line back to memory, but still leaves it in the cache. The third is `pcommit`, which acts as a sort of barrier to ensure that any prior cache flushes or writebacks actually get to memory. 

The effect of `pcommit` is global for all cores. The idea is to do all of the flushes, then `pcommit`; when it is done, all that data will have been written. On current processors, there is no way to be sure that everything has been stored. He said that `pcommit` support still needs to be added to DAX, the [direct access block layer](<https://lwn.net/Articles/591779/>) for persistent memory devices that he developed. 

Ts'o asked about other processors that don't have support for those kinds of instructions, but Wilcox didn't have much of an answer for that. He works for Intel, so other vendors will have to come up with their own solutions there. 

There was also a question about adding a per-CPU commit, which Wilcox said was under internal discussion. But Bottomley thought that if there were more complicated options, that could just lead to more problems. Rik van Riel noted that the scheduler could move the process to a new CPU halfway through a transaction anyway, so the target CPU wouldn't necessarily be clear. In answer to another question, Wilcox assured everyone that the flush operations would not be slower than existing solutions for SATA, SAS, and others. 

#### Error handling

His final topic was error handling. There is no status register that gives error indications when you access a persistent memory device, since it is treated like memory. An error causes a machine check, which typically results in a reboot. But if the problem persists, it could just result in another reboot when the device is accessed again, which will not work all that well. 

To combat this, there will be a log of errors for the device that can be consulted at startup. It will record the block device address where problems occur and filesystems will need to be able to map that back to a file and offset, which is "not the usual direction for a filesystem". Chinner spoke up to say that XFS would have this feature "soon". Ts'o seemed to indicate ext4 would also be able to do it. 

But "crashing is not a great error discovery technique", Ric Wheeler said; it is "moderately bad" for enterprise users to have to reboot their systems that way. But handling the problems when an `mmap()` is done for that bad region in the device is not easy either. Several suggestions were made (a signal from the `mmap()` call or when the page table entry is created, for example), but any of them mean that user space needs to be able to handle the errors. 

In addition, Chris Mason said that users are going to expect to be able to `mmap()` a large file that has one bad page and still access all of the other pages from the file. That may not be reasonable, but is what they will expect. At that point, the discussion ran out of time without reaching any real conclusion on error handling. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
