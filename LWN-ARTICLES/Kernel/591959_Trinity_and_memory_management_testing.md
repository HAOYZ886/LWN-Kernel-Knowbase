---
title: Trinity and memory management testing
url: https://lwn.net/Articles/591959/
date: "March 26, 2014"
category: "Development tools-Trinity"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

The [Trinity](<http://codemonkey.org.uk/projects/trinity/>) tool is a system call fuzz-testing utility for the Linux kernel. By supplying random data to the kernel in a focused way, Trinity has managed to expose a large number of bugs over the years. Trinity maintainer Dave Jones addressed the memory management track at the 2014 Linux Storage, Filesystem, and Memory Management Summit to talk about how he is using the tool to turn up memory management bugs in particular. 

He started by describing an idea he heard from Al Viro: create a memory range with `mmap()`, unmap a single page in the middle of that range, then pass the result to various system calls and see what happens. As he described it, "all hell broke loose." Large numbers of bugs have been turned up, followed by "heroic efforts" in the memory management community to fix them. As a result of those efforts, he said, he is now unable to find any problems in the 3.14-rc7 kernel's memory management code, which is a good thing. 

On the other hand, Sasha Levin has been using Trinity to find bugs in the linux-next tree, where they are rather more plentiful. Those bugs often move into the mainline during the merge window. Bringing more stability to the code in linux-next before it is merged would be a worthwhile thing to do. 

In general, Dave said, Trinity is good at finding bugs in the dark corners of the kernel that nobody makes much use of. So areas like huge pages, page migration, and the `mbind()` system call have been fertile ground. In the case of `mbind()`, it turned out that all callers were going through a user-space library. That library did argument checking, so, naturally, the system call itself did not. The result was a predictable pile of bugs which have now been fixed. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Lots of parts of the memory management subsystem are, he said, simply not getting adequate testing now. Trinity helps in this area, but its memory management testing is still on the rudimentary side. He wants to develop it further; he plans to work on memory management fuzzing for much of the rest of the year. But, even now, Trinity is finding more bugs in the memory management code than can be dealt with. 

Huge pages generate lots of bug reports from Fedora users; they also are the source of lots of problems found by Trinity. Reproducing those bugs in any more standard setting is hard, though; many of them involve applications, like the Java runtime, that Dave is unfamiliar with and uninterested in learning more about. So, for now, transparent huge pages are simply turned off for his Trinity runs. 

Reproduction of crashes provoked by Trinity is an ongoing problem in general. The tool can log everything that it does, but the logging is, itself, an expensive operation that can change timings to the point that a lot of problems simply go away. Many crashes are also the result of corrupted internal state in the kernel; the sequence that causes the corruption may happen a long time before that corruption causes the kernel to crash. So establishing the cause of crashes can be difficult. 

Dave had a couple of requests for memory management developers. One was for anybody adding new flags to existing system calls; when that happens, he would like to get a note so that he can start testing calls with that flag. He'll often notice them in the patch stream anyway, but an explicit notification is more reliable. The other thing he would like is to see more developers running Trinity on their systems. It is trivial to set up, he said; so there is no real reason not to make use of it. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
