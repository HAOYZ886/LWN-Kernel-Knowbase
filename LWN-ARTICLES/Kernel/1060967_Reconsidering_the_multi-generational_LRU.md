---
title: "Reconsidering the multi-generational LRU"
url: https://lwn.net/Articles/1060967/
date: "March 5, 2026"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet March 5, 2026"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 5, 2026

The [multi-generational LRU](<https://lwn.net/Articles/894859/>) (MGLRU) is an alternative memory-management algorithm that was merged for the 6.1 kernel in late 2022. It brought a promise of much-improved performance and simplified code. Since then, though, progress on MGLRU has stalled, and it still is not enabled on many systems. As the [2026 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) (LSFMM+BPF) approaches, several memory-management developers have indicated a desire to talk about the future of MGLRU. While some developers are looking for ways to improve the subsystem, another has called for it to be removed entirely. 

#### An MGLRU refresher

One of the core memory-management tasks a kernel must handle is to determine which pages belong in RAM and which should be pushed out to slower storage (or "reclaimed"). As a general rule, it is best to retain the pages that will be used the most in the near future, while reclaiming pages that will not be used again. Given the challenges involved in predicting the future, the kernel must rely heavily on information about how pages were used recently as a guide for what will happen going forward. The least-recently-used (LRU) lists are a key component of that solution. 

The classic (and still default) solution in the kernel relies on two LRU lists (more correctly, numerous pairs of such lists) called the "active" and "inactive" lists. Pages that are thought to be in current use should be on the active list, while those that are seemingly unused go onto the inactive list. When the time comes to reclaim pages for other use, the inactive list will be consulted for a list of potential victims. Much of the complexity (and many of the heuristics) in this solution are focused on properly sizing the two lists and deciding when to move pages from one to the other. 

The MGLRU extends that approach to multiple lists, deemed "generations". At one end, the youngest generation contains pages that are known (or at least thought) to have been used within the recent past. Each older generation tracks pages that have been idle for longer than those in the preceding generations. Various sorts of accesses will move a page from an older generation to a younger one; the oldest generation is pillaged by the kernel when the need for more free memory arises. The MGLRU is claimed to more accurately identify the truly cold pages and to use less CPU time while doing that work. See [this 2021 article](<https://lwn.net/Articles/851184/>) for more information about the design of MGLRU. 

#### The trouble with MGLRU

Recent discussions have made it clear that MGLRU is not seen as living up to all of its promises. It all started in mid-February, when Zicheng Wang posted [a request for an LSFMM+BPF discussion](<https://lwn.net/ml/all/cb0c0a0bfc7247cf85858eecf0db6eca@honor.com>) about MGLRU and, specifically, how it works with Android. Even though MGLRU has been in the kernel for some years, Wang said, many vendors of Android systems do not enable it. There are a number of problems that play into that decision. 

One complaint (which was later echoed by others) is that MGLRU does not properly balance reclaim between anonymous and file-backed pages. The traditional LRU maintains a separate pair of lists for each of those page types (thus the comment above about "numerous pairs" of LRU lists — there are other complications as well). Reclaim from the two sets of lists is normally biased somewhat toward file-backed pages, since they do not normally need to be written back to persistent storage; the longstanding ["swappiness" sysctl knob](<https://docs.kernel.org/admin-guide/sysctl/vm.html#swappiness>) can be used to adjust how aggressively the kernel attacks each list. 

With MGLRU, Wang said, anonymous pages tend to stay within the youngest two generations, causing them to never be reclaimed (and file-backed pages to be reclaimed overly aggressively). Adjusting the swappiness knob does not fix the problem. Wang's employer (an Android OEM called "Honor") addresses this problem by explicitly using memory control groups to force reclaim of anonymous pages from non-foreground apps, but there is no general solution in the mainline kernel. Kairui Song, who [proposed an MGLRU session as well](<https://lwn.net/ml/all/CAMgjq7BoekNjg-Ra3C8M7=8=75su38w=HD782T5E_cxyeCeH_g@mail.gmail.com>), also mentioned problems with the reclaim of anonymous pages. 

Wang had a number of other problems to discuss. MGLRU can reclaim too aggressively from any given control group, freeing memory beyond the required amount. It's too expensive on low-end devices, especially in situations where there are not a lot of reclaimable pages. There is also a disconnect between Android's notion of hot and cold apps (designed to prioritize the app the user is interacting with at any given time) and MGLRU, which (like the rest of the kernel) lacks that distinction. Some of these problems have been addressed with vendor-specific hacks; there is, for example, [a vendor hook](<https://android-review.googlesource.com/c/kernel/common/+/3870920>) that exempts the current foreground task from reclaim. Wang would like to discuss which of these vendor changes, if any, should find their way into the mainline kernel. 

Barry Song added [a separate complaint](<https://lwn.net/ml/all/CAGsJ_4zatnuLkCJyDe_o_yXmhngpc54kV6b7M3tu6JyvF-ZxDw@mail.gmail.com>): when the system performs readahead (speculatively reading data that it thinks user space may soon request), it places all of the resulting pages into the youngest generation, even though there is no guarantee that those pages will ever be used at all. That may cause pages actually in use to be reclaimed while leaving the readahead pages in RAM. The traditional LRU, instead, puts those pages onto the inactive list, where they will be reclaimed relatively quickly if they are not referenced again. This problem, at least, should be amenable to a relatively simple solution. 

Kairui Song's list of problems had a different focus, starting with the fact that MGLRU uses three page flags. These flags are in [perennial short supply](<https://lwn.net/Articles/335768/>); patches that try to allocate even one of them tend to run into stiff resistance. The desire to increase the number of generations managed by MGLRU also implies using even more page flags. He had a proposal for shifting those flags elsewhere, making systems with up to 63 generations possible while freeing the three page flags currently used by MGLRU. 

Another problem is performance regressions for some workloads (while others do better). Kairui Song thinks that these problems result from the control loop that manages reclaim in MGLRU, and that they could be addressed by better tracking the usage history of file-backed pages. Doing that, though, would require three page flags, presumably those that had just been freed by shifting the generation number elsewhere. 

The metrics provided by MGLRU differ from those out of the traditional LRU in a number of ways, he said, making it harder for other parts of the system to understand the memory-management state of any given page. He has a proposal for changing how the state of pages is tracked to improve that situation. Kalesh Singh also [described problems with metrics](<https://lwn.net/ml/all/CAC_TJvfi_5n2crGHbRVO0R8jdYcSCmgzUJPzhtSvWeCN6fCbQg@mail.gmail.com>), saying that they differ significantly between the two LRU implementations, and that makes life difficult for components like the Android user-space out-of-memory daemon. 

In passing, Kairui Song also mentioned the idea of adding a BPF hook that would allow the customization of generation-placement decisions. 

There were other problems mentioned as well but, perhaps surprisingly, most of the participants skipped over one other relevant issue: the fact that there are two competing LRU implementations in the kernel in the first place. Kairui Song did note that the problems he described were among the ""many reasons MGLRU is still not the only LRU implementation in the kernel"". David Rientjes [added](<https://lwn.net/ml/all/87c70313-64ce-fb3f-4b0a-8db0526b6c54@google.com>) that the discussion should cover ""what needs to be addressed so that MGLRU can be on a path to becoming the default implementation and we can eliminate two separate implementations"". That will be a challenging thing to do; there will certainly always be workloads that do better with one implementation than the other, so removing one will cause some workloads to regress. Getting those regressions down to a tolerable level will require some work yet. 

#### Just remove it?

A persistent fear among kernel developers (and developers in many projects, in truth) is that a developer will add a pile of complex code, then not be around to maintain it. Matthew Wilcox [asserted](<https://lwn.net/ml/all/aaBsrrmV25FTIkVX@casper.infradead.org>) that this is exactly what has happened with MGLRU: 

> To my mind, the biggest problem with MGLRU is that Google dumped it on us and ran away. [Commit 44958000bada](<https://git.kernel.org/linus/44958000bada>) claimed that it was now maintained and added three people as maintainers. In the six months since that commit, none of those three people have any commits in mm/! This is a shameful state of affairs. 
> 
> I say rip it out. 

The original developer of MGLRU, Yu Zhao, has, as is noted in the above-mentioned commit, ""moved on to other projects these days"". As can be seen in the (subscriber-only) [KSDB page for Zhao](</ksdb/developers/show?dev=601>), he has occasionally made improvements to MGLRU, but the last such was a handful of commits in the 6.14 kernel. While other developers are said to be working on this code, none of those who were added to the `MAINTAINERS` file have made any changes to MGLRU since. 

So it is true, to an extent, that MGLRU was contributed to the kernel and abandoned shortly thereafter. Axel Rasmussen, one of the named maintainers of MGLRU, [seemed to agree](<https://lwn.net/ml/all/CAJHvVcioDfCxGRszaTOJST8ADszkB62N0WzzY_JQHp0t2bawsw@mail.gmail.com>) with this assessment, but said that the situation would soon change: 

> I acknowledge this is a big problem. We have let the community down here, and we plan to correct this starting in April, e.g. by working together with Kairui and others to address outstanding issues. 

The lack of ongoing developer attention certainly has not helped MGLRU to overcome the problems that many potential users have encountered with it. Even so, there was little support expressed for the idea of removing it. Barry Song [asked to keep it around](<https://lwn.net/ml/all/20260227043139.95115-1-21cnbao@gmail.com>) so that those problems could be addressed: 

> It just needs more work. MGLRU has many strong design aspects, including using more generations to differentiate cold from hot, the look-around mechanism to reduce scanning overhead by leveraging cache locality, and data structure designs that minimize lock holding. 

Gregory Price [listed a number of perceived problems](<https://lwn.net/ml/all/aakxPYGgXMHFQf1K@gourry-fedora-PF4VCD3F>) with MGLRU. He did, however, stop short of calling for to be taken out of the kernel entirely. 

So the MGLRU discussion at LSFMM+BPF in May seems unlikely to spend much time on the idea of removing it entirely. But there will be a lot of interest in understanding the work that needs to be done to bring MGLRU up to the needed level of performance and, perhaps someday, be the only LRU implementation in the kernel. If some developers are willing to commit to doing that work, MGLRU may finally make the progress that has been missing for the last few years. It seems likely to be an interesting session.
