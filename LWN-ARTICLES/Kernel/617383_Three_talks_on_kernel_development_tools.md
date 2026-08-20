---
title: Three talks on kernel development tools
url: https://lwn.net/Articles/617383/
date: "October 22, 2014"
category: "Development tools-Coccinelle; Development tools-Undertaker"
author: "By Jonathan Corbet October 22, 2014 Linux
Plumbers Conference"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 22, 2014

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2014-Linux_Plumbers_Conference>)

By design, the Linux Plumbers Conference tries to emphasize discussions over presentations. The development tools microconference ran against that stream, though, consisting mostly of a series of presentations on interesting applications of tools to the kernel. Some of the tools are more immediately useful than others; here we'll focus on a few of the tools that can help developers now. 

#### Using Coccinelle for backports

The [kernel backports project](<https://backports.wiki.kernel.org/index.php/Main_Page>) works to provide drivers from leading-edge kernels backported to older stable kernel releases. The idea is to provide the best hardware support on a platform that, as a whole, is relatively stable and bug-free. It is not surprising that enterprise Linux distributors are interested in this work, but others are as well. 

The project currently has three core developers working to backport about 800 drivers to older kernels. Needless to say, that is quite a bit of work; Luis Rodriguez talked about how the [Coccinelle](<http://coccinelle.lip6.fr/>) tool can be used to make the problem tractable for a relatively small group of developers. (See [this article](<https://lwn.net/Articles/315686/>) for an introduction to Coccinelle and its capabilities). 

Obviously, some amount of compromise is needed to scale up to 800 backported drivers. Actually testing all of those drivers is one of the first things to go; the developers lack both the time and the hardware to do that testing. So all they can do is promise that the backported drivers will compile. Drivers are backported as far as the 3.0 stable kernel; the [![\[Luis Rodriguez\]](https://static.lwn.net/images/conf/2014/lce-lpc/LuisRodriguez-sm.jpg)](<https://lwn.net/Articles/617440/>) group used to offer backports into the 2.6 series, but interest in those has waned. Various types of drivers are not even considered for backporting; the DRM subsystem, for example, is complex, and there is not enough interest on the distributor side to justify the work. So most of the backported drivers are in subsystems like Ethernet, wireless, Bluetooth, NFS, Video4Linux, and so on. 

The backporting project, Luis said, serves as a motivation to developers to get their code upstream. Once a driver is in the mainline kernel, the backporting effort is available for free. 

Backporting is a bit of an unusual use case for Coccinelle, which is normally used to evolve the kernel forward. Most developers will occasionally use it to make a forward change in specific parts of the kernel. The backporting project, instead, uses it every day against all of the drivers on its list. 

Backporting has traditionally been done, Luis said, by sprinkling `#ifdef`s throughout the code. That is "backporting on drugs"; it leads to patches that are long and hard to review. And the same tiresome work has to be done for every driver on the list. There must be a better way. 

The first step in that better way is to move as much backporting-related code as possible into header files. If, for example, a function prototype has been changed, the relevant header will contain a new version of the function (under a different name) that works with older code. The various call sites can then be fixed up to use the new function, with no `#ifdef` lines in the code itself. For the easier changes, Luis said, Coccinelle can apply this kind of update with no trouble at all. 

What about harder changes? An example Luis raised was [threaded interrupt handlers](<https://lwn.net/Articles/302043/>); how does one port code using threaded handlers back to kernels that do not provide that capability? The answer in this case was to backport threaded handlers in a separate module and add a `compat_request_threaded_irq()` function for backported drivers. The hard part, though, was figuring out where to put the needed private data pointer. A tricky Coccinelle semantic patch was written to figure out which data structure each driver was using to represent its device, then add the new pointer to that structure. With that in place, the backport can be done in an automated mode for all drivers. 

In summary, Luis said, using Coccinelle for backporting makes the effort easier and more consistent; with Coccinelle, the team is able to backport far more drivers. By making one Coccinelle rule for each relevant evolution of the kernel, the team can keep up with the kernel's changes and get, for all practical purposes, automatic backports of a wide range of drivers. According to Luis, the effort is sufficiently worthwhile that, arguably, the semantic patches used for backporting should be kept in the upstream kernel — though nobody has proposed a patch to make that happen yet. 

#### Watch out for the undertaker

Kernel developers try to minimize the use of `#ifdef` constructs, but there are still massive numbers of them in the kernel source tree. The interesting thing, according to Valentin Rothberg, is that a lot of these blocks do not actually make sense. In many cases, there is no possible combination of kernel configuration choices that can cause a block of code to be compiled; that block is thus dead code. In others, code will be compiled in all cases; Valentin called such code "undead." 

Dead and undead code can come about for a variety of reasons. Misspelled `CONFIG_` symbol names are one obvious source; an unknown identifier simply evaluates to false rather than raising an error. Sometimes configuration symbols are not truly independent; imagine a `CONFIG_BAR` that can only be set if `CONFIG_FOO` is set first. If you then see code like: 

```
#ifdef CONFIG_BAR
          /* ... */
        #ifndef CONFIG_FOO
          /* this code is dead */
        #endif
        #endif
```

you know that the block of code in the middle will never be compiled. About 25% of all dead/undead code blocks in the kernel are caused by this kind of logic error. All told, Valentin found over 1000 dead or undead blocks in the 2.6.29 kernel; that number has dropped to about 600 in 3.17 — an improvement, but still too many. These `#ifdef` blocks are intentionally conditional; dead or undead blocks go against that intention. Sometimes they hide bugs as well; Valentin found a memory leak bug caused by a misspelling of `CONFIG_HOTPLUG_CPU`. 

The solution to this problem is to make use of Valetin's `undertaker-checkpatch` tool. It will examine a patch, note changes to `#ifdef` blocks, and flag any dead or undead code that it finds. Patches checked this way should not add more `#ifdef`-related problems to the kernel. Undertaker can be found at [this web page](<https://vamos.informatik.uni-erlangen.de/trac/undertaker>). 

Your editor asked why this work, which seems valuable, wasn't simply integrated into `checkpatch.pl` and merged upstream. There are a few problems with that idea, it seems, starting with the fact that `undertaker-checkpatch` is written in C++. It also takes a few minutes to run, something that might not be welcome in a typical kernel developer's workflow. In the end, Valentin said, it is a research project; the developers lack the time and motivation to turn it into production code. 

Thus it seems like `undertaker-checkpatch` might suffer the fate of many academic projects. But there is still one way that this tool could be put to use in the kernel development community: integrate it into Fengguang Wu's zero-day testing system. Then developers would receive a polite email when they add `#ifdef`-related problems to the kernel. That, Valentin said, is his preferred route toward making this tool more useful for the kernel community. 

#### Vampyr

A project closely related to Undertaker is Vampyr, which is being worked on by the same group of developers. Stefan Hengelein described the fundamental problem that Vampyr seeks to address: the fact that most kernel patches are only tested against one or two kernel configurations. But the actual configuration space for the kernel is huge, and it is often not hard to find combinations of configuration parameters that do not work as intended. 

So, Stefan said, patches need to be reviewed with all of the relevant configuration options in mind. That is hard to do, requiring a lot of brainpower. Current kernels have nearly 14,000 configuration options; it [![\[Stefan Hengelein\]](https://static.lwn.net/images/conf/2014/lce-lpc/StefanHengelein-sm.jpg)](<https://lwn.net/Articles/617442/>) is questionable whether any amount of available brainpower is up to looking at code with all of the possible combinations of that many options in mind. 

The alternative is the Vampyr tool, which is designed to create a "maximizing set" of kernel configurations for a given patch. It searches through the configuration space, finding combinations that result in different code being compiled. The result is typically a handful of configurations, each of which can then be used for build tests. 

Using Vampyr, the developers have manage to unearth a lot of warnings and some outright errors. The x86 code generates 15% more warnings when Vampyr is used, while MIPS increases by 58%. With the ARM architecture, use of Vampyr nearly doubled the number of warnings, and resulted in the identification of 91 confirmed bugs. The situation has improved in later kernels, Stefan said, especially in the x86 code — other architectures have not improved that much. 

The code is available under GPLv3, he said; it can be obtained from the Undertaker web site. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Linux Plumbers Conference.]
