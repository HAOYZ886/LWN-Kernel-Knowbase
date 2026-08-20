---
title: "How old is our kernel?"
url: https://lwn.net/Articles/374574/
date: "February 17, 2010"
category: "Git; Releases-2.6.33"
author: "By Jonathan Corbet February 17, 2010"
---

By **Jonathan Corbet**  
February 17, 2010

April 2005 was a bit of a tense time in the kernel development community. The BitKeeper tool which had done so much to improve the development process had suddenly become unavailable, and it wasn't clear what would replace it. Then Linus appeared with a new system called git; the current epoch of kernel development can arguably be dated from then. The opening event of that epoch was commit 1da177e4, the changelog of which reads: 

Initial git repository build. I'm not bothering with the full history, even though we have it. We can create a separate "historical" git archive of that later if we want to, and in the meantime it's about 3.2GB when imported into git - space that would just make the early git days unnecessarily complicated, when we don't have a lot of good infrastructure for it. 

Let it rip! 

The community did, indeed, let it rip; some 180,000 changesets have been added to the repository since then. Typically hundreds of thousands of lines of code are changed with each three-month development cycle. A while back, your editor began to wonder how much of the kernel had actually been changed, and how much of our 2.6.33-to-be kernel dates back to 2.6.12-rc2, which was tagged at the opening of the git era? Was there anything left of the kernel we were building in early 2005? 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Answering this question is a simple matter of bashing out some ugly scripts and dedicating many hours of processing time. In essence, the "`git blame`" command can be used to generate an annotated version of a file which lists the last commit to change each line of code. Those commit IDs can be summed, then associated with major version releases. At the end of the process, one has a simple table showing the percentage of the current kernel code base which was created for each major release since 2.6.12. Here's what it looks like: 

> ![\[Pretty bar chart\]](https://static.lwn.net/images/ns/kernel/age/age.full-kernel.png)

In summary: just over ~~41%~~ 31% of the kernel tree dates back to 2.6.12, and has not been modified since then. Our kernel may be changing quickly, but parts of it have not changed at all for nearly five years. Since then, we see a steady stream of changes, with more recent kernels being more strongly represented than the older ones. That curve will partly be a result of the general increase in the rate of change over time; 2.6.13 had fewer than 4,000 commits, while 2.6.33 will have almost 11,000. Still, one has to wonder what happened with 2.6.20 (5,000 commits) to cause that release to represent less than 2% of the total code base. 

Much of the really old material is interspersed with newer lines in many files; comments and copyright notices, in particular, can go unchanged for a very long time. The 2.6.12 top-level makefile set `VERSION=2` and `PATCHLEVEL=6`, and those lines have not changed since; the next line (`SUBLEVEL=33`) was changed in December. 

There are interesting conclusions to be found at the upper end of the graph as well. Using this yardstick, 2.6.33 is the smallest development cycle we have seen in the last year, even though this cycle will have replaced some code added during the previous cycles. 4.2% of the code in 2.6.33 was last touched in the 2.6.33 cycle, while each of the previous four kernels (2.6.29 through 2.6.32) still represents more than 5.5% of the code to be shipped in 2.6.33. 

Another interesting exercise is to look for entire files which have not been touched in five years. Given the amount of general churn and API change which has happened over that time, files which have not changed at all have a good chance of being entirely unused. Here is [a full list of files which are untouched since 2.6.12](<https://lwn.net/Articles/374663/>) \- all 1062 of them. Some conclusions: 

  * Every kernel tarball carries around drivers/char/ChangeLog, which is mostly dedicated to documenting the mid-90's TTY exploits of Ted Ts'o. There is only one change since 1998, and that was in 2001. Files like this may be interesting from a historical point of view, but they have little relevance to current kernels. 

  * Unsurprisingly, the documentation directory contains a great deal of material which has not been updated in a long time. Much of it need not change; the means by which one configures an ISA Sound Blaster card is pretty much as it always was - assuming one can find such a card and an ISA bus to plug it into. Similarly, Klingon language support (Documentation/unicode.txt), Netwinder support, and such have not seen much development activity recently, so the documentation can be deemed to be current, if not particularly useful. All told, 41% of the documentation directory dates back to 2.6.12. There was a big surge of documentation work in 2.6.32; without that, a larger percentage of this subtree would look quite old. 

  * Some old interfaces haven't changed in a long time, resulting in a lot of static files in `include/`. `<linux/sort.h>` declares `sort()`, which is used in a number of places. `<include/fcdevice.h>` declares `alloc_fcdev()`, and includes a warning that ""This file will get merged with others RSN."" Much of the sunrpc interface has remained static for a long time as well. 

  * Ancient code abounds in the driver tree, though, perhaps unsurprisingly, old header files are much more common than old C files. The ISDN driver tree has been quite static. 

  * Much of `sound/oss` has not been touched for a long time and must be nicely filled with cobwebs by now; there hasn't been much of a reason to touch the OSS code for some time. 

  * `net/decnet/TODO` contains a ""quick list of things that need finishing off""; it, too, hasn't been changed in the git era. One wonders how the DECnet hackers are doing on that list... 

So which subsystem is the oldest? This plot looks at the kernel subsystems (as defined by top-level directories) and gives the percentage of 2.6.12 code in each: 

> ![\[Oldest subsystems\]](https://static.lwn.net/images/ns/kernel/age/age-subsystems-fixed.png)

The youngest subsystem, unsurprisingly, is `tools/`, which did not exist prior to 2.6.29. Among subsystems which _did_ exist in 2.6.12, the core kernel, with about 13% code dating from that release, is the newest. At the other end, the sound subsystem is more than 45% 2.6.12 code - the highest in the kernel. For those who are curious about the age distribution in specific subsystems, [this page](<https://lwn.net/Articles/374622/>) contains a chart for each. 

In summary: even in a code base which is evolving as rapidly as the kernel, there is a lot of code which has not been touched - even by coding style or white space fixes - in the last five years. Code stays around for a long time. 

(For those who would like to play with this kind of data, the scripts used have been folded into the gitdm repository at git://git.lwn.net/gitdm.git). 

**Note** : this article has been edited to fix an error which overstated the amount of 2.6.12 code remaining in the full kernel.
