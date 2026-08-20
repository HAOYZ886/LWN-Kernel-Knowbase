---
title: Bringing kgdb into 2.6
url: https://lwn.net/Articles/70465/
date: "February 10, 2004"
category: Debugging; kgdb
author: 
---

The kernel development community has long been divided over the topic of interactive debuggers. Many hackers find debuggers to be an indispensable part of their development toolkits. Others claim that debuggers lead people to fix symptoms rather than problems; rather than use such a crutch, these people say, it is better to truly understand the code. Once you have "become one" with the code, finding bugs is not that hard. 

The latter view is held by Linus Torvalds, who [explained](<http://lwn.net/2000/0914/a/lt-debugger.php3>) his approach in very clear terms back in 2000: 

You can use a kernel debugger if you want to, and I won't give you the cold shoulder because you have "sullied" yourself. But I'm not going to help you use one, and I would frankly prefer people not to use kernel debuggers that much. So I don't make it part of the standard distribution, and if the existing debuggers aren't very well known I won't shed a tear over it. 

The end result is that there has never been support for interactive debuggers in the mainline kernel - at least, for the more popular architectures. 

The 2.6 kernel is now Andrew Morton's turf, however, and Andrew is more open to the value of debugging tools. In fact, he has carried a version of the kgdb patch in his -mm tree for a long time. Might Andrew merge kgdb into the 2.6 kernel at some point? 

The [answer](<https://lwn.net/Articles/70470/>) from Andrew seems to be "maybe": 

I wouldn't support inclusion of i386 kgdb until it has had a lot of cleanup, possible de-featuritisification and some thought has been applied to splitting it into arch and generic bits. It's quite a lot of work. 

In other words, there is no disagreement with the idea of merging kgdb, but the code needs some work first. Problems include a large number of `#ifdef`s, and the fact that the patch is relatively intrusive, touching many files. There are also objections to how the debugger works with the virtual memory subsystem, especially for the i386 architecture. All of these problems are probably solvable, given enough development time. The interest in a mainline kgdb is probably high enough that the cleanup work will happen, and kgdb may well be merged; a kgdb CVS repository [has been established](<https://lwn.net/Articles/70676/>) for those interested in this effort. An eventual merge into 2.6 seems unlikely to carry forward into 2.7, however.
