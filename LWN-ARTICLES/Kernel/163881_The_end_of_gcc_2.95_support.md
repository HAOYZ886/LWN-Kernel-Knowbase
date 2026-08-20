---
title: The end of gcc 2.95 support
url: https://lwn.net/Articles/163881/
date: "December 13, 2005"
category: "Build system; Development tools-Compiler toolchain; GCC"
author: 
---

For years, otherwise useful kernel patches have been rejected because they use language features which are not supported by version 2.95 of the gcc compiler. The developers have been reluctant to remove support for this ancient version of gcc (released in 1999) because some not-so-old distributions used it, and because a couple of architectures required it. More importantly, however: gcc 2.95 simply runs faster than later versions. For a kernel hacker waiting for a build to complete, compilation speed can be far more important than additional language features or more highly optimized code generation. 

In the middle of the mutex conversation, however, it was pointed out that some of the alternatives under consideration would not work with 2.95. In response, Andrew Morton, the biggest defender of 2.95 compatibility, [threw in the towel](<https://lwn.net/Articles/163891/>). It seems that quite a few things in the kernel already fail to work with 2.95, and the situation is not getting better. So, says Andrew: 

It's time to give up on it and just drink more coffee or play more tetris or something, I'm afraid. 

He followed up with [a patch](<https://lwn.net/Articles/163893/>) officially removing gcc 2.95 compatibility from the kernel. A [suggestion](<https://lwn.net/Articles/163897/>) to drop gcc 3.0 quickly followed; the 3.0 release was never widely used, and it lacks some features that the kernel developers would like to use. Moving directly to 3.1 as the oldest supported gcc would make life easier without a whole lot of additional pain. 

Nothing has been merged into the mainline yet - and may not be until 2.6.16 opens. But the writing is clearly on the wall: anybody still trying to use these older compilers with current kernels will have to upgrade soon.
