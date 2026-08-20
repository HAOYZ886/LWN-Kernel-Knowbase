---
title: "Who wrote 2.6.20?"
url: https://lwn.net/Articles/222773/
date: "February 21, 2007"
category: "Development model-Contributor statistics; Releases-2.6.20"
author: 
---

Time recently published an article entitled [Getting rich off those who work for free](<http://www.time.com/time/magazine/article/0,9171,1590440,00.html>) which, among other things, talked about free software this way: 

Open-source, volunteer-created computer software like the Linux operating system and the Firefox Web browser have also established themselves as significant and lasting economic realities. 

It is not uncommon to see Linux referred to as a volunteer-created system, as opposed to the corporate-sponsored, proprietary alternatives. There has been little research, however, into how much work on Linux is truly "volunteer" - done on a hacker's spare, unpaid time. In general, the assumption that Linux is created by volunteers is simply accepted. 

Determining the real provenance of free software can be a daunting task. There is a wealth of information available for those who look, however. In an attempt to shine some light in this area, your editor hacked up some scripts to do a lot of digging around in the kernel git repository. The idea was that, by looking at who is putting changes into the kernel, we can get a sense for where our source is coming from. 

### Who got patches into 2.6.20

This study looked at the stream of patches that changed the 2.6.19 kernel into the current 2.6.20 release. There were, as it turns out 4983 non-merge changesets in this release, contributed by 741 different developers. (Merge changesets mark where the contents of other repositories were pulled into the mainline, but they do not carry any code changes, so the analysis skipped them). These patches added 286,439 lines of code and removed 159,812 others, for a total growth of 126,627 lines over the 2.6.20 development cycle. 

Your editor's scripts looked over every non-merge commit in 2.6.20. For each, the developer listed as the "author" was given credit for the patch. This approach is not entirely fair, since one developer will, in some cases, be submitting code written by a group of people. In general, though, there is no easy way of getting around this problem - the true breakdown of authorship of a joint work simply is not available in the mainline repository. Your editor believes that this inaccuracy affects the accounting of a relatively small portion of the patches merged into the mainline. 

Beyond that, how one generates statistics from a patch stream is an interesting question. How does one measure the productivity of programmers? One possibility is to look at the number of changesets merged. By that metric, this is the list of the most prolific contributors to 2.6.20: 

> Developers with the most changesets  
> ---  
> Al Viro| 241| 4.8%  
> Andrew Morton| 92| 1.8%  
> Jiri Slaby| 92| 1.8%  
> Adrian Bunk| 87| 1.7%  
> Gerrit Renker| 79| 1.6%  
> Josef Sipek| 79| 1.6%  
> Avi Kivity| 68| 1.4%  
> Tejun Heo| 67| 1.3%  
> Patrick McHardy| 63| 1.3%  
> Ralf Baechle| 61| 1.2%  
> Randy Dunlap| 59| 1.2%  
> Alan Cox| 58| 1.2%  
> Mariusz Kozlowski| 57| 1.1%  
> Andrew Victor| 53| 1.1%  
> Paul Mundt| 52| 1.0%  
> Stefan Richter| 49| 1.0%  
> David S. Miller| 48| 1.0%  
> Russell King| 44| 0.9%  
> Benjamin Herrenschmidt| 44| 0.9%  
> Akinobu Mita| 43| 0.9%  
  
Looking at patch counts rewards developers who put in large numbers of small patches. Al Viro's patches include a vast number of code annotations (to enable better checking with `sparse`), include file fixups, etc. Many of the changes are small - many do not affect the resulting kernel executable at all - but there are a lot of them. Even so, as the biggest contributor, Al generated less than 5% of the total changesets added to the kernel. The top 20 contributors, all together, generated 28% of the total changesets in 2.6.20. 

One could make the argument that a better way to look at the problem is by the number of lines affected by a patch. In this way, a contributor's portion of the whole will not depend on whether it has been split into a long series of small patches or not. On the other hand, simply renaming a file can make it look like a developer has touched a large amount of code. Be that as it may, by looking at lines changed (defined as the greater of the number of lines added or removed by each individual changeset), one gets a table like this: 

> Developers with the most changed lines  
> ---  
> Jeff Garzik| 20712| 6.0%  
> Patrick McHardy| 15024| 4.3%  
> Jiri Slaby| 13917| 4.0%  
> Avi Kivity| 11726| 3.4%  
> Andrew Victor| 9710| 2.8%  
> Amit S. Kale| 9537| 2.7%  
> Stephen Hemminger| 9120| 2.6%  
> Geoff Levand| 8396| 2.4%  
> Michael Chan| 8307| 2.4%  
> Chris Zankel| 8099| 2.3%  
> Mauro Carvalho Chehab| 7390| 2.1%  
> Adrian Bunk| 6138| 1.8%  
> Yoshinori Sato| 5232| 1.5%  
> Al Viro| 4981| 1.4%  
> Benjamin Herrenschmidt| 4588| 1.3%  
> Thierry MERLE| 4549| 1.3%  
> Dan Williams| 4516| 1.3%  
> Jonathan Corbet| 3924| 1.1%  
> Gerrit Renker| 3857| 1.1%  
> Jiri Kosina| 3805| 1.1%  
  
Jeff Garzik comes out on top of this particular measurement by virtue of having deleted the long-unmaintained floppy tape subsystem. Patrick McHardy's work includes a number of additions to the netfilter subsystem, Jiri Slaby did a great deal of driver cleanup work, Avi Kivity was the contributor of the [KVM virtualization code](<http://lwn.net/Articles/216794/>), and Andrew Victor contributed a number of ARM-related patches and the Atmel AT91 i2c driver. (The contributions made by other authors can be found by searching out their name in the [2.6.20 short-form changelog](<http://lwn.net/Articles/220545/>)). 

Most of the developers in the above list got there by adding code to the kernel. It can be said, however, that the true heroes in the development community are those who remove code and make the kernel smaller. The developers who were best at removing more code than they added were: 

> Developers with the most lines removed  
> ---  
> Jeff Garzik| 19862| 12.4%  
> Chris Zankel| 5608| 3.5%  
> Adrian Bunk| 5528| 3.5%  
> Arnd Bergmann| 2224| 1.4%  
> Linus Torvalds| 1739| 1.1%  
> Atsushi Nemoto| 1425| 0.9%  
> Thierry MERLE| 911| 0.6%  
> David Gibson| 878| 0.5%  
> Dominik Brodowski| 528| 0.3%  
> Stefan Richter| 509| 0.3%  
  
Once again, Jeff Garzik's removal of ftape comes out on top, by far. Chris Zankel cleaned up the Xtensa architecture, removing a number of files in the process. Adrian Bunk worked on the ftape removal, got rid of the frame diverter code, removed an old, broken block driver, and generally performed cleanups all over the tree. Mr. Bunk is, in fact, the bane of old code; over the last year (since 2.6.16) he has removed a full 127,000 lines from the kernel source tree. Arnd Bergman got rid of a bunch of `syscall*()` macros. Linus Torvalds removed the broken x86 stack unwinder code. 

Finally, one could look at a different measure entirely: the number of patches signed off by each developer. A `Signed-off-by:` line is an indication that the person involved believes that the code is suitable for merging into the kernel; it implies that some degree of attention has been paid to the patch. Authors sign off their code, as do the subsystem maintainers who pass it up the chain. The top signers-off in 2.6.20 were: 

> Developers with the most signoffs  
> ---  
> Andrew Morton| 1422| 13.7%  
> Linus Torvalds| 1366| 13.2%  
> David S. Miller| 483| 4.7%  
> Jeff Garzik| 331| 3.2%  
> Greg Kroah-Hartman| 269| 2.6%  
> Al Viro| 241| 2.3%  
> Paul Mackerras| 232| 2.2%  
> Andi Kleen| 177| 1.7%  
> Mauro Carvalho Chehab| 170| 1.6%  
> Russell King| 166| 1.6%  
> Adrian Bunk| 120| 1.2%  
> Arnaldo Carvalho de Melo| 119| 1.1%  
> Ralf Baechle| 117| 1.1%  
> James Bottomley| 109| 1.1%  
> Patrick McHardy| 96| 0.9%  
> Jiri Slaby| 94| 0.9%  
> Avi Kivity| 87| 0.8%  
> Josef Sipek| 79| 0.8%  
> Paul Mundt| 78| 0.8%  
> Gerrit Renker| 78| 0.8%  
  
There were a total of 10,354 signoff lines in the 2.6.20 patch stream, so each changeset, on average, was signed off just over two times. It is interesting that Linus, who ultimately merges every patch, only signed off 13% of them. It seems that most patches, these days, go directly into the mainline from subsystem repositories without a signoff from Linus or Andrew. Most of the other names on that list, with just a few exceptions, are the maintainers of subsystem or architecture trees. 

### Who paid them

So now we have a sense for who got their fingers on the code which went into 2.6.20. But one interesting question still has not been answered: to what extent was that code contributed by volunteers (or "hobbyists")? Finding an answer to that question is somewhat trickier than looking at who wrote the patches, mostly because very few developers say "I wrote this on behalf of my employer." 

The approach taken by your editor was relatively simplistic, but, perhaps, the best that is practical. Any patch whose author's given email address indicates a corporate affiliation is assumed to have been developed by an employee of that corporation. So any patch posted by somebody with an `ibm.com` email address is accounted as having been done by an IBM employee. Things are complicated by the fact that many people who work for companies do not use corporate addresses; it is not unheard-of for companies to have policies explicitly prohibiting code contributions associated with their domains. Your editor has coped with this problem by filling in the relevant developer's affiliation whenever it is known to him; in some cases, the developer was asked for this information. 

This method has the effect of crediting _all_ of an employee's work to his or her employer. In many cases, the situation is probably more complicated than that; one assumes, for example, that a certain kernel hacker's employer has not directed him to [hack on Battle for Wesnoth](<http://www.flickr.com/photos/malcolmtredinnick/361460740/>). When looking only at kernel code, however, crediting all work to the employer is probably relatively safe. 

Using this approach, the top sources of changesets were: 

> Top changeset contributors by employer  
> ---  
> (Unknown)| 1244| 25.0%  
> Red Hat| 636| 12.8%  
> (None)| 383| 7.7%  
> IBM| 368| 7.4%  
> Novell| 295| 5.9%  
> Linux Foundation| 261| 5.2%  
> Intel| 178| 3.6%  
> Oracle| 126| 2.5%  
> Google| 97| 1.9%  
> University of Aberdeen| 79| 1.6%  
> HP| 78| 1.6%  
> Qumranet| 71| 1.4%  
> Nokia| 67| 1.3%  
> SGI| 64| 1.3%  
> Astaro| 63| 1.3%  
> MIPS Technologies| 61| 1.2%  
> SANPeople| 53| 1.1%  
> Miracle Linux| 43| 0.9%  
> MontaVista| 41| 0.8%  
> Broadcom| 39| 0.8%  
  
Looking instead at the number of lines of code changed, the results become: 

> Top lines changed by employer  
> ---  
> (Unknown)| 66154| 19.0%  
> Red Hat| 44527| 12.8%  
> (None)| 38099| 11.0%  
> IBM| 25244| 7.3%  
> Astaro| 15306| 4.4%  
> Linux Foundation| 13638| 3.9%  
> Qumranet| 12108| 3.5%  
> Novell| 11930| 3.4%  
> Intel| 11652| 3.4%  
> SANPeople| 9888| 2.8%  
> NetXen| 9607| 2.8%  
> Sony| 8497| 2.4%  
> Broadcom| 8349| 2.4%  
> Tensilica| 8195| 2.4%  
> Nokia| 5581| 1.6%  
> MontaVista| 4394| 1.3%  
> University of Aberdeen| 4324| 1.2%  
> LWN.net| 3975| 1.1%  
> Secretlab| 3370| 1.0%  
> HP| 3211| 0.9%  
  
[**Note** that these tables have been updated once since the article was originally published; the curious can see what [the original versions](<https://lwn.net/Articles/223673/>) looked like.] 

In these tables, the line marked "(Unknown)" is exactly that: patches for which existence of a supporting employer could not be determined. The line marked "(None)", instead, indicates the patches from developers known to be working on their own time. 

Either way, the results come out about the same: at least 65% of the code which went into 2.6.20 was created by people working for companies. If the entire "unknown" group turns out to be developers working on a volunteer basis - an unlikely result - then just over 1/3 of the 2.6.20 patch stream was written by volunteers. The real number will be lower, but it still shows that a significant portion of the code we run is written by developers who are donating their time. 

### One year's worth of changes

Looking at a single kernel release is instructive, but it can also be deceptive. The relatively short release cycle used by the kernel project makes it fairly easy for prolific developers to see few of their patches go into a specific release. In an attempt to gain a longer-term perspective, your editor forced his suffering system to crank through the entire history from 2.6.16 (released almost exactly one year ago) to the present. Some 28,000 non-merge changesets have been added to the mainline (by 1,961 developers) over that time, replacing 1.26 million lines of old code with 2.01 million lines of new code - the kernel grew by 754,000 lines. 

The developers who touched the most lines over that time were: 

> Developers with the most changed lines  
> ---  
> Adrian Bunk| 134021| 5.3%  
> Jeff Garzik| 87847| 3.5%  
> Andrew Vasquez| 75195| 3.0%  
> Mauro Carvalho Chehab| 68568| 2.7%  
> David Teigland| 46607| 1.9%  
> Ralf Baechle| 38559| 1.5%  
> David S. Miller| 35958| 1.4%  
> Andrew Victor| 35594| 1.4%  
> Bryan O'Sullivan| 33901| 1.4%  
> Paul Mundt| 27041| 1.1%  
> Dave Kleikamp| 26615| 1.1%  
> Lennert Buytenhek| 25192| 1.0%  
> Haavard Skinnemoen| 24372| 1.0%  
> Ben Dooks| 23207| 0.9%  
> Patrick McHardy| 23175| 0.9%  
> Ingo Molnar| 22456| 0.9%  
> James Bottomley| 22205| 0.9%  
> David Howells| 19168| 0.8%  
> Jiri Slaby| 18335| 0.7%  
> Divy Le Ray| 17909| 0.7%  
  
The results for employers were: 

> Top lines changed by employer  
> ---  
> (Unknown)| 740990| 29.5%  
> Red Hat| 361539| 14.4%  
> (None)| 239888| 9.6%  
> IBM| 200473| 8.0%  
> QLogic| 91834| 3.7%  
> Novell| 91594| 3.6%  
> Intel| 78041| 3.1%  
> MIPS Technologies| 58857| 2.3%  
> Nokia| 39676| 1.6%  
> SANPeople| 36038| 1.4%  
> SteelEye| 36021| 1.4%  
> Freescale| 35034| 1.4%  
> Linux Foundation| 34163| 1.4%  
> MontaVista| 30211| 1.2%  
> Simtec| 26166| 1.0%  
> Atmel| 25975| 1.0%  
> HP| 23714| 0.9%  
> SGI| 22057| 0.9%  
> Oracle| 21251| 0.8%  
> Open Grid Computing| 20505| 0.8%  
  
The end result of all this is that a number of the widely-expressed opinions about kernel development turn out to be true. There really are thousands of developers - at least, almost 2,000 who put in at least one patch over the course of the last year. Linus Torvalds is directly responsible for a very small portion of the code which makes it into the kernel. Contemporary kernel development is spread out among a broad group of people, most of whom are paid for the work they do. Overall, the picture is of a broad-based and well-supported development community. 

There are many other interesting things to be learned by looking at the kernel's development history. Expect more articles along these lines as your editor finds the time to improve his scripts.
