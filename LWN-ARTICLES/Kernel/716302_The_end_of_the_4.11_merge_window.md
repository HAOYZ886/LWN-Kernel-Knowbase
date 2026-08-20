---
title: The end of the 4.11 merge window
url: https://lwn.net/Articles/716302/
date: "March 7, 2017"
category: "Releases-4.11"
author: "By Jonathan Corbet March 7, 2017"
---

By **Jonathan Corbet**  
March 7, 2017

By the time Linus Torvalds released [4.11-rc1](<https://lwn.net/Articles/716293/>) and closed the merge window for this development cycle, 10,960 non-merge commits had been pulled into the mainline repository. Just over 800 of those were pulled after the writing of [last week's summary](<https://lwn.net/Articles/715781/>). Thus, there is a relatively small set of patches to cover here, but a couple of the more significant changes were saved for last. 

  * The long-awaited [`statx()` system call](<https://lwn.net/Articles/707602/>) has been merged. This new version of `stat()`, which has been in the works since 2010, adds a number of new features and efficiency improvements. See [the commit changelog](<https://git.kernel.org/linus/a528d35e8bfcc521d7cb70aaf03e1bd296c8493f>) for details. 

  * New hardware support includes: Renesas R-Car Gen3 thermal sensors, ZTE zx2967 SoC thermal sensors, and QLogic QEDF 25/40/100Gb FCoE initiators 

  * As predicted, the large [`sched.h` refactoring work](<https://lwn.net/Articles/713890/>) has been merged. In theory, all kernel code that needs to be fixed in responses to these changes has indeed been fixed, but there may still be a few loose ends here and there. 

The 4.11 kernel has now entered the stabilization period. If things go according to the normal schedule, the final 4.11 release can be expected on April 16 or 23.
