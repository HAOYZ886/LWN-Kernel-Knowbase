---
title: Device trees as ABI
url: https://lwn.net/Articles/561462/
date: "July 30, 2013"
category: "Development model-User-space ABI; Device tree"
author: "By Jonathan Corbet July 30, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 30, 2013

[Last week's device tree article](<https://lwn.net/Articles/560523/>) introduced the ongoing discussion on the status of device tree maintainership in the kernel and how things needed to change. Since then, the discussion has only intensified as more developers consider the issues, especially with regard to the stability of the device tree interface. While it seems clear that most (but not all) participants believe that device tree bindings should be treated like any other user-space ABI exported by the kernel, it is also clear that they are not treated in this way currently. Those seeking to change this situation will have a number of obstacles to overcome. 

Device tree bindings are a specification of how the hardware is described to the kernel in the device tree data structure. If they change in incompatible ways, users may find that newer kernels may not boot on older systems (or vice versa). The device tree itself may be buried deeply within a system's firmware, making it hard to update, so incompatible binding changes may be more than slightly inconvenient for users. The normal kernel rule is that systems that work with a given kernel should work with all releases thereafter; no explicit exception exists for device tree bindings. So, many feel, bindings should be treated like a stable kernel ABI. 

Perhaps the strongest advocate of the position that device tree bindings should be treated as any other ABI right now (rather than sometime in the future) is [ARM maintainer Russell King](<https://lwn.net/Articles/561464/>): 

We can draw the line at an interface becoming stable in exactly the same way that we do every other "stable" interface like syscalls - if it's in a -final kernel, then it has been released at that point as a stable interface to the world. [...] 

If that is followed, then there is absolutely no reason why a "Stable DT" is not possible - one which it's possible to write a DT file today, and it should still work in 20 years time with updated kernels. That's what a stable interface _should_ allow, and this is what DT _should_ be. 

As is often the case, though, there is a disconnect between what should be and what really is. The current state of device tree stability was perhaps best [summarized](<https://lwn.net/Articles/561465/>) by Olof Johansson: 

Until now, we have been working under the assumption that the bindings are _NOT LOCKED_. I.e. they can change as needed, and we _ARE_ assuming that the device tree has to match the kernel. That has been a good choice as people get up to speed on what is a good binding and not, and has given us much-needed room to adjust things as needed. 

Other developers agreed with this view of the situation: for the first few years of the ARM migration from board files to device trees, few developers (if any) had a firm grasp of the applicable best practices. It was a learning experience for everybody involved, with the inevitable result that a lot of mistakes were made. Being able to correct those mistakes in subsequent kernel releases has allowed the quick application of lessons learned and the creation of better bindings in current kernels. But Olof went on to say that the learning period is coming to a close: ""That obviously has to change, but doing so needs to be done carefully."" This transition will need to be done carefully indeed, as can be seen from the issues raised in the discussion. 

#### Toward stable bindings

For example: what should be done about "broken" bindings that exist in the kernel currently? Would they immediately come under a guarantee of stability, or can they be fixed one last time? There is a fair amount of pressure to stop making incompatible changes to bindings immediately, but to do so would leave kernel developers supporting bindings that do not adequately describe the hardware, are not extensible to newer versions of the hardware, and are inconsistent with other bindings. Thus, Tomasz Figa [argued](<https://lwn.net/Articles/561469/>), current device tree bindings should be viewed as a replacement for board files, which were very much tied to a specific kernel version: 

We have what we have, it is not perfect, some things have been screwed up, but we can't just leave that behind and say "now we'll be doing everything correctly", we must fix that up. 

Others contend that, by releasing those bindings in a stable kernel, the community already committed itself to supporting them. Jon Smirl has [advocated](<https://lwn.net/Articles/561471/>) for a solution that might satisfy both groups: add a low-level "quirks" layer that would reformat old device trees to contemporary standards before passing them to the kernel. That would allow the definitive bindings to change while avoiding breaking older device trees. 

Another open question is: what is the process by which a particular set of bindings achieves stable status, and when does that happen? Going back to Olof's original message: 

It's likely that we still want to have a period in which a binding is tentative and can be changed. Sometimes we don't know what we really want until after we've used it a while, and sometimes we, like everybody else, make mistakes on what is a good idea and not. The alternative is to grind most new binding proposals to a halt while we spend mind-numbing hours and hours on polishing every single aspect of the binding to a perfect shine, since we can't go back and fix it. 

Following this kind of policy almost certainly implies releasing drivers in stable kernels with unstable device tree bindings. That runs afoul of the "once it's shipped, it's an ABI" point of view, so it will not be popular with all developers. Still, a number of developers seem to think that, with the current state of the art, it still is not possible to create bindings that are long-term supportable from the beginning. Whether bindings truly differ from system calls and other kernel ABIs in this manner is a topic of ongoing debate. 

Regardless of when a binding is recognized as stable, there is also the question of who does this recognition. Currently, bindings are added to the kernel by driver developers and subsystem maintainers; thus, in some eyes, we have a situation where the community is being committed to support an ABI by people who do not fully understand what they are doing. For this reason, Russell [argued](<https://lwn.net/Articles/561474/>) that no device tree binding should be merged until it has had an in-depth review by somebody who not only understands device tree bindings, but who also understands the hardware in question. That bar is high enough to make the merging of new bindings difficult indeed. 

Olof's message, instead, proposed the creation of a "standards committee" that would review bindings for stable status. These bindings might already be in the kernel but not yet blessed as "locked" bindings. As Mark Rutland (one of the new bindings maintainers) [pointed out](<https://lwn.net/Articles/561475/>), this committee would need members from beyond the Linux community; device tree bindings are supposed to be independent of any specific operating system, and users may well want to install a different system without having to replace the device tree. Stephen Warren (another new bindings maintainer) [added](<https://lwn.net/Articles/561476/>) that bootloaders, too, make use of device trees, both to understand the hardware and to tweak the tree before passing it to the kernel. So there are a lot of constituents who would have to be satisfied by a given set of bindings. 

Tied to this whole discussion is the idea of moving device tree bindings out of the kernel entirely and into a repository of their own. Such a move would have the effect of decoupling bindings from specific kernel releases; it would also provide a natural checkpoint where bindings could be carefully reviewed prior to merging. Such a move does not appear to be planned for the immediate future, but it seems likely to happen eventually. 

There are also some participants who questioned the value of stable bindings in the first place. In particular, Jason Gunthorpe [described](<https://lwn.net/Articles/561477/>) the challenges faced by companies shipping embedded hardware with Linux: 

There is no way I can possibly ship a product with a DT that is finished. I can't tie my company's product release cycles to the whims of the kernel community. 

So embedded people are going to ship with unfinished DT and upgrade later. They have to. There is no choice. Stable DT doesn't change anything unless you can create perfect stable bindings for a new SOC instantaneously. 

In Jason's world, there is no alternative to being able to deal with device trees and kernels that are strongly tied together, and, as he sees it, no effort to stabilize device tree bindings is going to help. That led him to [ask](<https://lwn.net/Articles/561478/>): ""So who is getting the benefit of this work, and is it worth the cost?"" That particular question went unanswered in the discussion. 

Finally, in a world where device tree bindings have been stabilized, there is still the question of how to ensure that drivers adhere to those bindings and add no novelties of their own. The plan here appears to be the creation of a schema to provide a formal description for bindings, then to augment the `dtc` device tree compiler to verify device trees against the schema. Any strange driver-specific bindings would fail to compile, drawing attention to the problem. 

The conversation quickly acquired a number of interesting side discussions on how the schema itself should be designed. A [suggestion](<https://lwn.net/Articles/561480/>) that XML could be used evoked far less violence than one might imagine; kernel developers are still trying hard to be nice, it seems. But David Gibson's [suggestion](<https://lwn.net/Articles/561482/>) that a more C-like language be used seems more likely to prevail. The process of coming up with comprehensive schema definition and checking that it works with all device tree bindings is likely to take a while. 

Reaching a consensus on when device tree bindings should be stabilized, what to do about substandard existing bindings, and how to manage the whole process will also probably take a while. The topic has already been [penciled in](<https://lwn.net/Articles/561483/>) for an entire afternoon during the ARM Kernel Summit, to be held in Edinburgh this October. In the meantime, expect a lot of discussion without necessarily binding the community to more stable device trees.
