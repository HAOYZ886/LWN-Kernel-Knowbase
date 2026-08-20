---
title: Merging drivers early
url: https://lwn.net/Articles/270960/
date: "February 27, 2008"
category: "Development model-Driver merging; Device drivers"
author: "By Jake Edge February 27, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
February 27, 2008

Drivers tend to be a world unto themselves, with bugs only affecting a subset--often a tiny subset--of kernel users. Until a driver gets merged into the kernel though, anyone wishing to test it, or help clean it up, has to jump through some hoops. To try and help reduce those barriers, Linus Torvalds and others have been advocating early merging of drivers; getting them into the kernel and incrementally improving them from there. 

This policy of early merging of drivers is not universally embraced, with a recent remote DMA (RDMA) ethernet driver, which lives in the `infiniband` tree, getting singled out. Based on the problems he observed in the driver, Adrian Bunk [asked](<https://lwn.net/Articles/271004/>): ""Is it really intended to merge drivers without _any_ kind of review?"" This was, perhaps, an overly dramatic question as the driver _has_ undergone review, but not all of the changes have been reflected in the mainline version. There is still work to do, as Infiniband maintainer Roland Dreier points out: 

Just to be clear, this driver was reviewed. Many issues were found, and many were fixed while others are being worked on. 

It's a judgment call when to merge things, but in this case given the good engagement from the vendor, I didn't see anything to be gained by delaying the merge. 

It is a sentiment shared by other kernel hackers as well. When there is a developer who is responding to the feedback along with a working driver, getting it into the mainline kernel--where more eyes can scrutinize it--is seen as a positive step. Torvalds is very interested in seeing drivers earlier so that more collaboration can happen: 

I'd really rather have the driver merged, and then *other* people can send patches! 

The thing is, that's what merging really means - people can work on it sanely together. Before it's merged, it's a lot harder for people to work on it unless they are really serious about that driver, so before merging, the janitorial kind of things seldom happen. 

Other maintainers explained their criteria for accepting drivers that are not quite up to usual kernel standards. The consensus seems to be that drivers with the following characteristics are acceptable: 

  * compiles and seems to work
  * has no obvious security holes
  * has an active maintainer
  * does not affect people who don't have the hardware
  * does not introduce unnecessary or not fully thought out user space interfaces

There is little in the way of a downside to making drivers available earlier. Since they are self-contained, they generally don't cause problems elsewhere in the kernel. As long as reviewers are keeping an eye out for security problems, which could lead to an unsuspecting user's box being compromised, there are not many ways for a driver to negatively impact the kernel as a whole. User space interfaces via `ioctl()`, sysfs, or other means also need to be closely examined as they will have to be maintained as part of the kernel interface. 

Along the way, much grumbling was heard about `checkpatch`, the perl script that complains about various stylistic problems with a patch. Notably absent from the list above is any kind of requirement that `checkpatch` errors or warnings be handled. The main complaint against `checkpatch` is its checks for line length; the resulting "fixes" to kernel source sometimes leave much to be desired. While it is generally agreed that too many overly long lines can result in code that is difficult to read, exactly what constitutes such a line tends to be an aesthetic judgment. Slavish adherence to a fixed number of characters on a line in order to appease `checkpatch` is clearly seen as a problem. 

To some, this makes `checkpatch` less than useful, bordering on dangerous to readability. Torvalds stated that he has [considered removing it from the kernel tree](<https://lwn.net/Articles/271051/>) on more than one occasion. Human judgment is required to interpret the warnings from `checkpatch` and sometimes it is not being applied. On the other hand, Ingo Molnar gives an [impassioned defense of the tool](<http://lwn.net/Articles/270620/>): 

Based on this first hand experience, my opinion about checkpatch has changed, rather radically: i now believe that checkpatch is almost as important to the long term health of our kernel development process as BitKeeper/Git turned out to be. If i had to stop using it today, it would be almost as bad of a step backwards to me as if we had to migrate the kernel source code control to CVS. 

Molnar goes on to outline the pros and cons of `checkpatch`, all of which stands in stark contrast to some of his [earlier complaints about the tool](<http://article.gmane.org/gmane.linux.kernel/585411>). 

For most drivers, the path into the kernel has been made a lot easier. This will have the effect of getting working, or mostly working, drivers into the hands of users more quickly. More importantly, it will also get the code into the hands of the Linux kernel community faster. The likely result is a fully working, cleanly coded driver sooner than it might have happened in the past. An already quick turnaround for hardware support in Linux may have just gotten faster.
