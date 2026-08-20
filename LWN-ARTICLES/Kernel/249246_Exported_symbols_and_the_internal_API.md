---
title: Exported symbols and the internal API
url: https://lwn.net/Articles/249246/
date: "September 11, 2007"
category: "Development model-Loadable modules; Modules-Exported symbols"
author: "By Jonathan Corbet September 11, 2007"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 11, 2007

Loadable kernel modules do not automatically have access to all symbols (functions and variables) defined in the kernel. In fact, access is limited to those symbols which have been explicitly exported for modular use. The idea behind this whitelist-like policy is that it helps the kernel developers to keep the module interface under control, limiting the ability of modules to dig into parts of the kernel where they are not welcome. The practice turns out to be a little more messy: current kernels have over 16,000 `EXPORT_SYMBOL()` declarations sprinkled around the source. 

Unsurprisingly, there are developers who would like to reduce the number of exported symbols. It is often the case that, once a symbol can be shown to have no users among in-tree modules, it will be removed altogether. But there is not universal agreement on just how this process should be handled; as a result, we see occasional debates on how stable the modular API should actually be and what provisions should be made for out-of-tree code. 

Adrian Bunk recently posted [a patch to unexport `sys_open()` and `sys_read()`](<https://lwn.net/Articles/249256/>). These symbols (which implement the `open()` and `read()` system calls) have been on the hit-list for a long time. It is easy to make catastrophic mistakes when using them from kernel space, and there is almost no situation where opening and reading files from within the kernel is considered to be the right thing to do. But removing the exports has always proved hard, until now - there have always been stubborn in-tree users which have kept the export around. 

The final holdout in 2.6.23 is the wavefront sound driver which uses `sys_open()` and `sys_read()` to obtain firmware to load into the device. The kernel has had a proper API for dealing with firmware loads for years, so no driver should be trying to read firmware directly from files itself. The current ALSA development tree contains a patch for the wavefront driver which makes it use the firmware API; once that patch is merged, there will be no more in-tree users of those symbols. Adrian, forever on the lookout for things to remove from the kernel, noticed this fact and promptly sent in a patch. 

Andrew Morton's [response](<https://lwn.net/Articles/249261/>) went like this: 

But I think it is better to give people some warning when we're planning on breaking out-of-tree things. I do occasionally receive reports of "hey, the X driver which I get from Y doesn't work any more". Often it's open-source stuff, too. I see no point in irritating our users more than we need to. 

Andrew would like to have the symbols marked with `EXPORT_UNUSED_SYMBOL()` for one development cycle so that maintainers of out-of-tree code can get the resulting warning message and fix their code in response. It quickly became clear that he is in a minority among the developers on this issue. Adrian was particularly upset, complaining that other developers are allowed to make no-warning changes which break almost every module in existence while his patch, which affects very few modules, must go through a special process. He [says](<https://lwn.net/Articles/249263/>): 

Andrew, please define API rules, IOW rules for addition, removal and changing of exported code, that are valid for *everyone* or go to hell with your EXPORT_UNUSED_SYMBOL. 

Christoph Hellwig also responded strongly, leading to [this amusing (but not for the easily offended) exchange](<https://lwn.net/Articles/249265/>). Calmer voices made a few arguments against the warning period: 

  * These symbols have been on the chopping block for a long time, and most out-of-tree module authors should have figured that out by now. It is worth noting, though, that the feature removal schedule in the kernel documentation says nothing about `sys_open()` and `sys_read()`. 

  * In this sort of situation warnings are almost entirely ineffective. Users tend not to see them at all, and they do not report them in any case. [According to Alan Cox](<https://lwn.net/Articles/249267/>): ""Short of using their sound card to scream 'Next release you are screwed' they won't notice (and if you the sound card trick they'll think they got rooted....)"" 

  * Keeping unused symbols around bloats the kernel and increases the load on developers who must remember to remove them in a future release. 

Andrew does not appear willing to budge on the issue, though. He [does not want to unnecessarily upset users](<https://lwn.net/Articles/249268/>) who use out-of-tree modules: 

Fact is, people use external modules. To get their machines working correctly, to get their work done, to do stuff they want done. 

Many of these people are non-programmers. So when they download a new kernel and find that the module which they use doesn't work because of something which we've done, they get pissed off, and we lose a tester. This has happened many times. 

To avoid this problem, he wants exported symbols targeted for removal to marked with `EXPORT_UNUSED_SYMBOL()` (or `EXPORT_UNUSED_SYMBOL_GPL()`) for one development cycle. The exports should be marked with a comment noting when the export should be removed altogether. Each release cycle would include a quick grep to find the symbols which are now due to be removed for real. He concludes: 

Total cost of this effort: maybe ten developer minutes per release, and a few tens of additional bytes in the released vmlinux. I think that for a few additional testers and a few less-pissed-off users (nothing to do with developers), this cost is justified. That's all. 

Elsewhere he has noted that, if a warning is sufficiently widespread, somebody, somewhere, will act on it. One gets the sense that he has not convinced a whole lot of developers that this position is right. But Andrew is in a position to enforce it and most of the others seem to think that, in the end, it's easier to just go along with what he wants in this case. The end result is the same, it just takes a little longer.
