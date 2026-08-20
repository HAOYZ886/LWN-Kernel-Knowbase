---
title: "Kernel Summit 2005: Development process and quality assurance"
url: https://lwn.net/Articles/144281/
date: "July 20, 2005"
category: "Development model-Kernel quality"
author: 
---

From LWN's [2005 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2005/>).   
---  
The final session of the 2005 Kernel Summit was dedicated to the development process and what can be done to produce higher-quality kernels. There is a general sense of satisfaction with the "new" (one year old) development model, but also a sense that the quality of the resulting kernels is not quite what it could be. 

Andrew Morton started off the discussion by saying how he would like to see things go. Once the 2.6.13 kernel is released, he would like to merge all of the major subsystem trees within one week. The following week would be dedicated to big fixes, then the kernel would go into stabilization mode. There are currently two problems, according to Andrew: 

  * Many subsystems are waiting too long to merge their changes into the mainline, with the result that they miss many weeks of testing time. Getting those changes in sooner could help testers find some of the bugs which have been slipping through into the stable releases. 

  * The kernel developers are simply not fixing bugs, even after they have been found. According to Andrew, we have to try harder, to be more diligent. 

It was noted that the source management change did not help the situation; Linus stated that the kernel community lost three months as a result of that change. He also thought it was worth noting that the "dot releases" (2.6.x.y) are widely seen as being successful. 

To address the problem of patches not being merged early enough in the cycle, it was suggested that a deadline be imposed. One or two weeks after the start of a kernel cycle, only fixes would be accepted into the mainline. One way of enforcing this change would be to cease merging git repositories after that time, and only accept patches sent via mail. Git is not inherently a problem, but it does make it easy to slip large piles of code into the kernel. This step may not be taken, but it does seem likely that some sort of deadline for major merges will be imposed. 

Getting developers to fix bugs is a harder problem. There was some talk of ways to make bugzilla work better, but, in the end, it comes down to the developers taking the time to deal with the bugs in their parts of the kernel. Until that happens, kernel releases will likely continue to have more bugs than any of us would like.
