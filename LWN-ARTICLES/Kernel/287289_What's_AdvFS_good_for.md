---
title: "What's AdvFS good for?"
url: https://lwn.net/Articles/287289/
date: "June 25, 2008"
category: AdvFS; Filesystems
author: "By Jonathan Corbet June 25, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
June 25, 2008

On June 23, HP [announced](<http://lwn.net/Articles/287057/>) that it was releasing the source for the "Tru64 Advanced Filesystem" (or AdvFS) under version 2 of the GPL. This is, clearly, a large release of code from HP. What is a bit less clear is what the value of this release will be for Linux. In the end, that value is likely to be significant, but it will be probably realized in relatively indirect and difficult-to-measure ways. 

AdvFS was originally developed by Digital Equipment Corporation for its version of Unix; HP picked it up when it acquired Compaq, which had acquired DEC in 1998. This filesystem offers a number of the usual features. It is intended to be a high-performance filesystem, naturally. Extent-based block management and directory indexes are provided. It does journaling for fast crash recovery. There is an undelete feature. AdvFS is also designed to work in clustered environments. 

Much of the thought that went into AdvFS was concerned with avoiding the need to take the system down. There is a snapshot feature which can be used to make consistent backups of running systems. Defragmentation can be done online. There is a built-in volume management layer which allows storage devices to be added to (or removed from) a running filesystem; files can also be relocated across devices. The internal volume manager can perform striping of files across devices, but nothing more advanced than that; AdvFS will happily work on top of a more capable volume manager, though. 

There are a few things which AdvFS does not have. There is no checksumming of data, and, thus, no ability to catch corruption. Online filesystem integrity checking does not appear to be supported. The maximum filesystem size (16TB) probably seemed infinite in the early 1990's, but it's starting to look a little tight now. In general, AdvFS looks like something which was a very nice filesystem ten or fifteen years ago, but it has little that is not either available in Linux now, or in the works for the near future. And AdvFS doesn't even work with Linux - no porting effort has been made, and it's not clear that one will be made. So is this release just another dump of code being abandoned by its corporate owner? 

One could make a first answer by saying that, even if this were true, it would still be welcome. If a company gives up on a piece of code, it's far preferable to put it out for adoption under the GPL than to let it rot until nobody can find it anymore. But there may well be value in this release. 

Even if there is no point in trying to make it work under Linux, the AdvFS code is the repository of more than a decade of experience of making a high-end filesystem work in a commercial environment. Your editor had stopped working with DEC systems by the time AdvFS came out, but the word he heard from others is that the early releases were, shall we say, something that taught administrators about the value of frequent backups. But after a few major releases, AdvFS had stabilized into a fast, solid, and reliable filesystem. The current code will embody all of the hard lessons that were learned in the process of getting to that point. 

Chris Mason, who is currently working on the Btrfs filesystem, [puts it this way](<https://lwn.net/Articles/287301/>): 

The idea is that well established filesystems can teach us quite a lot about layout, and about the optimizations that were added in response to customer demand. Having the code to these optimizations is very useful. 

Having that code licensed under the GPL is especially useful: any code which is useful in its current form can be pulled quickly into Linux. And, even when the code itself cannot be used, the ideas that it embodies can be borrowed without fear. And that is [exactly what HP was hoping to encourage](<https://lwn.net/Articles/287306/>) with this release: 

In case its not clear, this is a GPLv2 technology release, not an actual port to Linux. We're hoping that the code and documentation will be helpful in the development of new file systems for Linux that will provide similar capabilities, and perhaps used to make tweaks to existing file systems. 

And that would appear to be likely to happen. Over time, the best ideas and experience from AdvFS should find their way into the filesystems supported by Linux, even if AdvFS, itself, never becomes one of those filesystems. So HP has made a significant contribution to the kernel development process, one which will probably never show up in the changeset counts and other easily-obtained metrics. 

(Those interested in learning more about AdvFS would be well advised to grab the documentation tarball from the [AdvFS sourceforge page](<http://advfs.sourceforge.net/>). The "Hitchhiker's guide" is a good starting place, though, at 229 pages, it's not for hitchhikers who prefer to travel light.)
