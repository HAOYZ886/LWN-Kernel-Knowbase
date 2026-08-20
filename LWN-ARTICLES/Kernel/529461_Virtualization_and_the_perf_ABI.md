---
title: Virtualization and the perf ABI
url: https://lwn.net/Articles/529461/
date: "December 19, 2012"
category: "Development model-User-space ABI; Performance monitoring"
author: "By Jake Edge December 19, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
December 19, 2012

Breaking the application binary interface (ABI) between the kernel and user space is a well-known taboo for Linux. That line may seem a little blurrier to some when it comes to the ABI for tools like `perf` that ship with the kernel. As a recent discussion on the linux-kernel mailing list shows, though, Linus Torvalds and others still have that line in sharp focus. 

The issue stems from what appears to be a fairly serious bug in some x86 processors. Back in July, David Ahern [reported](<https://lkml.org/lkml/2012/7/9/264>) that KVM-based virtual machines would crash when recording certain events on the host. On some x86 processors, the "Precise Events Based Sampling" (PEBS) mechanism can be used to gather precise counts of events like CPU cycles. Unfortunately, PEBS and hardware virtualization don't play nicely together. 

As Ahern reported, running: 

```
perf record -e cycles:p -ag -- sleep 10
```

on the host would reliably crash all of the guests. That particular command will record the events specified, CPU cycles in this case, to a file; more information about `perf` can be found [here](<http://code.google.com/p/kernel/wiki/PerfUserGuide>). It turns out that PEBS incorrectly treats the contents of the Data Segment (DS) register as a guest address, rather than as a host address. That leads to memory corruption in the guest, which will crash all of the virtual machines on the system. The "`:p`" (precise) attribute on the `cycles` event (which can be repeated for higher precision levels as in `cycles:pp`) asks for more precise measurements, which leads to PEBS being used. Without that attribute, the cycle counts measured are less accurate, but do not cause the VM crashes. 

That problem led Peter Zijlstra to [change `perf_event.c`](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=20b279ddb38ca42f8863cec07b4d45ec24589f13>) in the kernel to disallow precise measurements unless guest measurement has been specifically excluded. Using the "`:H`" (host-only) attribute will still allow precise measurements as `perf` will set the `exclude_guest` flag on the event. That flag will inhibit PEBS activity while in the guest. In addition, Ahern [changed `perf`](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=1342798cc13e3b48d9b5738f0c8fa812ccea8101>) so that `exclude_guest` would be automatically selected if the "precise" attribute was set. There's just one problem with those solutions: existing `perf` binaries do not set `exclude_guest`, so users would get an `EOPNOTSUPP` error. 

It turns out that one of those existing users is Torvalds, who [complained](<https://lwn.net/Articles/529716/>) that: 

```
perf record -e cycles:pp
```

no longer worked for him. Ahern [suggested](<https://lwn.net/Articles/529718/>) using "`cycles:ppH`", but that elicited an [annoyed response](<https://lwn.net/Articles/529720/>) from Torvalds. Why should he have to add a new flag to deal with virtualization, when he isn't running it? ""That whole 'exclude_guest' test is insane when there isn't any virtualization going on."" 

Ahern [countered](<https://lwn.net/Articles/529728/>) that it's worse to have VMs explode because someone runs a precise `perf`. But that's beside the point, as Torvalds [pointed out](<https://lwn.net/Articles/529730/>): 

You broke the WORKING case for old binaries in order to give an error return in a case that NEVER EVEN WORKED with those binaries. Don't you see how insane that is? 

The 'H' flag is totally the wrong way around. Exactly because it only "fixes" a case that was already working, and makes a case that never worked anyway now return an error value. That's not sane. Since the old broken case never worked, nobody can have depended on it. See why I'm saying that it's the people who use virtualization who should be forced to use the new flag, not the other way around? 

Forcing existing `perf` binary users to change their habits is the crux of the matter. Beyond breaking the ABI, which is clearly not allowed, it makes `perf` break for real users as Ingo Molnar [said](<https://lwn.net/Articles/529732/>): ""Old, working binaries are actually our _most_ important usecase: it's 99.9% of our current installed base ..."". While it is certainly a problem that older kernels can have all their guests crashed with a simple command, the proper solution is not to require either upgrading `perf` or changing the flags (which could well be buried in scripts or other automation). 

Existing `perf` binaries set the `exclude_guest` flag to zero, while binaries that have Ahern's change set it to one. That means newer kernels that seek to fix the crashing guest bug cannot rely on a particular value for that flag. The "proper" way to have handled the problem is to use a new `include_guest` flag (or similar), which defaults to zero. Older binaries cannot change that flag (since they don't know about it), so the kernel code can use it to exclude the precise flag for guests on x86 systems. Other architectures may not suffer from the same restriction. 

Beyond that, Torvalds [argues](<https://lwn.net/Articles/529766/>) that if the user asks for a precise measurement but doesn't specify either the "`H`" or "`G`" (include guests) attribute, the code should try to do the right thing. That means it should measure both the host and guests on systems that support it, while backing off to just the host for x86. Meanwhile it could return `EOPNOTSUPP` if the user explicitly asks for a broken combination (e.g. precise and include guests on x86). Molnar concurred. Ahern seemed a bit unhappy about things, but said that he would start working on a patch that has not appeared yet. 

It is worth noting that Torvalds [admitted](<https://lwn.net/Articles/529950/>) that he could trivially recompile `perf` to get around the whole problem; it was a principle that he was standing up for. Even though some tools like `perf` are distributed with the kernel tree, that does not relax the "no regressions" rule. Some critics of the move to add tools to the kernel tree were concerned that it would facilitate ABI changes that could be glossed over by requiring keeping the tools and kernel in sync. This discussion clearly shows that not to be the case. 

Having a way to crash all the VMs on a system is clearly undesirable, but as Torvalds pointed out, that had been true for quite some time. Undesirable behavior does not rise to the level of allowing ABI breakage, however. In addition, distributions and administrators can always limit access to `perf` to the root user--though that obviously may still lead to unexplained VM crashes as Ahern [noted](<https://lwn.net/Articles/529772/>). Molnar [pointed out](<https://lwn.net/Articles/529773/>) that the virtualization use case is a much smaller piece of the pie, so making everyone else pay for a problem they may never encounter just doesn't make sense. Either through a patch or a revert, it would seem that the "misbehavior" will disappear before 3.8 is released.
