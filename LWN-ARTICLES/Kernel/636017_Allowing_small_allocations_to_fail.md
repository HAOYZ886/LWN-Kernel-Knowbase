---
title: Allowing small allocations to fail
url: https://lwn.net/Articles/636017/
date: "March 11, 2015"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet March 11, 2015 LSFMM 2015"
---

By **Jonathan Corbet**  
March 11, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

As Michal Hocko noted at the beginning of his session at the 2015 [Linux Storage, Filesystem, and Memory Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>), the news that the memory-management code will normally retry small allocations indefinitely rather than returning a failure status came as a surprise to many developers. Even so, this behavior is far from new; it was first added to the kernel in 2001\. At that time, only order-0 (single-page) allocations were treated that way, but, as the years went by, that limit was raised repeatedly; in current kernels, anything that is order-3 (eight pages) or less will not normally be allowed to fail. The code to support this mode of operation has become more complex over time as well. 

Relatively late in the game, the `__GFP_NOFAIL` flag was added to specifically annotate the places in the kernel where failure-proof allocations are needed, but the "too small to fail" behavior has never been removed from other allocation operations. After 14 years, Michal said, there will certainly be many places in the code that depend on these semantics. That is unfortunate, since the failure-proof mode is error-prone and unable to deal with real-world situations like infinite retry loops outside of the allocator, locking conflicts, and out-of-memory (OOM) situations. The result is occasional lockups as described in [this article](<https://lwn.net/Articles/627419/>). 

There have been various attempts to get around the problem, such as adding timeouts to the OOM killer (see [this article](<https://lwn.net/Articles/635354/>)), but Michal thinks such approaches are "not nice." The proper way to handle that kind of out-of-memory problem is to simply fail allocation requests when the necessary resources are not available. Most of the kernel already has code to check for and deal with such situations; [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2015/lsfmm/MichalHocko-sm.jpg)](<https://lwn.net/Articles/636053/>) beyond that, the memory-management code should not be attempting to dictate the failure strategy to the rest of the kernel. 

Changing the allocator's behavior is relatively easy; the harder question is how to make such a change without introducing all kinds of hard-to-debug problems. The current code has worked for 14 years, so there will be many paths in the kernel that rely on it. Changing its behavior will certainly expose bugs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Michal posted [a patch](<https://lwn.net/Articles/636018/>) just before the summit demonstrating the approach to the problem that he is proposing. That patch adds a new sysctl knob that controls how many times the allocator should retry a failed attempt before returning a failure status; setting it to zero disables retries entirely, while a setting of `-1` retains the current behavior. There is a three-stage plan for the use of this knob. In the first stage, the default setting would be for indefinite retries, leaving the kernel's behavior unchanged. Developers and other brave people, though, would be encouraged to set the value lower. The hope is to find and fix the worst of the resulting bugs in this stage. 

In the second stage, an attempt would be made to get distributors to change the default value. In the third and final stage, the default would be changed in the upstream kernel itself. Even in this stage, where, in theory, the bugs have been found, the knob would remain in place so that especially conservative users could keep the old behavior. 

Michal opened up the discussion by asking if the assembled developers thought this was the right approach. Rik van Riel said that most kernel code can handle allocation failure just fine, but a lot of those allocations happen in system calls. In such cases, the failures will be passed back to user space; that is likely to break applications that have never seen certain system calls fail in this way before. 

Ted Ts'o added that the kernel would mostly likely be stuck in the first stage for a very long time. As soon as distributions start changing the allocator's behavior, their phones will start ringing off the hook. In the ext4 filesystem, he has always been nervous about passing out-of-memory failures back to user space because of the potential for application problems. If the system call interface does that instead it won't be his fault, he said, but things will still break. 

Peter Zijlstra observed that `ENOMEM` is a valid return from a system call. Ted agreed, but said that, after all these years, applications will break anyway, and then users will be knocking at his door. He went on to say that in large data-center settings (Google, for example) where the same people control both kernel and user space it should be possible to find and fix the resulting bugs. But just fixing the bugs in open-source programs is going to be a long process. In the end, he said, such a change is going to have to provide a noticeable benefit to users — a much more robust kernel, say — or we will be torturing them for no reason. 

Andrew Morton protested that the code we have now seems to work almost all of the time. Given that the reported issues are quite rare, he asked, what problem are we actually trying to solve? Andrea Arcangeli noted that he'd observed lockups and that the OOM killer's relatively unpredictable behavior does not help. He tried turning off the looping in the memory allocator and got errors out of the ext4 filesystem instead. It was a generally unpleasant situation. 

Andrew suggested that making the OOM killer work better might be a better place to focus energy, but Dave Chinner disagreed, saying that it was an attempt to solve the wrong problem. Rather than fix the OOM killer, it would be better to not use it at all. We should, he said, take a step back and ask how we got into the OOM situation in the first place. The problem is that the system has been overcommitted. Michal said that overcommitting of resources was just the reality of modern systems, but Dave insisted that we need to look more closely at how we manage our resources. 

Andrew returned to the question of improving the OOM killer. Perhaps, he said, it could be made to understand lock dependencies and avoid potential deadlock situations. Rik suggested that was easier said than done, though; for example, an OOM-killed process may need to acquire new locks in order to exit. There will be no way for the OOM killer to know what those locks might be prior to choosing a victim to kill. Andrew acknowledged the difficulties but insisted that not enough time has gone into making the OOM killer work better. Ted said that OOM killer improvements were needed regardless of any other changes; since the allocator's default behavior cannot be changed for years, we will be stuck with the OOM killer for some time. 

Michal was nervous about the prospect of messing with the OOM killer. We don't, he said, want to go back to the bad old days when its behavior was far more random than it is now. Dave said, though, that it is not possible to have a truly deterministic OOM killer if the allocation layers above it are not deterministic. It will behave differently every time it is tested. Until things are solidified in the allocator, the OOM killer is, he said, not the place to put effort. 

The session wound down with Michal saying that starting to test kernels that fail small allocations will be helpful even if the distributors do not change the default for a long time. Dave said that he would turn off looping in the xfstests suite by default. There was some talk about the best values to use, but it seems it matters little as long as the indefinite looping is turned off. Expect to see a number of interesting bugs once this testing begins. 

[Your editor would like to thank LWN subscribers for funding his travel to LSFMM 2015.]
