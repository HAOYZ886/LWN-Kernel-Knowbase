---
title: Topics in live kernel patching
url: https://lwn.net/Articles/706327/
date: "November 14, 2016"
category: Live patching
author: "By Jonathan Corbet November 14, 2016 Linux Plumbers Conference"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 14, 2016

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Plumbers_Conference>)

Getting live-patching capabilities into the mainline kernel has been a multi-year process. Basic patching support was merged for the 4.0 release, but further work has been [stalled](<https://lwn.net/Articles/634649/>) over disagreements on how the consistency model — the code ensuring that a patch is safe to apply to a running kernel — should work. The addition of [kernel stack validation](<https://lwn.net/Articles/658333/>) has addressed the biggest of the objections, so, arguably, it is time to move forward. At the [2016 Linux Plumbers Conference](<https://www.linuxplumbersconf.org/2016/>), developers working on live patching got together to discuss current challenges and future directions. 

This article is not an attempt at a comprehensive summary for a half-day of fast-moving discussion; instead, the goal is to cover some of the more interesting topics as a way of showing that challenges that the live-patching developers must overcome and how they plan to get there. 

#### Unhelpful optimizations

A smart optimizing compiler is necessary for anybody who wants to get reasonable performance from their code, but problems arise if the compiler gets too smart. Developers working with concurrency in the kernel have had to worry about aggressive optimizations for some time; according to Miroslav Benes, live-patching developers have to worry as well. Compiler optimizations can change how code is compiled in subtle ways that can lead to mayhem when a patch is applied. 

Starting with the easiest problems before moving on to the trickier ones, Benes noted that the automatic inlining of functions can be a problem if an inlined function must be patched. In that case, the solution is relatively easy; all callers of the function must be changed in the resulting live [![\[Miroslav Benes\]](https://static.lwn.net/images/conf/2016/lpc/MiroslavBenes-sm.jpg)](<https://lwn.net/Articles/706329/>) patch. The `-fpartial-inlining` option can complicate things by only inlining portions of functions, but it doesn't change the basic nature of the problem. 

The `-fipa-src` option is a bit more subtle, in that it can lead to the removal of unused function parameters or change the way in which parameters are passed into a function. In other words, it changes the ABI of the function in response to its observations on how the function works. A live patch to that function could change how this optimization operates, leading to a surprising change in the ABI. The good news here is that, when this happens, GCC will change the name of the compiled function, so the broken ABI is immediately obvious. But this can prevent the direct patching of a buggy function; callers must be patched as well. 

Code compiled with `-fipa-pure-const` may change in response to how a function operates; if a function is seen as not accessing memory, the compiler will make assumptions about the state of memory before and after calling it. If a patch changes the function's behavior, those assumptions may no longer hold; once again, it will be necessary to patch callers when this happens. 

An "even crazier" option is `-fipa-icf`, which performs identical code folding. It can cause a function to be entirely replaced by an equivalent found elsewhere in the code, and it can be hard to detect that this change has happened. Code folding is also a problem for the kernel's stack unwinder. Other types of code elimination can happen if GCC thinks that a specific global variable won't change over a given function call. If the function is patched to now change the global variable, the calling code may well be incorrect. This sort of change, too, is hard to detect; it would be nice, he said, to have a GCC option to ask it to create a log of the optimizations it has done. 

Perhaps the scariest option is `-fipa-ra`, which tracks the registers used by called functions and avoids saving those that will not be changed. A patch to the called function could easily cause it to use a new register, leading to data corruption in the calling functions and a likely significant reduction in the continuous uptime that live-patching users were hoping to enjoy. This optimization is hard to detect; it can be thought of as an ABI change for the called function, but no name changes are made. This one, he said, is "not good news." For now, this optimization is disabled by GCC when `-pg` is turned on, and the Ftrace subsystem, needed for live patching, needs `-pg`. But there is no inherent reason why those two options need to be incompatible, so this behavior could change at any time. 

This list, Miroslav said, is only a small subset of the optimizations that can create problems for live patches. As compiler developers pursue increasingly aggressive optimizations, this problem is only going to get worse. 

#### Patch building

The kernel has a standard way to apply a live patch, but there is not, yet, any sort of mainlined mechanism for the creation of live patches. Josh Poimboeuf gave a brief summary of the patch-creation tools out there with an eye toward picking one for upstream. 

The first of these is [kpatch-build](<https://github.com/dynup/kpatch/tree/master/kpatch-build>). It works by building the kernel both with and without the patch applied, then does a binary diff to see which functions changed. All of the changed functions are then extracted and packaged up into a "Frankenstein kernel [![\[Josh Poimboeuf\]](https://static.lwn.net/images/conf/2016/lpc/JoshPoimboeuf-sm.jpg)](<https://lwn.net/Articles/706330/>) module" that is shipped with the live patch. It is a powerful system, he said, with a number of advantages, including the fact that it automatically deals with most of the optimization issues mentioned in the previous talk. 

On the other hand, kpatch-build is quite complex. It has to know about all of the special sections used by the kernel, and it has problems with certain kinds of changes. It only works on the x86_64 architecture at the moment; all of those special sections differ across architectures, so turning it into a multi-architecture tool will not be easy. And, he said, kpatch-build is brittle and a maintenance nightmare. 

An alternative is to just use the regular kernel build system and its module-building infrastructure. The changed function is copied and pasted into a new module, some boilerplate is added to register the function with the live-patching API, and the job is done. It's easy, but has its own problems; in particular, this module is unable to access non-exported symbols, which the patched function may need to do. This problem can be worked around by using `kallsyms_lookup_name()`, but this solution is error-prone, slow, and "yucky." 

The third alternative is new; indeed, he [posted the proposal](<https://lwn.net/Articles/704836/>) the week before the conference. This alternative uses the copy-and-paste approach, but adds an API and a postprocessing tool that allows the generated module to gain access to non-exported symbols. The code works now, though there are a number of possible improvements, including automating the process of attaching to non-exported symbols and detecting interference from compiler optimizations. 

In the brief discussion at the end of the talk, it became clear that there were not a lot of concerns about the new tooling, so that is the direction things seem likely to go. 

#### Module dependencies

Live patches can make changes to loadable modules, which leads to an interesting question: what happens if the module isn't present in the system when a patch is applied, but is loaded afterward? The live-patching code currently has some complicated infrastructure designed to detect this case and apply patches to modules as they are loaded. Jessica Yu, who has just taken over as the maintainer of the loadable module subsystem in the kernel, talked briefly about changing this mechanism to require that alle affected modules be loaded before a live patch is applied. 

Live patches are, themselves, loadable modules. Allowing a patch module to be loaded before any modules it affects requires carrying a fair amount of information and complex infrastructure, and it circumvents the normal [![\[Jessica Yu\]](https://static.lwn.net/images/conf/2016/lpc/JessicaYu-sm.jpg)](<https://lwn.net/Articles/706331/>) module dependency mechanism. As a result, there is a fair amount of code duplication, including a reimplementation of much of the module loader in the live-patching code. 

There are a couple of ways that things could be changed. One would be to simply require that all modules being patched be loaded before the patch itself is loaded. That would work, but it forces the loading of code into the kernel that is unneeded and may never be used on any given site. The alternative would be to split the live-patch module into multiple pieces, each of which applies a patch to a single kernel module. Then, only the pieces that are relevant to any given running system need to be loaded. 

Making this change would simplify the live-patching code and reduce code duplication, but there's a problem: there isn't an easy way to force a necessary patch module to be loaded when a module needing patching is loaded. The `depmod` tool just doesn't recognize that sort of dependency. FreeBSD has a nice `MODULE_DEPEND()` macro, but Linux has never needed that infrastructure. 

Splitting the patch module, it turns out, could be problematic for any sort of wide-ranging change. [CVE-2016-7097](<https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-7097>) was mentioned as an example; it included a virtual filesystem layer API change that had to be propagated to all filesystems. If it were to be split apart, the result would be a long list of modules that would need to be loaded to apply the patch. 

There was a lively discussion on whether the rules concerning live patches for modules should be changed, much of it focused on a question asked by Steve Rostedt: if a module isn't present in the kernel, why not just fix it on disk rather than lurking in the kernel, waiting to patch it should it ever be loaded? Jiri Kosina replied that replacing on-disk modules would be hard from a distributor's point of view; it would introduce modules that no longer belong to the kernel package. Live patches can also be disabled; in that case, modified modules would have to be somehow restored. Some consistency models can also create trouble; it is possible to have both the pre-patch and post-patch code live and running in the kernel at the same time. So it's not obvious that fixing things on-disk is a workable solution, though Rostedt was adamant that it should be considered. 

As the discussion wound down, it became fairly clear that the consensus was against changing how module dependencies work in live patching. The mechanism that the kernel has now, in the end, works well enough; it looks like it will not be going away anytime soon. 

#### Other topics

Petr Mladek talked about the problems that come with modifying data structures in live patches. One has to start by locating the affected data and accesses; that is easy with a global variable, harder for data stored in multiple lists, and nearly impossible for uses that have been hidden via casts. Switching to the new values must be done carefully, once all of the code is in a position to handle them. Many techniques, such as the use of shadow structures to add data to existing structures, suffer from performance problems. And the problem of reverting a live patch gets that much harder when data changes have been made. 

Miroslav Benes returned to talk about the problems associated with patching functions in the scheduler. It turns out that `schedule()` is a tricky function to work with, since it returns with a different stack than the one it was called with. This caused difficulties with a 2015 live patch fixing a security problem with x86 local descriptor table handling. 

He outlined a solution to the problem involving putting the instruction pointer into the context that is saved when a context switch is made. That information can be used after a patch is applied to ensure that the version of `schedule()` that restores a given context is the same as the one that saved it. The solution is workable, but it's not clear that it matters that much; security issues in `schedule()` are rare and there may not be a need to apply another live patch to it anytime soon. 

Jiri Kosina led a brief session on future work. The consistency model, as noted above, has been blocked for a long time on related issues. Now that the stack-validation work has been done and, hopefully, kernel-stack tracebacks can be trusted, it should be possible for that work to continue. There will likely be new proposals in that area soon. 

In particular, the [hybrid consistency model](<https://lwn.net/Articles/685464/>) is likely to move forward. It should be reliable now that the stack traces are correct, but there is an associated problem: it requires a kernel built with frame pointers, and that has a significant performance cost — on the order of 10%. Nobody seems to know why turning on frame pointers hurts that badly; simply compiling the kernel with one register disabled does not have the same effect. Mel Gorman is evidently doing some benchmarking to try to track this problem down. 

Kosina said that he is currently working on a port to the arm64 architecture. Beyond that, he said, there's not much point about worrying about other possible developments in live patching. The hybrid consistency model is likely to keep the group entertained for quite some time. 

The microconference closed with a wide-ranging talk from Balbir Singh; much of it was taken up by low-level PowerPC details that are probably of relatively little interest to those outside the room. He did raise a few larger questions, though. One of those is expanding live patching to user-space code as well; there are, evidently, users who are interested in that capability. 

He asked: what are the benefits of using live patching rather than performing a live cluster update? If a cluster can be taken down and upgraded one machine at a time, there is no real need for a live-patching infrastructure. We don't all run clusters, but users whose uptime needs make them consider live patching maybe should be using clusters. 

His last question had to do with rootkits; a live-patching mechanism is obviously a nice tool by which code can be injected into a running kernel. Kosina said that he doesn't really understand what the worry is in this regard. A live patch is just a module; if an attacker can load modules into the kernel, the game is already over. But, Singh said, there could be a vulnerability in the live patch itself; this is something that has happened to other vendors in the past. Live patching is meant to be a way to quickly close security problems, but, like any other sort of patch, it always runs the risk of introducing new vulnerabilities of its own. 

[Thanks to LWN subscribers for supporting our travel to the event.]
