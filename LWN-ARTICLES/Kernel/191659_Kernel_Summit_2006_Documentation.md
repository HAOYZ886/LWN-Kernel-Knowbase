---
title: "Kernel Summit 2006: Documentation"
url: https://lwn.net/Articles/191659/
date: "July 18, 2006"
category: Documentation
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
The final session on Monday was concerned with documentation. This discussion was led by Michael Kerrisk, Randy Dunlap, and Jonathan Corbet. Your editor started off with a survey of internal kernel API documentation. The in-kernel documentation directory, shipped with the source code, contains much good stuff, but it is disorganized and unmaintained. Some of the treasures to be found there include details on running Java binaries under the 1.3 kernel and a discussion of the trials and tribulations of putting Linux on systems with more than 16MB of memory. The latter states: 

There are some buggy motherboards which cannot properly deal with the memory above 16MB. Consider exchanging your motherboard. 

Linus stole your editor's punchline by noting that this file clearly contains advice which is still good, so it should be retained. 

Other sources of kernel API documentation include books - all of which are currently obsolete. There is also a fair amount of information available on web sites, including this one. A few questions were presented for discussion, but what the developers really wanted to talk about was the tools used to generate the KernelDoc documents. These tools are effective, but very slow, and they lack the ability to express everything of interest about a given interface. Your editor got the impression that, if the KernelDoc tools were improved, much of the documentation problem would simply go away. Who knew it was so easy? 

The second half of the session was led by Michael Kerrisk, who has been putting a great deal of time into improving the kernel man pages distribution. These pages are the primary documentation of the kernel user-space API, and they have seen a relatively low level of attention for some time. The networking pages, for example, cover the 2.2 version of the interface. 

The man-pages distribution contains over 800 pages. Despite the time that Michael has put in, he is having a hard time keeping up with the changes being made by the kernel developers. The writing work is hard enough on its own, but, even before then, Michael must watch the patch stream and notice that API changes have been made. It would be nice if there were a better way. 

Michael talked about various approaches to improving the maintenance of the man pages. These include getting other maintainers to help out (by, at a minimum, calling attention to API changes), paying a writer to do the work, or requiring kernel developers to write man pages for any APIs they add. While the last one might seem like a good idea, Linus made the claim that "a lot of kernel developers have huge problems communicating with humans." Forcing them to write documentation might not be in anybody's best interest. 

So the creation of kernel documentation is likely to remain a task performed by a relatively small group of people. OSDL is currently considering a proposal to fund a writer for a fixed period; if that proposal goes through, we may see kernel documentation coming out at a higher rate.
