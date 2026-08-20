---
title: Rationalizing the ARM tree
url: https://lwn.net/Articles/439314/
date: "April 19, 2011"
category: "Architectures-Arm; Development model"
author: "By Jonathan Corbet April 19, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 19, 2011

The kernel's ARM architecture support is one of the fastest-moving parts of a project which, as a whole, is anything but slow. Recent [concerns](<https://lwn.net/Articles/437162/>) about the state of the code in the ARM tree threaten to slow things down considerably, though, with some developers now worrying in public that support for new platforms could be delayed indefinitely. The situation is probably not that grim, but some changes will certainly need to be made to get ARM development back on track. 

Top-level ARM maintainer Russell King recently [looked at the ARM patches in linux-next](<https://lwn.net/Articles/439324/>) and was not pleased with what he saw. About 75% of all the architecture-specific changes in linux-next were for the ARM architecture, and those changes add some 6,000 lines of new code. Some of this work is certainly justified by the fact that the appearance of new ARM-based processors and boards is a nearly daily event, but it is still problematic in an environment where there have been calls for the ARM code to shrink. So, Russell suggested: ""Please take a moment to consider how Linus will react to this at the next merge window."" 

As it turns out, relatively little consideration was required; Linus showed up and [told the ARM developers](<https://lwn.net/Articles/439326/>) what to expect: 

Hint for anybody on the arm list: look at the dirstat that rmk posted, and if your "arch/arm/{mach,plat}-xyzzy" shows up a lot, it's quite possible that I won't be pulling your tree unless the reason it shows up a lot is because it has a lot of code removed. 

People need to realize that the endless amounts of new pointless platform code is a problem, and since my only recourse is to say "if you don't seem to try to make an effort to fix it, I won't pull from you", that is what I'll eventually be doing. 

Exactly when I reach that point, I don't know. 

A while back, most of the ARM subplatform maintainers started managing their own trees and sending pull requests directly to Linus. It was a move that made some sense; the size and diversity of the ARM tree makes it hard for a single top-level maintainer to manage everything. But it has also led to a situation where there seems to be little overall control, and that leads to a lot of duplicated code. As Arnd Bergmann [put it](<https://lwn.net/Articles/439330/>): 

Right now, every subarchitecture in arm implements a number of drivers (irq, clocksource, gpio, pci, iommu, cpufreq, ...). These drivers are frequently copies of other existing ones with slight modifications or (worse) actually are written independently for the same IP blocks. In some cases, they are copies of drivers for stuff that is present in other architectures. 

The obvious solution to the problem is to pull more of the code out of the subplatforms, find the commonalities, and eliminate the duplications. It is widely understood that a determined effort along these lines could reduce the amount of code in the ARM tree considerably while simultaneously making it more generally useful and more maintainable. Some work along these lines has already begun; some examples include [Thomas Gleixner's work](<https://lwn.net/Articles/439176/>) to consolidate interrupt chip drivers, [Rafael Wysocki and Kevin Hilman's work](<https://lwn.net/Articles/439091/>) to unify some of the runtime power management code, and Sascha Hauer's ["sanitizing crazy clock data files" patch](<https://lwn.net/Articles/439332/>). 

Some of the ongoing work could benefit architectures beyond ARM as well. It has been observed, for example, that most GPIO drivers tend to look a lot alike. There are, after all, only so many ways that even the most imaginative hardware designers can come up with to control a wire with a maximum of two or three states. The kernel has an unbelievable number of GPIO drivers; if most of them could be reduced to declarations of which memory-mapped I/O bits need to be twiddled to read or change the state of the line, quite a bit of code could go away. 

There is also talk of reorganizing the ARM tree so that most drivers no longer live in subplatform-specific directories. Once all of the drivers of a specific type can be found in the same place, it will be much easier to find duplicates and abstract out common functionalities. 

All of this work takes time, though, and the next merge window is due to open in less than two months. Any work which is to be merged for 2.6.40 needs to be in a nearly-complete state by now; most of the work that satisfies that criterion will be business as usual: adding new platforms, boards, and drivers. Russell [worries](<https://lwn.net/Articles/439333/>) that this work is now unmergeable: 

Will we ever be able to put John's code in the kernel? Honestly, I have no idea. What I do know is that unless we start doing something to solve the problem we have today with the quantity of code under arch/arm _and_ the constant churn of that code, we will _never_ be able to add new platform support in any shape or form to the kernel. 

Russell has an occasional tendency toward drama that might cause readers to discount the above, but he's not alone in these worries. Mark Brown is [concerned](<https://lwn.net/Articles/439334/>) that ARM development will come to a halt for the next several months; he also has [expressed doubts](<https://lwn.net/Articles/439335/>) about the whole idea that the ARM tree must shrink before it can be allowed to grow again: 

What we're telling people to do is work on random improvements to more or less tangentially related code. This doesn't seem entirely reasonable and is going to be especially offputting for new contributors (like the people trying to submit new platforms, many of them will be new to mainline work) as it's a pretty big jump to start working on less familiar code when you're still trying to find your feet and worried about stepping on people's toes or breaking things, not to mention justifying your time to management. 

If these fears hold true, we could be looking at a situation where the kernel loses much of its momentum - both in support for new hardware and in getting more contributions from vendors. The costs of such an outcome could be quite high; it is not surprising that people are concerned. 

In the real world, though, such an ugly course of events seems unlikely. Nobody expects the ARM tree to be fixed by the 2.6.40 merge window; even Linus, for all his strongly-expressed opinions, is not so unreasonable. Indeed, he is currently [working on a patch to git](<https://lwn.net/Articles/439466/>) to make ARM cleanup work not look so bad in the statistics. What is needed in the near future is not a full solution; it's a clear signal that the ARM development community is working toward that solution. Some early cleanup work, some pushback against the worst offenses, and a plan for following releases should be enough to defer the Wrath Of Linus for another development cycle. As long as things continue to head in the right direction thereafter, it should be possible to keep adding support for new hardware. 

Observers may be tempted to view this whole episode as a black mark for the kernel development community. How can we run a professional development project if this kind of uncertainty can be cast over an entire architecture? What we are really seeing here, though, is an example of how the community tries to think for the long term. Cramming more ARM code into the kernel will make some current hardware work now, but, in the long term, nobody will be happy if the kernel collapses under its own weight. With luck, some pushback now will help to avoid much more significant problems some years down the line. Those of us who plan to still be working on (and using) Linux then will benefit from it.
