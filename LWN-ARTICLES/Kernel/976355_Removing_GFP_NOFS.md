---
title: Removing GFP_NOFS
url: https://lwn.net/Articles/976355/
date: "June 5, 2024"
category: "Filesystems; Memory management-GFP flags"
author: "By Jake Edge June 5, 2024 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
June 5, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The `GFP_NOFS` flag is meant for kernel memory allocations that should not cause a call into the filesystems to reclaim memory because there are already locks held that can potentially cause a deadlock. The "scoped allocation" API is a better choice for filesystems to indicate that they are holding a lock, so `GFP_NOFS` has [long been on the chopping block](<https://lwn.net/Articles/753078/>), though progress has been slow. In a filesystem-track session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Matthew Wilcox wanted to discuss how to move kernel filesystems away from the flag with the eventual goal of removing it completely. 

He began the session by saying that there are several changes that people would like with regard to the `GFP` flags, but that the scoped-allocation API (i.e. [`memalloc_nofs_save()` and `memalloc_nofs_restore()`](<https://elixir.bootlin.com/linux/v6.9.3/source/include/linux/sched/mm.h#L366>) as mentioned in the [LSFMM+BPF topic discussion](<https://lwn.net/ml/linux-fsdevel/ZZcgXI46AinlcBDP@casper.infradead.org/>)) for `GFP_NOFS` went in long ago, while the conversion to it is far from complete. He also wanted to talk a bit about Rust, he said. There is a desire to bring in Rust code from outside the kernel for, say, a hash table, but that requires the ability to allocate memory, which means making `GFP` flags available. "Why the hell would we want to add `GFP` flags to every Rust thing that we bring into the kernel? That's crazy." 

So, he asked, what interface would the filesystem developers like to have to indicate that an allocation should not recurse into the filesystem code; the existing interface is, seemingly, not the right one, since filesystems are generally not using it. Josef Bacik said that from the Btrfs perspective, there is no real advantage to switching away from `GFP_NOFS` because there are other `GFP` flags that it still needs to use. The flags are already plumbed through the Btrfs code, so it is just easier to add another `GPF_NOFS` use when that situation arises. 

[ ![\[Matthew Wilcox\]](https://static.lwn.net/images/2024/lsfmb-wilcox-sm.png) ](<https://lwn.net/Articles/976854/>)

But the Rust use case is not one he had considered before; it is the first example he has seen where the save and restore interface makes a difference that makes the switch worth doing. Bacik said that he would be willing to change the code to eliminate `GFP_NOFS` if that was requested, but it has not really "seemed like a pressing need", so far, to him. The Rust need for the switch changes that equation in his mind. 

Jan Kara said that he has been working on removing the flag from ext4 and other places, but has found it inconvenient to do in some parts of the code. Passing the cookie that gets returned from the save operation through, so that it is available to pass to the restore, is ugly and makes the conversion harder, he said. Similarly, there are locks that are taken in some paths such that recursing back into the filesystem needs to be prevented (which is what `GFP_NOFS` protects against). Rather than having code that manually does the save and restore while managing the cookie, marking those locks in a way that causes the system to automatically handle the allocation scope would make things easier for conversions. 

Rust for Linux developer Wedson Almeida Filho said that it was painful to have to manage and think about all of the different `GFP` flags in the code. He wondered if there were some way to automatically set the scope by detecting the areas where one of them is needed. Ideally, that detection would happen at compile time; there could perhaps be support in Rust for that, he suggested. 

Wilcox said that it depends on the specific filesystem, because they work in different ways with regard to their locking. The detection would have to know that a particular lock is taken during the reclaim path, for example. Kara suggested that [lockdep](<https://www.kernel.org/doc/html/latest/locking/lockdep-design.html>) might help, but Wilcox seemed skeptical, noting that lockdep cannot say that a particular lock is never taken in the reclaim path, for example. Dave Chinner agreed that it would be difficult to detect that situation since the filesystem code, its locking, and the interaction with the reclaim path are extremely complex. 

There was some discussion between Chinner, Almeida, and Kent Overstreet about the difficulty of automatically detecting the proper context. There is also some interaction with lockdep, which emits false-positive warnings; that has led the XFS developers to use the `__GFP_NOLOCKDEP` flag (some of which is [described in a patch from January](<https://lwn.net/Articles/958426/>)). It was all rather fast-moving and technically deep. 

Overstreet raised the idea of [preventing kernel allocations from failing](<https://lwn.net/Articles/964793/>) (effectively making `GFP_NOFAIL` more widespread). He is opposed to that effort because he thinks that it is important to ensure that the error paths are well-tested both for allocation failure and other errors. But he wondered if making that change might eliminate the need for [memory pools](<https://www.kernel.org/doc/html/latest/core-api/mm-api.html?highlight=mempool#memory-pools>) (or "mempools"). 

Chinner said that there is a difference between mempools and the no-fail case: mempools provide a guarantee of forward progress that no-fail does not. In particular, when memory is being reclaimed, there is no guarantee that whatever needs the memory will actually get it. That reclaimed memory could be allocated to some other task, unlike with mempools. He thought it would be difficult to be sure that making that switch would work in all cases. 

But what a no-fail policy does do, he continued, is remove the possibility of dereferencing null pointers when there is an allocation failure. Those kinds of bugs generally have security implications, so eliminating the possibility of allocation failure can remove a whole class of security-sensitive bugs. 

Overstreet said that making the error paths easier to test is another approach. He plans to post some patches for that, including ways to inject errors at any memory-allocation site; those patches rely on his recently merged [memory-allocation profiler](<https://lwn.net/Articles/974380/>). Bacik said that Btrfs has ways to inject errors to test its error paths using BPF scripts. Overstreet said that it is important to be able to target the error injection for code that is under your control; simply randomly failing memory allocations for the kernel as a whole is not viable. Bacik said that the Btrfs error paths are systematically tested using the BPF code. 

The session ran out of time without coming to any conclusions on the path forward, which is unfortunate, Wilcox said. Everyone seemed interested in removing `GFP_NOFS` allocations, but there is no concrete proposal for how to get there; he will try to work on one. Now that he realizes there is a major push to get rid of those allocations, Bacik said that he will work with the other Btrfs developers to not add any more and to start removing the ones that are there.
