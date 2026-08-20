---
title: "Link-time optimization for the kernel"
url: https://lwn.net/Articles/512548/
date: "August 21, 2012"
category: Build system; Optimization tools
author: "By Jonathan Corbet August 21, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 21, 2012

The kernel tends to place an upper limit on how quickly any given workload can run, so it is unsurprising that kernel developers are always on the lookout for ways to make the system go faster. Significant amounts of work can be put into optimizations that, on the surface, seem small. So when the opportunity comes to make the kernel go faster without the need to rewrite any performance-critical code paths, there will naturally be a fair amount of interest. Whether the "link-time optimization" (LTO) feature supported by recent versions of GCC is such an opportunity or not is yet to be proved, but Andi Kleen is determined to find out. 

The idea behind LTO is to examine the entire program after the individual files have been compiled and exploit any additional optimization opportunities that appear. The most significant of those opportunities appears to be the inlining of small functions across object files. The compiler can also be more aggressive about detecting and eliminating unused code and data. Under the hood, LTO works by dumping the compiler's intermediate representation (the "GIMPLE" code) into the resulting object file whenever a source file is compiled. The actual LTO stage is then carried out by loading all of the GIMPLE code into a single in-core image and rewriting the (presumably) further-optimized object code. 

The LTO feature first appeared in GCC 4.5, but it has only really started to become useful in the 4.7 release. It still has a number of limitations; one of those is that all of the object files involved must be compiled with the same set of command-line options. That limitation turns out to be a problem with the kernel, as will be seen below. 

Andi's [LTO patch set](<https://lwn.net/Articles/512335/>) weighs in at 74 changesets — not a small or unintrusive change. But it turns out that most of the changes have the same basic scope: ensuring that the compiler knows that specific symbols are needed even if they appear to be unused; that prevents the LTO stage from optimizing them away. For example, symbols exported to modules may not have any callers in the core kernel itself, but they need to be preserved for modules that may be loaded later. To that end, Andi's first patch defines a new attribute (`__visible`) used to mark such symbols; most of the remaining patches are dedicated to the addition of `__visible` attributes where they are needed. 

Beyond that, there is a small set of fixes for specific problems encountered when building kernels with LTO. It seems that functions with long argument lists [can get their arguments corrupted](<https://lwn.net/Articles/512685/>) if the functions are inlined during the LTO stage; avoiding that requires marking the functions `noinline`. Andi complains ""I wish there was a generic way to handle this. Seems like a ticking time bomb problem."" In general, he acknowledges the possibility that LTO may introduce new, optimization-related bugs into the kernel; finding all of those could be a challenge. 

Then there is the requirement that all files be built with the same set of options. Current kernels are not built that way; different options are used in different parts of the tree. In some places, this problem can be worked around by disabling specific optimizations that depend on different compiler flags than are used in the rest of the kernel. In others, though, features must simply be disabled to use LTO. These include the "modversions" feature (allowing kernel modules to be used with more than one kernel version) and the function tracer. Modversions seems to be fixable; getting ftrace to work may require changes to GCC, though. 

It is also necessary, of course, to change the build system to use the GCC LTO feature. As of this writing, one must have a current GCC release; it is also necessary to install a development version of the binutils package for LTO to work. Even a minimal kernel requires about 4GB of memory for the LTO pass; an "allyesconfig" build could require as much as 9GB. Given that, the use of 32-bit systems for LTO kernel builds is out of the question; it is still possible, of course, to build a 32-bit kernel on a 64-bit system. The build will also take between two and four times as long as it does without LTO. So developers are unlikely to make much use of LTO for their own work, but it might be of interest to distributors and others who are building production kernels. 

The fact that most people will not want to do LTO builds actually poses a bit of a problem. Given the potential for LTO to introduce subtle bugs, due either to optimization-related misunderstandings or simple bugs in the new LTO feature itself, widespread testing is clearly called for before LTO is used for production kernels. But if developers and testers are unwilling to do such heavyweight builds, that testing may be hard to come by. That will make it harder to achieve the level of confidence that will be needed before LTO-built kernels can be used in real-world settings. 

Given the above challenges, the size of the patch set, and the ongoing maintenance burden of keeping LTO working, one might well wonder if it is all worth it. And that comes down entirely to the numbers: how much faster does the kernel get when LTO is used? Hard numbers are not readily available at this time; the LTO patch set is new and there are still a lot of things to be fixed. Andi [reports](<https://lwn.net/Articles/512554/>) that runs of the "hackbench" benchmark gain about 5%, while kernel builds don't change much at all. Some networking benchmarks improve as much as 18%. There are also some unspecified "minor regressions." The numbers are rough, but Andi believes they are encouraging enough to justify further work; he also expects the LTO implementation in GCC to improve over time. 

Andi also suggests that, in the long term, LTO could help to improve the quality of the kernel code base by eliminating the need to put inline functions into include files. 

All told, this is a patch set in a very early stage of development; it seems unlikely to be proposed for merging into a near-term kernel, even as an experimental feature. In the longer term, though, it could lead to faster kernels; use of LTO in the kernel could also help to drive improvements in the GCC implementation that would benefit all projects. So it is an effort that is worth keeping an eye on.
