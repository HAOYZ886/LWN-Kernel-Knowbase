---
title: Testing for kernel performance regressions
url: https://lwn.net/Articles/509577/
date: "August 3, 2012"
category: "Development tools-MMTests; Performance regressions"
author: "By Jonathan Corbet August 3, 2012"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 3, 2012

It is not uncommon for software projects — free or otherwise — to include a set of tests intended to detect regressions before they create problems for users. The kernel lacks such a set of tests. There are some good reasons for this; most kernel problems tend to be associated with a specific device or controller and nobody has anything close to a complete set of relevant hardware. So the kernel depends heavily on early testers to find problems. The development process is also, in the form of the stable trees, designed to collect fixes for problems found after a release and to get them to users quickly. 

Still, there are places where more formalized regression testing could be helpful. Your editor has, over the years, heard a large number of presentations given by large "enterprise" users of Linux. Many of them expressed the same complaint: they upgrade to a new kernel (often skipping several intermediate versions) and find that the performance of their workloads drops considerably. Somewhere over the course of a year or so of kernel development, something got slower and nobody noticed. Finding performance regressions can be hard; they often only show up in workloads that do not exist except behind several layers of obsessive corporate firewalls. But the fact that there is relatively little testing for such regressions going on cannot help. 

Recently, Mel Gorman ran an extensive set of benchmarks on a set of machines and posted the results. He found some interesting things that tell us about the types of performance problems that future kernel users may encounter. 

His results include [a set of scheduler tests](<https://lwn.net/Articles/509585/>), consisting of the "starve," "hackbench," "pipetest," and "lmbench" benchmarks. On an Intel Core i7-based system, the results were generally quite good; he noted a regression in 3.0 that was subsequently fixed, and a regression in 3.4 that still exists, but, for the most part, the kernel has held up well (and even improved) for this particular set of benchmarks. At least, until one looks at the results for other processors. On a Pentium 4 system, various regressions came in late in the 2.6.x days, and things got a bit worse again through 3.3. On an AMD Phenom II system, numerous regressions have shown up in various 3.x kernels, with the result that performance as a whole is worse than it was back in 2.6.32. 

Mel has a hypothesis for why things may be happening this way: core kernel developers tend to have access to the newest, fanciest processors and are using those systems for their testing. So the code naturally ends up being optimized for those processors, at the expense of the older systems. Arguably that is exactly what should be happening; kernel developers are working on code to run on tomorrow's systems, so that's where their focus should be. But users may not get flashy new hardware quite so quickly; they would undoubtedly appreciate it if their existing systems did not get slower with newer kernels. 

He ran the [sysbench](<http://sysbench.sourceforge.net/>) tool on three different filesystems: [ext3](<https://lwn.net/Articles/509591/>), [ext4](<https://lwn.net/Articles/509592/>), and [xfs](<https://lwn.net/Articles/509593/>). All of them showed some regressions over time, with the 3.1 and 3.2 kernels showing especially bad swapping performance. Thereafter, things started to improve, with the developers' focus on fixing writeback problems almost certainly being a part of that solution. But ext3 is still showing a lot of regressions, while ext4 and xfs have gotten a lot better. The ext3 filesystem is supposed to be in maintenance mode, so it's not surprising that it isn't advancing much. But there are a lot of deployed ext3 systems out there; until their owners feel confident in switching to ext4, it would be good if ext3 performance did not get worse over time. 

[Another test](<https://lwn.net/Articles/509597/>) is designed to determine how well the kernel does at satisfying high-order allocation requests (being requests for multiple, physically-contiguous pages). The result here is that the kernel did OK and was steadily getting better—until the 3.4 release. Mel says: 

This correlates with the removal of lumpy reclaim which compaction indirectly depended upon. This strongly indicates that enough memory is not being reclaimed for compaction to make forward progress or compaction is being disabled routinely due to failed attempts at compaction. 

On the other hand, the test does well on idle systems, so the anti-fragmentation logic seems to be working as intended. 

Quite a few other test results have been posted as well; many of them show regressions creeping into the kernel in the last two years or so of development. In a sense, that is a discouraging result; nobody wants to see the performance of the system getting worse over time. On the other hand, identifying a problem is the first step toward fixing it; with specific metrics showing the regressions and when they first showed up, developers should be able to jump in and start fixing things. Then, perhaps, by the time those large users move to newer kernels, these particular problems will have been dealt with. 

That is an optimistic view, though, that is somewhat belied by the minimal response to most of Mel's results on the mailing lists. One gets the sense that most developers are not paying a lot of attention to these results, but perhaps that is a wrong impression. Possibly developers are far too busy tracking down the causes of the regressions to be chattering on the mailing lists. If so, the results should become apparent in future kernels. 

Developers can also run these tests themselves; Mel has released the whole set under the name [MMTests](<https://lwn.net/Articles/502747/>). If this test suite continues to advance, and if developers actually use it, the kernel should, with any luck at all, see fewer core performance regressions in the future. That should make users of all systems, large or small, happier.
