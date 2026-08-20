---
title: The end of the 4.9 merge window
url: https://lwn.net/Articles/703779/
date: "October 19, 2016"
category: "Releases-4.9"
author: "By Jonathan Corbet October 19, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 19, 2016

By the time that Linus [released 4.9-rc1](<https://lwn.net/Articles/703664/>) and closed the merge window for the 4.9 development cycle, 14,308 non-merge changesets had found their way into the mainline repository. As expected, this cycle has already broken the previous record for the busiest cycle ever, and it has a while to go still. 820 of those changesets were merged after [last week's summary](<https://lwn.net/Articles/703110/>) was written. Some of the more interesting changes found in this last set include: 

  * The XFS filesystem has gained support for shared extents — ranges of file data that can be shared between multiple owners — and a copy-on-write mechanism to manage modifications to those extents. That, in turn, allows XFS to support [`copy_file_range()`](<https://lwn.net/Articles/659523/>) along with other nice features like data deduplication. 

  * The NFS server now supports the NFS4.2 COPY operation, allowing file data to be copied without traveling to the client and back. 

  * The watchdog subsystem has a new "pretimeout" mechanism to allow the system to respond just prior to the expiration of a timer. Two new "governors" are provided; one simply prints a log message, while the other will panic the system in the hope of generating more useful information for debugging the problem. 

  * A set of [`EXPORT_SYMBOL()` improvements](<https://lwn.net/Articles/674303/>) has been merged. It is now possible to place export directives into assembly code, and the handling of exported symbols in library objects has been improved. One immediate practical result is that it is now possible to place all `EXPORT_SYMBOL()` directives next to the definition of the symbol that is being exported. At the moment, checksums (for use with `CONFIG_MODVERSIONS`) for assembly symbols are not generated; that [should be fixed](<https://lwn.net/Articles/703808/>) in the near future. 

  * The build system can now use "thin archives" for the creation of intermediate objects, rather than linking them with `ld -r`. A thin archive contains symbol information, but simply points to the component object files rather than making copies. The main purpose here seems to be to make the PowerPC build work more smoothly; see [this commit](<https://git.kernel.org/linus/a5967db9af51a84f5e181600954714a9e4c69f1f>) for some more information. 

  * The build system can also perform dead code and data elimination. This option is potentially hazardous, since, without some extra effort, the linker may see some needed code as being dead, but it can also reduce the resulting image size considerably. 

  * There is a new GCC plugin called "latent_entropy", which [comes from](<https://grsecurity.net/pipermail/grsecurity/2012-July/001093.html>) the grsecurity/PaX patch set. It will instrument the kernel in an attempt to collect randomness, especially during the early bootstrap process. 

  * New hardware support includes: Loongson 1C processors, Freescale "data patch acceleration architecture" hardware buffer and queue-management subsystems, and Imagination Technologies ASCII LCD displays. 

At this point the feature work is done; all that remains is to stabilize all that new code for the final 4.9 release. If all goes according to the usual schedule, that release can be expected on December 4 or 11.
