---
title: "What is to be done about MGLRU?"
url: https://lwn.net/Articles/1072866/
date: "May 20, 2026"
category: "Android; Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 20, 2026 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 20, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

"Reclaim" is the task of finding memory that can be taken away from its current user and put to better uses within the system; it is a core part of the memory-management picture. The addition of the [multi-generational LRU](<https://lwn.net/Articles/851184/>) (MGLRU) was meant to provide a better reclaim implementation than the "traditional LRU" that preceded it, but MGLRU has complicated the situation instead. No fewer than three memory-management-track sessions at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) were focused on MGLRU, with an eye toward integrating it more fully, improving its performance, and addressing some problems encountered with Android systems. 

#### Unifying the reclaim code

[![\[Shakeel Butt\]](https://static.lwn.net/images/conf/2026/lsfmm/ShakeelButt2-sm.png)](<https://lwn.net/Articles/1072877/>) When Shakeel Butt [proposed](<https://lwn.net/ml/linux-mm/20260325210637.3704220-1-shakeel.butt@linux.dev/>) a reclaim-oriented discussion, he was clear about his opinion on the current state of that code: 

> Memory reclaim in the kernel is a mess. We ship two completely separate eviction algorithms -- traditional LRU and MGLRU -- in the same file. mm/vmscan.c is over 8,000 lines. 40% of it is MGLRU-specific code that duplicates functionality already present in the traditional path. Every bug fix, every optimization, every feature has to be done twice or it only works for half the users. This is not sustainable. It has to stop. 

He was joined by Emil Tsalapatis at the resulting session to try to find some answers to a problem that, he said, is getting worse. They would like to find some way to unify the kernel's two reclaim subsystems. 

The methodology he proposed is not to pick a winner from the two alternatives but, instead, to find ways to unify the two as much as possible. That requires understanding both implementations well, which is complicated by the fact that the MGLRU developer has disappeared, and nobody really understands it at all, he said. (Subsequent discussion would make it clear that MGLRU is better understood than he thought). The unification task will need ways of evaluating performance — a good set of workloads that can reveal the strengths and weaknesses of specific algorithms. 

Lorenzo Stoakes suggested modularizing the code as a useful first step; that might be a good way to increase the amount of code shared between the two implementations. Butt then went into his high-level plan, which had four steps: 

  1. Separate the two code bases into their own files, rather than having them both in one big file. 
  2. Define workloads that can be used to evaluate reclaim patches. 
  3. Find the common features between the two implementations. 
  4. Compare the implementations of each feature. 

A participant questioned the plan to separate the files, worrying that it might confuse the Git history, but others generally seemed to think that some sort of cleanup was needed. Vlastimil Babka thought that separating the implementations might complicate the task of unifying them. Johannes Weiner said that MGLRU does not have much Git history in any case. We should separate what we can, he said, but there is a lot of intermingled code that will be difficult to tease out. Then ways should be found to implement features in common code used by both. 

Chris Li mentioned that Kairui Song has a number of good MGLRU improvements pending ([example](<https://lwn.net/ml/all/20260428-mglru-reclaim-v7-0-02fabb92dc43@tencent.com/>)); that work would not be helped by churning the code and creating a lot of conflicts. Improvements are good, he said, and we do not want to lose them. Butt agreed, but also said that this work should be evaluated based on whether it helps in the unification of the two implementations. Stoakes said that conflicts happen frequently in kernel development; they are managed and not allowed to block other work. 

An audience member worried that some features are not easily measurable with a set of benchmarks. He mentioned, in particular, that MGLRU will resort much more quickly to the out-of-memory killer than the traditional LRU will; he saw that as a good feature that prevents thrashing. 

Tsalapatis took over at this point to discuss the plan in more detail. It involves cleanly separating the two implementations to get a better sense of the structure of each and to see what is common between the two. The gathering of workloads will not try to create an exhaustive set; the purpose will be to get the important ones that can show improvements (or regressions). The definition of common features is an important part of this task; the two implementations have many features in common, but they use a different vocabulary to refer to them. Both implementations rely on a lot of heuristics; those need to be identified, with an eye toward any that are portable across both. For example, they calculate the ratio of file-backed to anonymous pages differently, and it is not clear why. Finally, for a unified implementation, decisions will have to be made between the available options for each feature. 

[![\[Emil Tsalapatis\]](https://static.lwn.net/images/conf/2026/lsfmm/EmilTsalapatis-sm.png)](<https://lwn.net/Articles/1072878/>) He covered a number of examples, starting with page-table-entry harvesting, which is the process of getting information (primarily the "this page was accessed" bit) from the page tables. The traditional LRU walks entries using the reverse-mapping infrastructure, while MGLRU uses page-table scans. It would be interesting to try switching them around and, eventually, make this implementation generic. 

The calculation of refault distance (essentially, how much time passes between when a page is reclaimed and it is faulted back in) is different between the two implementations, and he does not know why. The traditional LRU has an implementation that is ""battle-tested""; perhaps MGLRU could switch to using it. The two implementations maintain different statistics counters; they will have to be unified if code is to be shared. 

MGLRU, he said, gives priority to file-backed pages that have been mapped with [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>); the traditional LRU, instead, avoids deactivating executable pages. These two heuristics might be equivalent, but Tsalapatis was not sure. The MGLRU marks pages that have been quickly refaulted; the traditional LRU, instead, moves them directly to the active list. When it comes time to reclaim within a control group, the traditional LRU will evict pages until the lower watermark is reached, while MGLRU has a more complex load-balancing algorithm. 

The balance between file-backed and anonymous memory is one of the key heuristics applied by reclaim algorithms. The traditional LRU is driven by the [swappiness](<https://docs.kernel.org/admin-guide/sysctl/vm.html#swappiness>) setting, assisted by some special heuristics. MGLRU, instead, has [its proportional-integral-derivative (PID) controller](<https://docs.kernel.org/mm/multigen_lru.html#pid-controller>) to drive that decision, but still takes swappiness into account. It is not clear that the PID controller is actually beneficial. 

Song broke in at this point to say that answers to many of these questions can be found in his patch sets. He said he had chosen the wrong title, which should say "unify" somewhere. 

Time was running out; Tsalapatis said that it sounded like there was an agreement on how to proceed. John Hubbard, though, was not so sure. He would like to see a single reclaim implementation in the kernel; developers should pick one that has the best of both of the current implementations, then move on. Splitting the two, he said, does not help in reaching that goal. Butt said that the ultimate goal was to have just one reclaim implementation in the kernel. Stoakes said that there might turn out to be good reasons to keep both, but developers won't know for a while; meanwhile, having them both in the same file is hurting. He thought there was a general agreement on cleaning things up. 

Suren Baghdasaryan said that the plan generally looked good, and that the main contention was over splitting the two implementations. But he thought that the split should not be that hard, and that it could wait for the current work to be merged if need be. Weiner agreed, saying that the session had started with a claim that much of the system is not well understood and that the implementations should be unified. Song, he said, has already addressed a lot of that. Song's work should be the starting point, after which the split can happen. Li suggested that Song could do the split, but Weiner said that the task cannot be assigned in that way; this work needs to be a group effort. 

#### The plan to improve MGLRU

[![\[Kairui Song\]](https://static.lwn.net/images/conf/2026/lsfmm/KairuiSong2-sm.png)](<https://lwn.net/Articles/1072885/>) The next day, Song ran a session of his own to describe the work he has done with the MGLRU and what he thinks should come next. The MGLRU, he said, is not perfect but it is ""surprisingly good"" for many workloads. It has lower locking contention than the traditional LRU, and its lazy-promotion scheme makes it relatively efficient. It has lower reverse-mapping overhead, and its built-in bloom filter helps it to focus on the hot areas of memory. An audience member asked how much that filter actually helps; Song answered that it is easy to measure, and the answer is that it helps quite a bit. 

Song's goal is to make MGLRU ""better and smarter"". MGLRU performs better than the traditional LRU for many workloads now; the fact that it has been adopted by many distributions is a strong signal in that regard. It solves a number of practical problems; its time-to-live (TTL) heuristic helps to prevent thrashing, for example. Its working-set detection is getting better as well. One thing missing from MGLRU is writeback throttling for control groups; that is being fixed now, but global writeback throttling is still missing. 

MGLRU makes a distinction between folios created in response to a page fault and others; the fault-instantiated folios are activated more quickly. This behavior comes from an assumption that applications expect memory reads to not stall, Song said, but reads from a file mapped with `mmap()` do not fit that model. There is a [a patch](<https://lwn.net/ml/all/20260418120233.7162-1-baohua@kernel.org/>) from Barry Song that aims to fix this situation. He thought that it might be better to just remove this behavior, though. 

There are metrics on the number of active and inactive pages maintained in `/proc/meminfo`, but MGLRU doesn't manage them well, with the result that the numbers are ""jumpy"". It deems the two youngest generations of pages as being active, leading to big changes when the generations age. There is work underway (in the ["MGLRU-FG" patch set](<https://lwn.net/Articles/1070902/>)) to fix that problem. 

Another problem is that MGLRU does not adequately protect the page cache, Song said. Workloads with a lot of anonymous pages perform well, but those that are more dependent on the page cache can regress. It's not just a matter of balancing, he said, but properly identifying the working set; the PID controller is not doing that properly. The MGLRU-FG work aims to unify the metrics in this area and improve working-set detection; the result is significantly better performance. 

David Hildenbrand said that, when MGLRU was merged, the plan was to make it the default; he asked Song why people would choose not to use it now. Song answered that many distributions have switched, but there are users who are running into regressions in page-cache performance. Once that problem is fixed, he said, he sees no reason not to just use MGLRU. 

Hildenbrand then asked about page-flag usage; those bits are in short supply, and MGLRU uses a number of them. The MGLRU-FG series reduces that usage from four to three bits, Song said. Hildenbrand asked if that meant MGLRU could now be used on 32-bit systems (which have fewer page flags available); a unified reclaim implementation must work on ""all the architectures we don't care about"". Song said that the flag usage could be reduced to two, but with a slight performance impact. 

The pace increased toward the end of the session as Song mentioned some of the other things he has been working on. He is trying to unify the calculation of refault distance between the two implementations. A "traditional-LRU compatibility mode" can be implemented by reducing the number of generations and tiers to two. He is also working to increase the number of generations (normally set to four) while reducing the associated use of page flags; eventually it should be possible to support 64 generations. Matthew Wilcox asked how much better 64 would be; Song didn't really have an answer but said that the cost of running the experiment is low. 

An idea for the future would be to extend MGLRU using BPF; it would provide a hook to allow a BPF program to decide which generation a faulted folio should be placed into. Another hook could allow a program to move a folio to a different generation on access. That would allow the implementation of a number of different reclaim policies, under administrator control. As time ran out, one audience member suggested that this might be overkill. 

#### MGLRU on Android

[![\[Zicheng Wang\]](https://static.lwn.net/images/conf/2026/lsfmm/ZichengWang-sm.png)](<https://lwn.net/Articles/1072879/>) Zicheng Wang works for HONOR, a smartphone manufacture that has chosen to enable MGLRU on all the devices it ships — that is about 70 million devices running MGLRU overall. Android, he said, heavily overcommits memory and needs aggressive reclaim to perform well. But that reclaim operation must fit into an 8.3ms time budget; anything longer might cause rendering stalls on a 120-frame-per-second screen. Reclaim cannot be allowed to block critical paths. 

The Android app lifecycle starts with launch, he said, followed by transitions between the foreground, background, and frozen states. Android tends to preload a lot of pages in the early stages for quick launch, relying on aggressive reclaim to clean up the memory that is not needed. MGLRU, he said, reclaims too many file-backed pages, degrading performance in some scenarios; that can lead to a slower camera launch, for example. One solution to this problem is active aging; when an app goes into the background, its pages are targeted for aging. That helps to distribute anonymous pages across the generations, and yields significant improvements at the cost of a bit more CPU usage. 

MGLRU makes it hard to control how much reclaiming is done; it continues to reclaim pages after the target watermarks have been hit. Among other things, that breaks the time budget that reclaim must fit into. HONOR has addressed this problem by adding a hook to tell MGLRU when it is time to quit. 

Throttling of processes in direct reclaim can put tasks to sleep waiting for the kswapd kernel thread to free more memory; that is causing threads to stall for too long. The problem seems to be that kswapd is wasting a lot of time scanning control groups that will yield little reclaimable memory. He had no real solution for that problem. 

A foreground app's file-backed pages can be frequently reclaimed, causing increased latency as they must be faulted back in. He has addressed that by adding a hook to skip over the foreground app during reclaim. There is also a problem with the automatic activation of pages brought in by the readahead code; that can activate pages that will never be used, at the expense of the pages an app actually needs. 

To conclude, Wang said that he would like to see some sort of generic interfaces provided by MGLRU; that would be better than the current vendor-specific hacks. The parameters that control aging should be exposed, he said. There are some knobs currently in debugfs that, perhaps, should move to sysfs so they could be made available on production systems. And, he said, MGLRU needs better awareness of the priorities of the tasks running on the system. 

The session concluded without discussion. 

Wang has [posted the slides](<https://docs.google.com/presentation/d/1hUogz6InyLn13c8CjHuvEIzE4rT7saVRUV6xpWZoNfQ/edit?slide=id.g3ddb7916804_2_27#slide=id.g3ddb7916804_2_27>) from this session.
