---
title: "Handling 32KB-block drives"
url: https://lwn.net/Articles/636978/
date: "March 18, 2015"
category: "Block layer-Large physical sectors"
author: "By Jake Edge March 18, 2015 LSFMM 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
March 18, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

There have been requests from certain disk drive manufacturers for the kernel to support 32KB block (or sector) sizes, James Bottomley said to kick off the discussion at a combined storage and filesystem session at the 2015 LSFMM Summit. He noted that the page cache could only handle 4KB granularity, and he didn't see that changing any time soon, which means that 32KB block sizes cannot be directly supported. But he wondered if aligning and sizing requests for 32KB boundaries most of the time would work for the disk drives. 

Dave Chinner said that XFS can already handle making requests that are aligned and sized correctly, but Bottomley asked if that included metadata reads and writes. Metadata is the biggest problem, Bottomley said. Shorter writes can be supported by doing a read-modify-write (RMW) underneath the covers, in the filesystem, block layer, or in the disk itself. 

Support for 4KB disk sectors, instead of the traditional 512-byte sectors, was [added to Linux](<https://lwn.net/Articles/322777/>) long ago, Ric Wheeler said. There are disk drives with 4KB logical and physical sectors out there now, Bottomley added. But that change matched up with the 4KB Linux page size. As Ted Ts'o pointed out, the page cache will need to be able to evict 4KB pages, which means that something will need to do an RMW operation on disks with larger block sizes. 

[ ![\[James Bottomley\]](https://static.lwn.net/images/2015/lsf-bottomley-sm.jpg) ](<https://lwn.net/Articles/637032/>)

Chris Mason pointed out that even if all filesystems had changes made in their data paths to do all I/O in 32KB chunks, and those changes were ready for the 4.1 kernel (which is, of course, only a thought experiment), it will be years before the code is in the hands of users. It will take at least a year before the enterprise distributions pick up the changes and at least another year before users are comfortable switching. Given that the disk drive makers want support now, it would make sense for them to add emulation of 512-byte sectors, as they did with the 4KB drives, so no changes are required of the kernel. 

Christoph Hellwig agreed, noting that virtual-memory eviction has various corner cases that will require page-sized writes. Chinner was also on board with that, saying that the "easy solution is to fix it in the drive". That is also true for supporting shingled magnetic recording (SMR) drives, he continued. 

Bottomley asked about ext4 support for doing 32KB I/O. Ts'o said that it would require some work but that it could be done. The same is true for Btrfs, Mason said. "We're all wrong but in slightly different ways", he said of Linux filesystem support. Ts'o said that there would need to be support added to the virtual-memory subsystem to support 32KB I/O. The filesystems could do their own RMW to ensure the full 32KB was in the cache when doing writes. 

Chinner asked about workloads that generate lots of small files. Bottomley said those would essentially waste an additional 28KB per file. Each would require an RMW operation as well, which might not perform all that well for some workloads. 

There was a suggestion that having 4KB emulation (rather than 512-byte emulation) would be better, but Chinner called it "immaterial". There are all kinds of "mapping tricks" already done by SSDs, any emulation would essentially be the same. SSD makers won't even say what the sector size is for those devices, Bottomley said. But Chinner said that he didn't care and didn't really want to know. Some were concerned about the performance implications of hiding RMW operations in the drive, however. 

One way to support larger block sizes in the page cache would be to move to larger pages throughout the kernel. The last time the idea of larger page sizes was raised with the memory management (MM) folks, they were not happy with the idea, Bottomley said. He wondered if it was worth raising the issue on day two of the summit in a plenary session. But Ric Wheeler said that the topic was raised in New Orleans (in 2013) and he didn't think the MM developers were "adamantly opposed" to the idea, just that no one was working on it. 

But, as Chinner pointed out, 32KB is not likely to be the end of the line. Even if the page size were increased to 32KB, disk drive manufacturers will someday want 128KB or 256KB (or beyond) for the block size. So a solution that is not dependent on the page size of the system is needed. Using `vmalloc()` allocations rather than contiguous allocations might help. [Compound pages](<https://lwn.net/Articles/619514/>) might also be part of any eventual solution. 

In the end, Bottomley summed up the discussion by saying that filesystems could "pull tricks" to make most I/O 32KB-friendly, but would need help from the MM subsystem to have it all be aligned correctly. Given the time frames, it would seem that drive makers need to do some kind of emulation for now. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
