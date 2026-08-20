---
title: A last look at the 4.19 stable series
url: https://lwn.net/Articles/1000933/
date: "December 12, 2024"
category: "Releases-4.19"
author: "By Jonathan Corbet December 12, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 12, 2024

The release of the [4.19.325 stable kernel update](<https://lwn.net/ml/all/2024120520-mashing-facing-6776@gregkh/>) on December 5 marked the end of an era of sorts. This kernel had been supported for just over six years since [its initial release](<https://lwn.net/ml/linux-kernel/20181022073224.GA5658@kroah.com/>) in October 2018; over that time, 325 updates were released, adding 30,109 fixes. Few Linux kernels receive public support for so long; it is worth taking a look at this kernel's history to see how it played out. 

The 4.19 release is unique in that it was made in [a time of uncertainty](<https://lwn.net/Articles/765108/>) in the Linux community; Linus Torvalds had taken a break from his role to rethink how he worked within the community. Meanwhile, developers were working to adopt and understand the kernel's new code of conduct. That uncertainly is nearly forgotten now, though; the pace of development never slowed, and the new normal (quite similar to the old normal) was quickly established. But 4.19 remains special as the only mainline release that was not made by Torvalds. 

#### Where the work came from

Over the six years of its support, 4.19.x incorporated fixes from 4,898 individual developers. By comparison, the mainline kernel has received over 487,000 commits from 13,578 developers during this time period. In other words, just over 6% of the commits going into the mainline kernel were backported to 4.19.x; just over a third of the developers contributing to the mainline saw their work backported into this stable series. The most active contributors to the 4.19.x stable series were: 

> Top bug-fix contributors to 4.19.x  
> ---  
> Developer| Changesets| Pct  
> Greg Kroah-Hartman | 457| 1.5%  
> Eric Dumazet | 445| 1.5%  
> Dan Carpenter | 373| 1.2%  
> Takashi Iwai | 293| 1.0%  
> Arnd Bergmann | 258| 0.9%  
> Johan Hovold | 253| 0.8%  
> Hans de Goede | 231| 0.8%  
> Nathan Chancellor | 191| 0.6%  
> Christophe JAILLET | 188| 0.6%  
> Yang Yingliang | 173| 0.6%  
> Thomas Gleixner | 168| 0.6%  
> Colin Ian King | 151| 0.5%  
> Jason A. Donenfeld | 150| 0.5%  
> Krzysztof Kozlowski | 147| 0.5%  
> Yue Haibing | 144| 0.5%  
> Jan Kara | 142| 0.5%  
> Pablo Neira Ayuso | 138| 0.5%  
> Randy Dunlap | 138| 0.5%  
> Geert Uytterhoeven | 133| 0.4%  
> Eric Biggers | 125| 0.4%  
  
It should be noted that, of Greg Kroah-Hartman's patches, 325 were simply setting the version number for each release; without those, he would still be on the above list, but toward the bottom. Over half (217) of the fixes contributed by Eric Dumazet were reported by the syzbot fuzz-testing system, which has proved to be a potent bug-finding tool. 

Indeed, syzbot is responsible for a large share (17%) of the 5,155 fixes that carried Reported-by tags: 

> Top bug reporters in 4.19.x  
> ---  
> Reporter| Reports| Pct  
> Syzbot| 1297| 22.2%  
> Hulk Robot | 270| 4.6%  
> kernel test robot | 253| 4.3%  
> Dan Carpenter | 109| 1.9%  
> Guenter Roeck | 36| 0.6%  
> TOTE Robot | 28| 0.5%  
> Lars-Peter Clausen | 27| 0.5%  
> Igor Zhbanov | 26| 0.4%  
> Jann Horn | 25| 0.4%  
> Nick Desaulniers | 24| 0.4%  
> Nathan Chancellor | 24| 0.4%  
> Geert Uytterhoeven | 24| 0.4%  
> Stephen Rothwell | 22| 0.4%  
> Jianlin Shi | 22| 0.4%  
> Abaci Robot | 21| 0.4%  
> Eric Dumazet | 19| 0.3%  
> Tetsuo Handa | 18| 0.3%  
> Qian Cai | 18| 0.3%  
> Naresh Kamboju | 18| 0.3%  
> Linus Torvalds| 17| 0.3%  
  
The fact that only 17% of the commits in this stable series have Reported-by tags (when all of the commits are meant to be bug fixes) suggests that many bug reporters may still be going uncredited. That percentage is higher than a typical mainline release, though, where about 6% of the commits have Reported-by tags. 

There were nearly 500 employers that supported the work going into the 4.19 stable series; the most active of those were: 

> Employers supporting 4.19.x fixes  
> ---  
> Company| Changesets| Pct  
> (Unknown)| 3317| 11.0%  
> Red Hat| 2412| 8.0%  
> (None)| 2368| 7.9%  
> Google| 2310| 7.7%  
> Huawei Technologies| 1870| 6.2%  
> Intel| 1858| 6.2%  
> SUSE| 1337| 4.4%  
> Linaro| 929| 3.1%  
> IBM| 854| 2.8%  
> Linux Foundation| 716| 2.4%  
> (Consultant)| 584| 1.9%  
> Oracle| 413| 1.4%  
> Arm| 388| 1.3%  
> AMD| 375| 1.2%  
> Meta| 345| 1.1%  
> Broadcom| 321| 1.1%  
> (Academia)| 312| 1.0%  
> Renesas Electronics| 304| 1.0%  
> NXP Semiconductors| 300| 1.0%  
> NVIDIA| 282| 0.9%  
  
The list of companies supporting this work differs a bit from the [companies that contributed to 4.19](<https://lwn.net/Articles/767635/>), or to [more recent kernels](<https://lwn.net/Articles/997959/>). The percentage of unknown and volunteer contributors is somewhat higher, and the companies with the highest contribution rates are, unsurprisingly, mostly in the business of supporting older kernels for relatively long periods of time. Red Hat may take some criticism for how it manages its enterprise kernels, but it is clearly doing a lot of work that helps the mainline stable kernels as well. 

#### Backporting

While writing the patches behind all of these fixes is a big part of the work going into a stable kernel update, there is more to it than that. Somebody must identify the patches as stable material and backport them — a task that can be as simple as a `git cherry-pick` command or as complex as completely rewriting the patch. Backporting a patch is essentially a new application of the patch, so it requires a new Signed-off-by line. Thus, by comparing the signoffs in a backported patch to those in the original, we can identify who did that work. 

Of the 30,109 patches applied to 4.19.x, 29,534 contained references to the originally applied mainline patch. Looking at those commits, 243 developers were identified as having performed stable backporting, but that work was not evenly spread out. The most active backporters were: 

> Top 4.19.x backporters  
> ---  
> Sasha Levin| 16,319| 55.25%  
> Greg Kroah-Hartman| 12,122| 41.04%  
> Ovidiu Panait| 79| 0.27%  
> Ben Hutchings| 61| 0.21%  
> Thadeu Lima de Souza Cascardo| 52| 0.18%  
> Jason A. Donenfeld| 48| 0.16%  
> Sudip Mukherjee| 46| 0.16%  
> Lee Jones| 45| 0.15%  
> Suleiman Souhlal| 31| 0.10%  
> Thomas Gleixner| 29| 0.10%  
> Nathan Chancellor| 25| 0.08%  
> Luis Chamberlain| 24| 0.08%  
> Mathieu Poirier| 22| 0.07%  
> Eric Biggers| 20| 0.07%  
  
While Sasha Levin and Kroah-Hartman clearly did the bulk of the backporting work — nearly 13 commits for every one of the 2,236 days of the 4.19.x series between them — it's worth noting that other developers usually only get involved in the backporting work when the backport itself is not trivial. So there may have been a fair amount of work involved to get those commits into a stable update. Still, it is hard not to think, when looking at those numbers, that the important task of creating these stable updates falls too heavily on too few developers. 

#### Where the bugs came from

Finally, we can also have a look at when the bugs fixed in 4.19.x were introduced. Just over 16,000 of the commits in 4.19.x contained Fixes tags indicating the commit that introduced a bug; with a bit of CPU time, those tags can be turned into a histogram of which release introduced each bug fixed in this stable series: 

> ![\[Histogram of bug-introducing
releases\]](https://static.lwn.net/images/2024/v4.19-pre.png)

As shown above, just over 1,000 of the bugs fixed in 4.19.x were introduced in 4.19 itself. After that, there is a long tail of bugs that covers every release in the Git history. There are 367 bugs attributed to 2.6.12, all of which predate the Git era entirely. It remains true that bugs can lurk in the kernel for a long time before somebody finds and fixes them. 

The above is not the full story, though; there are still about 2,000 fixes missing. They appear, instead, in this plot: 

> ![\[Histogram of bug-introducing
releases\]](https://static.lwn.net/images/2024/v4.19-post.png)

This plot shows all of the bugs that were fixed in 4.19.x that were introduced in commits added to the mainline _after_ the 4.19 release. Each of these, in other words, is a bug that was introduced into the stable series by way of a backported patch. While it appears that 4.20 was the biggest source of backported bugs and things have gotten better since, the real situation is almost certainly a bit less optimistic. As we have seen, bugs take time to show up and be fixed; if the maintenance of 4.19 were to continue, the plot would probably flatten out. The white space is likely just representing bugs that have not yet been fixed. As Kroah-Hartman noted in the 4.19.325 release announcement, there are currently 983 known bugs with CVE numbers that are unfixed in this kernel. 

It is worth noting that the above picture is still incomplete, since only a little over half (53%) of the patches going into 4.19.x carried Fixes tags. That is a lot of commits, in a series that is supposed to be dedicated to fixes, that lack that documentation. There will be a number of reasons for that, starting with the fact that, sometimes, developers simply do not take the time to track down the commit that introduced a bug they are fixing. But there is also a steady stream of changes for which a Fixes tag is not appropriate; these include new device IDs, documentation improvements, and a lot of fixes for hardware vulnerabilities. 

As Kroah-Hartman said in the 4.19.325 release: ""it had a good life, despite being born out of internal strife"". This kernel found its way into many distributions, and onto a massive number of Android devices. In a real sense, the stable kernels are the real end product from the kernel development community, and this one, despite the large number of updates, has stood up well. The time has come to say a final goodbye to this kernel — and to ensure that all remaining systems have been upgraded to something that is still supported.
