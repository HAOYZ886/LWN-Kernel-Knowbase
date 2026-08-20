---
title: Removing exported symbols in a stable kernel
url: https://lwn.net/Articles/125621/
date: "March 1, 2005"
category: "Modules-Exported symbols"
author: 
---

The kernel developers have set a long term goal: reduce the number of kernel symbols exported to modules. There is a general feeling that the module interface has gone out of control, and that modules are allowed to reach into too many parts of the core kernel. Additionally, there seems to be no reason for many exports; quite a few exported symbols are not used by any modules in the mainline kernel. So almost every 2.6.x release has unexported at least a handful of symbols, sometimes to the detriment of out-of-tree modules. 

It looked like more of the same when Adrian Bunk posted [a patch unexporting `do_settimeofday()`](<https://lwn.net/Articles/125622/>), which is not used by any mainline modules. There didn't seem to be any reason to allow modules to change the kernel's idea of what time it is, so the symbol could go. 

Andrew Morton has [drawn the line](<https://lwn.net/Articles/125623/>), however, on symbol removals. He now wants them to be marked as being deprecated (when used in a module), added to the feature removal schedule, and actually removed a year down the line. His position is: 

I don't see much point in playing these games. Deprecate it, pull it out next year, done. 

If this view sticks, it means that the days of abrupt disappearance of exported symbols are done. Symbols can still go away, but there will be some advance warning before it happens. Whether it will stick remains to be seen, however; there is a definite subset of kernel hackers who feel that there is no need to make life easier for out-of-tree modules. 

So what happened with the patch? It [turns out](<https://lwn.net/Articles/125636/>) that the ARM architecture has a number of out-of-tree real-time clock modules which need to be able to call `do_settimeofday()`. So Adrian withdrew the patch, and the symbol remains exported.
