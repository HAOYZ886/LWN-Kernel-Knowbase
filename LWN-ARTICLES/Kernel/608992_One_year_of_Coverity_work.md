---
title: One year of Coverity work
url: https://lwn.net/Articles/608992/
date: "August 20, 2014"
category: "Development tools-Static analysis"
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

For the last year, Dave Jones has been working with the [Coverity scanner](<https://scan.coverity.com/>) to identify and fix possible bugs in the kernel. Like many developers, he worried that things could get worse over time; with all the code that is going into the kernel, there must be a lot of defects coming in as well. But the actual news turns out to be a bit better than that. 

What Dave feeds to Coverity is a "kitchen-sink build" with almost everything enabled. Where there is a configuration choice to be made, he tries to choose like a distributor would. The end result is that the kernel he builds has some 6,955 options selected. Running the scanner on this kernel takes several hours. Indeed, Dave has been keeping Coverity busy enough that the company eventually set up a dedicated server for kernel runs, enabling him to do two or three runs each day. 

When Dave ran a scan of the 3.11 kernel, he ended up with a "defect density" of 0.68 per thousand lines of code — somewhat above the company's "open-source average" of 0.59. The results since then look like this: 

> Kernel| Defect  
> density  
> ---|---  
> 3.11| 0.68  
> 3.12| 0.62  
> 3.13| 0.59  
> 3.14| 0.55  
> 3.15| 0.55  
> 3.16| 0.53  
  
The current defect density, after the closing of the 3.17 merge window, is 0.52. So things have consistently gotten better over time. 

He has put together [a ranking](<http://codemonkey.org.uk/2014/08/14/linux-kernel-coverity-scan-hot-areas/>) of the areas in the kernel with the most defects. At the top of the list is the staging tree. That, he said, is good; if any other subsystem were worse than staging, that would indicate a real problem. Most of the other entries in the list were in the driver tree, which is not surprising; that is where most of the code is. 

The biggest single issue raised by Coverity, he said, was dead code. Sometimes the warning makes sense, but it is not always correct. Some code will be unreachable due to configuration choices, for example. Second on the list was a failure to check return values; that, too, may or may not be a real bug, depending on the situation. The third item on the list — checking for a null pointer after the pointer has already been dereferenced — is clearly bad news and needs to be fixed. 

Also scary are the static buffer overrun errors. These can be "nasty," and there are too many of them, though the situation is getting better. These, too, are not always bugs; the networking layer, for example, performs [![\[Dave Jones\]](https://static.lwn.net/images/conf/2014/ks/DaveJones-sm.jpg)](<https://lwn.net/Articles/609000/>) tricks with its `skb` data structures that look like buffer overruns but in reality are not. Coverity also flags a large number of resource leaks; unsurprisingly, these are usually found in error paths. 

There are numerous other classes of potential errors. Some of them, such as "statement with no effect," tend to be harmless and may well be intentional. For example, assigning a variable to itself has no effect, but it was evidently an effective way of shutting down "possibly uninitialized" warnings from the compiler in the past. Others, like unchecked use of data from user space, can be more serious; in this case, most of the offending uses are behind capability checks and are thus not immediately exploitable. 

The good news, Dave said, is that there are now less than 50 use-after-free errors flagged in the kernel. Various other classes of "dumb" errors have been nearly eliminated from the kernel as well. Dave said he keeps a close eye on those; when one pops up in a new kernel, he tries to get it fixed quickly. 

Ted Ts'o asked how many of the issues flagged by Coverity are real bugs. Dave's feeling is that only a small minority of the reports indicate serious bugs. If one is interested in security issues, he said, the [Trinity](<https://lwn.net/Articles/536173/>) fuzz tester has found more of them than Coverity has. 

What about ARM coverage? The commercial Coverity product has it, but the free version for the open source community does not. In truth, Dave said, if the code can be made to compile on an x86 compiler, Coverity will scan it. So he has pondered approaches like commenting out all of the inline assembly in the ARM tree and scanning the result. That is a project for the future, though. 

Other developers can have a look at the Coverity results if they want to help to fix the reported problems. It's a matter of going to [scan.coverity.com](<http://scan.coverity.com>) and signing up for the "Linux" project. Then drop a note to Dave, and he will approve access to the results. As is usually the case, more eyes on these results should lead to a better kernel in the long run.
