---
title: Xen again
url: https://lwn.net/Articles/335812/
date: "June 3, 2009"
category: "Virtualization-Xen; Xen"
author: "By Jonathan Corbet June 3, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 3, 2009

Your editor is widely known for his invariably correct and infallible predictions. So, certainly, he would never have said [something like this](<http://lwn.net/Articles/321696/>): 

Mistakes may have been made in Xen's history, but it is a project which remains alive, and which has clear reasons to exist. Your editor predicts that the Dom0 code will find little opposition at the opening of the 2.6.30 merge window. 

OK, anybody needing any further evidence of your editor's ability to foresee the future need only look at his investment portfolio...or, shall we say, the smoldering remains thereof. Needless to say, Xen Dom0 support did not get through the 2.6.30 merge window, and it's not looking very good for 2.6.31 either. 

Dom0, remember, is the hypervisor portion of the Xen system; it's the One Ring which binds all the others. Unlike the DomU support (used for ordinary guests), Dom0 remains outside of the mainline kernel. So anybody who ships it must patch it in separately; for a patch as large and intrusive as Dom0, that is not a pleasant task. It is a necessary one, though; Xen has a lot of users. As [expressed](<https://lwn.net/Articles/335814/>) by Xen hacker Jeremy Fitzhardinge: 

Xen is very widely used. There are at least 500k servers running Xen in commercial user sites (and untold numbers of smaller sites and personal users), running millions of virtual guest domains. If you browse the net at all widely, you're likely to be using a Xen-based server; all of Amazon runs on Xen, for example. Mozilla and Debian are hosted on Xen systems. 

Xen developers and users would all like to see that code merged into the mainline. A number of otherwise uninvolved kernel developers have also argued in favor of merging this code. So one might well wonder why there is still opposition. 

One problem is a fundamental disagreement with the Xen design, which calls for a separate user-space hypervisor component. To some developers, it looks like an unfortunate mishmash of code in the mainline kernel, in Xen-specific kernel code, and in user space - with, of course, a set-in-concrete user-space ABI in the middle. Many developers are more comfortable with the fully in-kernel hypervisor approach taken by KVM. Thomas Gleixner [is especially worried](<https://lwn.net/Articles/335815/>) about the possible results of merging the Xen Dom0 code for this reason (among several others): 

Aside of that it can also hinder the development of a properly designed hypervisor in Linux: 'why bother with that new stuff, it might be cleaner and nicer, but we have this Xen dom0 stuff already?'. 

Steven Rostedt, who has worked on Xen in the past, [also dislikes the hypervisor design](<https://lwn.net/Articles/335816/>) and the effects it has on kernel development: 

The major difference between KVM and Xen is that KVM _is_ part of Linux. Xen is not. The reason that this matters is that if we need to make a change to the way Linux works we can simply make KVM handle the change. That is, you could think of it as Dom0 and the hypervisor would always be in sync. 

If we were to break an interface with Dom0 for Xen then we would have a bunch of people crying foul about us breaking a defined API. One of Thomas's complaints (and a valid one) is that once Linux supports an external API it must always keep it compatible. This will hamper new development in Linux if the APIs are scattered throughout the kernel without much thought. 

Steven suggests merging the Xen hypervisor into the mainline so that it's all part of Linux, and to make the hypervisor ABI an internal, changeable interface. Some other developers - generally those most hostile to merging Dom0 in its current form - supported this idea. It's certainly not the first time that this sort of idea has been raised. But, despite many calls to bring some of the "plumbing layer" into the kernel proper, that has yet to happen; it seems unlikely that something as large as Xen would be the first user-space component to break through that barrier - even if the Xen developers were amenable to that approach. 

The hypervisor design would probably not be an insurmountable obstacle to merging by itself. But there are other complaints. The maintainers of the x86 architecture dislike the changes made to their code by the Dom0 patches. By their reckoning, there are far too many "`if (xen)...`" conditionals and too many `#ifdef`s. They would very much like to see the Xen code cleaned up and made less intrusive into the core x86 code. Linus [supports them](<https://lwn.net/Articles/335817/>) on this point: 

The fact is (and this is a _fact_): Xen is a total mess from a development standpoint. I talked about this in private with Jeremy. Xen pollutes the architecture code in ways that NO OTHER subsystem does. And I have never EVER seen the Xen developers really acknowledge that and try to fix it. 

The Xen cause was also not helped by [some performance numbers](<https://lwn.net/Articles/335818/>) posted by Ingo Molnar. If you choose the right benchmark, it seems, you can show that the paravirt_ops layer imposes a 1% overhead on kernel performance. Paravirt_ops is the code which abstracts low-level machine operations; it can enable the same kernel to run either on "bare metal" or virtualized under a hypervisor. It adds a layer of indirect function calls where, before, inline code was used. Those function calls come at a cost which has now been quantified by Ingo (but one should note that Rusty Russell has [shown](<https://lwn.net/Articles/335892/>) that, with the right benchmark, a number of other common configuration options have a much higher cost). 

The problem here is not that Xen users have a slower kernel; the real issue is that any kernel which might ever be run under Xen must be built with paravirt_ops enabled. There are few things which make distributors' lives more miserable than forcing them to build, ship, and support another kernel configuration. So most distributor kernels run with paravirt_ops enabled; that means that all users, regardless of whether they have any interest in Xen, pay the price. In some cases, that cost is too high; Nick Piggin [said:](<https://lwn.net/Articles/335819/>)

FWIW, we had to disable paravirt in our default SLES11 kernel. (admittedly this was before some of the recent improvements were made). But there are only so many 1% performance regressions you can introduce before customers won't upgrade (or vendors won't publish benchmarks with the new software). 

Ingo is strongly critical of the perceived cost of paravirt_ops, but he also [proposes a solution](<https://lwn.net/Articles/335820/>): 

Note what _is_ acceptable and what _is_ doable is to be a bit more inventive when dumping this optional, currently-high-overhead paravirt feature on us. My message to Xen folks is: use dynamic patching, fix your hypervisor and just use plain old-fashioned _restraint_ and common sense when engineering things, and for heaven's sake, _care_ about the native kernel's performance because in the long run it's your bread and butter too. 

He goes on to say that merging Dom0 now would only make things worse; it would give the Xen developers less incentive to fix the problems while, simultaneously, making it harder for distributors to disable paravirt_ops in their kernels. 

And that, perhaps, leads to the fundamental disconnect in this discussion. There are two distinctive lines of thought with regard to when code with known problems should be merged: 

  * Some developers point out that code which is in the mainline benefits from the attention of a much wider pool of developers and improves much more quickly. It is easy to find examples of code which, after languishing for years out of the mainline, improved quickly after being merged. This is the reasoning behind the -staging tree and the general policy toward merging drivers sooner rather than later. 

  * Some developers - sometimes, amusingly, the same developers - say, instead, that the best time to get fundamental problems fixed is before merging. This is undoubtedly true for user-space ABI issues; those often cannot be fixed at all after they have been shipped in a stable kernel. But holding code out of the mainline is also a powerful lever which subsystem maintainers can employ to motivate developers to fix problems. Once the code is merged, that particular tool is no longer available. 

Both of these themes run through the Xen discussion. There is no doubt that the Xen Dom0 code would see more eyeballs - and patches - after being merged. So some developers think that the right thing to do is to merge this much-requested feature, then fix it up afterward. Chris Mason [put it this way](<https://lwn.net/Articles/335822/>): 

The idea that we should take code that is heavily used is important. The best place to fix xen is in the kernel. It always has been, and keeping it out is just making it harder on everyone involved. 

But the stronger voice looks to be the one saying that the problems need to be fixed first. The deciding factors seem to be (1) the user-space ABI, and (2) the intrusion into the core x86 code; those issues make Xen different from yet another driver or filesystem. That, in turn, suggests that the Dom0 code is not destined for the mainline anytime soon. Instead, the Xen developers will be expected to go back and fix a list of problems - a lot of work with an uncertain result at the end.
