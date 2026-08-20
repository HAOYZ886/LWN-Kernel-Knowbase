---
title: Supporting block I/O contexts
url: https://lwn.net/Articles/502472/
date: "June 18, 2012"
category: "Block layer; Filesystems-ext4; Solid-state storage devices"
author: "By Jonathan Corbet June 18, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 18, 2012

Memory storage devices, including flash, are essentially just random-access devices with some peculiar restrictions. Given direct access to the device, Linux kernel developers could certainly come up with drivers that would provide optimal performance and device lifetime. In the real world, though, these devices are hidden behind their own proprietary operating systems and software stacks; much of the real (commercial) value seems to be in the software bundled inside. As a result, the kernel must try to coax the device's firmware into doing an optimal job. Over time, the storage industry has added various mechanisms by which an operating system can pass hints down to the device; the "trim" or "discard" mechanism is one of those. Newer eMMC and unified flash storage (UFS) devices add a new hint in the form of "contexts"; patches exist to support this feature, but they seem to have raised more questions than they have answered. 

The standards documents describing contexts do not appear to be widely available—or at least findable. From what your editor has been able to divine, "contexts" are a small number added to I/O requests that are intended to help the device optimize the execution of those requests. They are meant to differentiate different types of I/O, keeping large, sequential operations separate from small, random requests. I/O can be placed into a "large unit" context, where the operating system promises to send large requests and, possibly, not attempt to read the data back until the context has been closed. 

Saugata Das recently posted [a small patch set](<https://lwn.net/Articles/502474/>) adding context support to the ext4 filesystem and the MMC block driver. At the lower level, context numbers are associated with block I/O requests by storing the number in the newly-added `bi_context` (in `struct bio`) and `context` (in `struct request`) fields. The virtual filesystem layer takes responsibility for setting those fields, but, in the end, it defers to the actual filesystems to come up with the proper context numbers. There is a new address space operation (called `get_context()`) by which the VFS can call into the filesystem code to obtain a context number for a specific request. The block layer has been modified to avoid merging block I/O requests if those requests have been assigned to different contexts. 

There was little discussion of the lower-level changes, which apparently make sense to the developers who have examined them. The filesystem-level changes have seen rather more discussion, though. Saugata's patch set only touches the ext4 filesystem; those changes cause ext4 to use the inode number of the file under I/O as the context number. Thus, all I/O requests to a single file will be assigned to the same context, while requests to different files would go into different contexts (within limits—eMMC hardware, for example, only supports 15 contexts, so many inode numbers will be mapped onto a single context number at the lower levels). The question that came up was: is using the inode number the right policy? Coming up with an answer involves addressing two independent questions: (1) what does the "context" mechanism actually do?, and (2) how can Linux filesystems provide the best possible context information to the storage devices? 

Arnd Bergmann (who has [spent a lot of time](<https://lwn.net/Articles/428584/>) understanding the details of how flash storage works) has [noted](<https://lwn.net/Articles/502476/>) that the standard is deliberately vague on what the context mechanism does; the authors wanted to create something that would outlive any specific technology. He went on to say: 

That said, I think it is rather clear what the authors of the spec had in mind, and there is only one reasonable implementation given current flash technology: You get something like a log structured file system with 15 contexts, where each context writes to exactly one erase block at a given time. 

The effect of such an implementation would be to concentrate data written under any one context into the same erase block(s). Given that, there are at least a couple of ways to use contexts to optimize I/O performance. 

For example, one could try to concentrate data with the same expected lifetime, so that, when part of an erase block is deleted, all of the data in that erase block will be deleted. Using the inode number as the context number could have that effect; deleting the file associated with that inode will delete all of its blocks at the same time. So, as long as the file is not subject to random writes (as, say, a database file might be), using contexts in this manner should reduce the amount of garbage collection and read-modify-write cycles needed when a file is deleted. 

Another helpful approach might be to use contexts to separate large, long-lived files from those that are shorter and more ephemeral. The larger files would be well-placed on the medium, and the more volatile data would be concentrated into a smaller number of erase blocks. In this case, using the inode number to identify contexts may or may not work well. Large files would be nicely separated, but the smaller files could be separated from each other as well, which may not be desirable: if several small files would fit into a single erase block, performance might be improved if all of those files were written in the same context. In this case, some other policy might be more advisable. 

But what should that policy be? Arnd suggested that using the inode number of the directory containing the file might work better. Various commenters thought that using the ID of the process writing to the file could work, though there are some potential difficulties when multiple processes write the same file. Ted Ts'o [suggested](<https://lwn.net/Articles/502478/>) that grouping files written by the same process in a short period of time could give good results. Also useful, he thought, might be to look at the size of the file relative to the device's erase block size; files much smaller than an erase block would be placed into the same context, while larger files would get a context of their own. 

A related idea, also [from Ted](<https://lwn.net/Articles/502479/>), was to look at the expected I/O patterns. If an existing file is opened for write access, chances are good that a random I/O pattern will result. Files opened with `O_CREAT`, instead, are more likely to be sequential; separating those two types of files into different contexts would likely yield better results. Some flags used with `posix_fadvise()` could also be used in this way. There are undoubtedly other possibilities as well. Choosing a policy will have to be done with care; poor use of contexts could just as easily reduce performance and longevity instead of increasing them. 

Figuring all of this out will certainly take some time, especially since devices with actual support for this feature are still relatively rare. Interestingly, [according to Arnd](<https://lwn.net/Articles/502480/>), there may be an opportunity in getting ext4 to supply context information early: 

Having code in ext4 that uses the contexts will at least make it more likely that the firmware optimizations are based on ext4 measurements rather than some other file system or operating system. From talking with the emmc device vendors, I can tell you that ext4 is very high on the list of file systems to optimize for, because they all target Android products. 

Ext4 is, of course, the filesystem of choice for current Android systems. So, conceivably, an ext4 implementation could drive hardware behavior in the same way that much desktop hardware is currently designed around what Windows does. 

Given that the patches are relatively small and that policies can be changed in the future without user-space compatibility issues, chances are good that something will be merged into the mainline as soon as the 3.6 development cycle. Then it will just be a matter of seeing what the hardware manufacturers actually do and adjusting accordingly. With luck, the eventual result will be longer-lasting, better-performing memory storage devices.
