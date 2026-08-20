---
title: Merging swsusp2
url: https://lwn.net/Articles/113555/
date: "December 1, 2004"
category: Software suspend
author: 
---

Once upon a time, the 2.6 kernel had three software suspend implementations \- two of which were in the mainline - but none which were seen to work all that well. Since then, the two mainline implementations have been merged, and the out-of-tree [swsusp2](<http://softwaresuspend.berlios.de/>) implementation has come a long way. Still, two implementations of a low-level core function seems like too many, so there is interest in bringing them together in the mainline. Swsusp2 developer Nigel Cunningham has made a new effort in that direction by posting [a set of 51 patches](<http://lwn.net/Articles/112704/>) which merge swsusp2 into the 2.6 kernel. 

There is a great deal of code in these patches. Some of the more interesting pieces include: 

  * A [set of new exported symbols](<https://lwn.net/Articles/113560/>); these include fundamental things like `sys_ioctl()`, `avenrun`, and a number of low-level swap functions. These exports were poorly received; there is currently a great interest in reducing the number of symbols exported to modules. A patch which, instead, exports fundamental interfaces is bound to encounter some resistance. 

The swsusp2 code wants these symbols exported because the entire suspend mechanism can be built as a module and loaded only when the system is to be suspended. This can be a nice feature; swsusp2 is a lot of code, and it is all excess baggage anytime the system is actually being used. The costs of making swsusp2 modular may prove too high for that feature to be accepted into the mainline, however. 

  * [A change to the workqueue API](<https://lwn.net/Articles/113565/>) allowing the creator of a workqueue to specify whether that queue should be left running during the suspend process. Some workqueues perform tasks which are needed during the image writeout process, and thus cannot be stopped prematurely. 

  * [A replacement refrigerator](<https://lwn.net/Articles/113572/>), the code charged with putting all processes on hold so that the system is in a quiet state before suspending. The intrusive macros which characterized the swsusp2 refrigerator some time ago are gone. The new version relies upon a `PF_SYNCTHREAD` task flag to keep processes (temporarily) from being frozen while they are doing some work involved with writing data to disk. 

  * A complicated I/O infrastructure for writing the suspend image to disk. The new I/O code is said to be much faster, and it supports features like image compression, writing to swap files or LVM devices, etc. Future plans call for more esoteric features, such as suspending across a network. These features may be nice, but it has been [pointed out](<https://lwn.net/Articles/113575/>) that the swsusp2 I/O code alone outweighs the entire mainline software suspend code by a significant amount. 

  * Various types of "nice displays" when a suspend or resume is in progress. These displays include progress bars, splash screens, and more. 

Nigel [offers a number of reasons](<https://lwn.net/Articles/113579/>) for merging swsusp2. It is claimed to be much faster as a result of the use of asynchronous I/O, readahead on resume, and (for systems with slow drives) image compression. It is far more configurable; users can select the sort of display they like, image compression and/or encryption, etc. Suspending to swap files, LVM devices, and more is supported. And so on. There is little disagreement that swsusp2 offers some nice features, but there is some concern over how Nigel is trying to proceed: 

I'm thus seeking to simply merge the existing code, let Pavel and others get to the point where they're ready to say "Okay, we're satisfied that suspend2 does everything swsusp does and more and better." Then we can remove swsusp. 

The kernel developers are increasingly resistant to wholesale merging of large blocks of code - especially when that code duplicates functionality already found in the kernel. They would rather see a series of incremental patches, each of which takes a small, useful step in the right direction. Nigel does not want to do that; swsusp2 is vastly different, internally, than the mainline suspend code, and evolving one into the other looks like a long, painful, and pointless job. He may have to do that work, however, before any of the swsusp2 code can be merged. 

A bigger obstacle, however, may be the fact that, while swsusp2 was being developed, the mainline software suspend code was progressing too. Your editor is able to reliably suspend to memory and disk with a vanilla 2.6.9 kernel. SUSE enables software suspend and [calls it a feature](<http://www.novell.com/products/linuxprofessional/features/new.html>) in its 9.2 release. Since the in-kernel suspend code seems to actually work, enthusiasm for replacing it with a larger, more complex version is not as high as it might otherwise have been. The ultimate fate of swsusp2 may yet be to contribute its best improvements to the mainline, but to never be merged in its entirety..
