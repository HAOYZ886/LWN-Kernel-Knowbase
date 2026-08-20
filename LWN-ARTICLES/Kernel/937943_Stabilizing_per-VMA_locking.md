---
title: "Stabilizing per-VMA locking"
url: https://lwn.net/Articles/937943/
date: "July 13, 2023"
category: "Memory management-mmap sem; Releases-6.4"
author: "By Jonathan Corbet July 13, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 13, 2023

The kernel-development process routinely absorbs large changes to fundamental subsystems and still produces stable releases every nine or ten weeks. On occasion, though, the development community's luck runs out. The [per-VMA locking work](<https://lwn.net/Articles/906852/>) that went into the 6.4 release is a case in point; it looked like a well-tested change that improved page-fault scalability. There turned out to be a few demons hiding in that code, though, that made life difficult for early adopters of the 6.4 kernel. 

The `mmap_lock` controls access to a process's address space. If a process has a lot of things going on at the same time — a large number of threads, for example — contention for the `mmap_lock` can become a serious performance problem. Handling page faults, which can involve changing page-table entries or expanding a virtual memory area (VMA), has traditionally required the `mmap_lock`, meaning that many threads generating page faults concurrently can slow a process down. 

Developers have been working for years to address this problem, often pursuing one form or another of [speculative page-fault handling](<https://lwn.net/Articles/730531/>), where the work is done without the protection of the lock. If contention occurred during the handling of a fault, the kernel would drop the work it had done before making it visible and start over. This work never got to the point of being merged, though. Eventually, Suren Baghdasaryan's per-VMA locking work was merged instead. Since a page fault happens within a specific VMA, handling it should really only require serialized access to that one VMA rather than to the entire address space. Per-VMA locking is, thus, a finer-grained version of `mmap_lock` that allows for more parallelism in the execution of specific tasks. 

After a few iterations, this work was pulled into the mainline during the 6.4 merge window. Entry into the mainline tends to increase the amount of testing a change is subject to, but none of that testing turned up any problems with per-VMA locking, which saw no fixup changes during the remainder of the 6.4 development cycle. Per-VMA locking, it seemed, was a done deal. 

#### Trouble arises

A mainline kernel release gets even more testing, though. The 6.4 release happened on June 25; it took four days for Jiri Slaby to [report](<https://lwn.net/ml/linux-kernel/dbdef34c-3a07-5951-e1ae-e9c6e3cdf51b@kernel.org/>) problems with the Go language build. On July 2, Jacob Young [reported](<https://bugzilla.kernel.org/show_bug.cgi?id=217624>) ""frequent but random crashes"" and narrowed the problem down to a simple test case that could reproduce it. In both cases, investigation quickly pointed a finger at the per-VMA locking work. There were, it seems, cases where unexpected concurrent access was corrupting data structures within the kernel. 

Random crashes and data corruption are not the sort of experience users are looking for when they upgrade to a new "stable" kernel release. For extra fun, the [StackRot vulnerability](<https://lwn.net/Articles/937377/>) led to the merging of some significant changes to how stack expansion is handled — changes that had not seen public review and which were quickly shipped in the [6.4.1](<https://lwn.net/Articles/937080/>) stable update. That work introduced another VMA-locking bug in 6.4 kernels, having added another case that needed explicit serialization. 

Baghdasaryan responded quickly to all of these problems, posting [a patch](<https://lwn.net/ml/linux-mm/20230703182150.2193578-1-surenb@google.com/>) to simply disable the feature until the problems were worked out. That patch needed some tweaking of its own, but it was never applied to the mainline in any form. Memory-management maintainer Andrew Morton [went into](<https://lwn.net/ml/linux-kernel/20230704091808.aa2ed3c11a5351d9bf217ac9@linux-foundation.org/>) ""wait-a-few-days-mode"" in the hopes that a proper fix for the problem would emerge. Greg Kroah-Hartman, meanwhile, [said](<https://lwn.net/ml/linux-kernel/2023070509-undertow-pulverize-5adc@gregkh/>) that nothing could be done for the stable kernels until some patch landed in the mainline kernel. As a result, the 6.4 kernel lacked any sort of workaround for this serious bug. 

On July 5, Thorsten Leemhuis (working in his regression-tracking role) [wondered](<https://lwn.net/ml/linux-kernel/7668c45a-70b1-dc2f-d0f5-c0e76ec17145@leemhuis.info/>) how long the wait would be, noting that the faster-moving community distributions would already be starting to ship the 6.4 kernel. Morton [answered](<https://lwn.net/ml/linux-kernel/20230705084906.22eee41e6e72da588fce5a48@linux-foundation.org/>) that he would send the fixes that existed at the time, but that did not actually happen for a few more days, leading Leemhuis to [think](<https://lwn.net/ml/linux-kernel/df1d7d39-56f3-699c-0d0f-fcc8774f182e@leemhuis.info/>) that he needs to start sounding the alarm more quickly on serious regressions like this one. Had he done so in this case, he thought, the problem might have been addressed more quickly. 

Morton sent [a patch series](<https://lwn.net/ml/linux-kernel/20230708104658.927e647f75d55c4a5d99cbf5@linux-foundation.org/>) to Linus Torvalds on July 8 that included, among other things, the disabling of per-VMA locking. Torvalds, though, undid that change when merging the set, because three other patches had gone straight into the mainline by then: 

  * [The first one](<https://git.kernel.org/linus/c137381f71ae>) added some locking to the new stack-expansion code, undoing the problem that had been introduced by the security fixes. 
  * [The next](<https://git.kernel.org/linus/33313a747e81>) ensures that newly created VMAs are locked before they are made visible to the rest of the kernel. The problem fixed did not affect any released kernels, but it was a latent bug that would be exposed by the planned expansion of per-VMA locking in the future. 
  * Finally, [this patch](<https://git.kernel.org/linus/fb49c455323f>) fixed the problem that was being reported. If a process called `fork()`, then incurred a page fault on a VMA while that VMA was being copied into the child process, the result could be the corruption of the VMA. The fix is to explicitly lock each VMA before beginning the copy, slowing down `fork()`-heavy workloads by as much as 5%. 

It's worth noting that Torvalds was [opposed](<https://lwn.net/ml/linux-kernel/CAHk-=wi=+qS+5v_7mVxKWUSOKaxzC2V8N7hyFVt1qTWGM_pmAQ@mail.gmail.com/>) to the idea of disabling per-VMA locking; instead he insisted that, if it could not be fixed, it should be removed entirely: 

> I seriously believe that if the per-vma locking is so broken that it needs to be disabled in a development kernel, we should just admit failure, and revert it all. 
> 
> And not in a "revert it for a later attempt" kind of way. 
> 
> So it would be a "revert it because it added insurmountable problems that we couldn't figure out" thing that implies *not* trying it again in that form at all, and much soul-searching before somebody decides that they have a more maintainable model for it all. 

By all appearances, the fixes (which were included in the [6.4.3](<https://lwn.net/Articles/937886/>) stable update on July 11) are sufficient, and per-VMA locking is now stable. There should, thus, be no need to revert and soul-search in this case. That said, it's worth keeping in mind that this work looked stable before the 6.4 release as well. 

#### Closing thoughts

The per-VMA locking work made fundamental changes to a core kernel subsystem, moving some page-fault handling outside of a sprawling lock, the coverage of which is probably not fully understood by anybody. The fact that it turned out to have some subtle bugs should not be especially surprising. It is hard to make this kind of change to the kernel without running into trouble somewhere, but this is also precisely the sort of change that developers need to be able to make if Linux is to adapt to current needs. 

In theory, avoiding subtle locking problems is one of the advantages of using a language like Rust. Whether, say, a VMA abstraction written in Rust could truly ensure that accesses use proper locking while maintaining performance has not yet been proven in the kernel context, though. 

There is a case to be made that this regression could have been handled better. Perhaps there should be a way for an immediate "disable the offending feature" patch to ship in a stable release, even if that change has not gone into the mainline, and even if a better fix is expected in the near future. Kernel developers often make the point that the newest kernels are the best ones that the community knows how to make and that users should not hesitate to upgrade to them. Responding more quickly when an upgrade turns out to be a bad idea would help to build confidence in that advice. 

Meanwhile, though, developers did respond quickly to the problem and proper fixes that allowed the feature to stay in place were found. The number of users exposed to the bug was, hopefully, relatively small. There is, in other words, a case to be made that this problem was handled reasonably well and that we will all get the benefits of faster memory management as a result of this work.
