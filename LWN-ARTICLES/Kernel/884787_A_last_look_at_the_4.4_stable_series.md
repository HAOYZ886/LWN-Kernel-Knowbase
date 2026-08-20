---
title: A last look at the 4.4 stable series
url: https://lwn.net/Articles/884787/
date: "February 17, 2022"
category: "Development model-Stable tree; Releases-4.4"
author: "By Jonathan Corbet February 17, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 17, 2022

Linus Torvalds [released the 4.4 kernel](<https://lore.kernel.org/lkml/CA+55aFx93de2f2bRbCp74uNX0LxkgpSKE8uOGFrk63xe_9YDwg@mail.gmail.com/>) on January 10, 2016 and promptly left the building for the greener fields of 4.5. This kernel was finished from his point of view, but it was just beginning its life in the wider world, and became the first long-term-stable release to be supported for more than two years. Indeed, the 4.4 release became one of the longest-supported and most widely used releases in the history of the kernel project (so far); it was deployed in vast numbers of Android devices, among other places. The [final 4.4 stable release](<https://lwn.net/ml/linux-kernel/1643877137240249@kroah.com/>) took place on February 3, over six years after 4.4 was "finished"; it is time to take a look at what happened to 4.4 in its stable life. 

There were 302 stable updates released for 4.4 over the 2,216 days of its supported life — approximately one release per week for the entire six years. Those releases added 18,974 non-merge changesets to that "stable" kernel (about 8.6 patches per day, every day). By virtue of that work, the 4.4 kernel grew by nearly 90,000 lines of code; 72 new source files were added during that time. 

#### Contributing developers

The 4.4 stable series thus looks, in some ways, like another development cycle, but with even more patches. Some of the other numbers are larger as well; the fixes added to 4.4 were contributed by 3,528 developers over those six years, representing just under 500 different employers. Kernel developers may like adding new features but, in the end, almost all of them end up fixing bugs as well. The top bug fixers were: 

> Top bug fixes in 4.4.x  
> ---  
> Developer| Changesets| Pct  
> Greg Kroah-Hartman | 419| 2.2%  
> Eric Dumazet | 367| 1.9%  
> Takashi Iwai | 345| 1.8%  
> Arnd Bergmann | 336| 1.8%  
> Johan Hovold | 327| 1.7%  
> Dan Carpenter | 288| 1.5%  
> Thomas Gleixner | 168| 0.9%  
> Al Viro | 121| 0.6%  
> Eric Biggers | 118| 0.6%  
> Colin Ian King | 108| 0.6%  
> Linus Torvalds| 92| 0.5%  
> Geert Uytterhoeven | 87| 0.5%  
> Xin Long | 86| 0.5%  
> Jan Kara | 84| 0.4%  
> Nathan Chancellor | 84| 0.4%  
> Alan Stern | 82| 0.4%  
> Steven Rostedt (VMware) | 82| 0.4%  
> Hans de Goede | 81| 0.4%  
> Cong Wang | 80| 0.4%  
> Christophe JAILLET | 79| 0.4%  
  
Note that most of Greg Kroah-Hartman's "fixes" are actually the 302 tags applied to mark each release, leaving 117 actual fixes. The developers involved say a lot about where the bugs show up; Eric Dumazet works in networking and Takashi Iwai is the sound maintainer, for example. Arnd Bergmann, instead, puts a lot of effort into fixing problems all over the kernel tree, as does Dan Carpenter. Johan Hovald works all over the driver tree with a focus on the USB subsystem. It is worth noting that even the most productive contributors of fixes were individually responsible for less than 2% of the total. 

A bit of a different story comes out if one looks at testers and reviewers. 

> Test and review credits in 4.4.x  
> ---  
> | Tested-by  
> ---  
> Guenter Roeck | 75| 3.7%  
> Pavel Machek | 57| 2.8%  
> Jon Hunter | 55| 2.7%  
> Linux Kernel Functional Testing | 53| 2.6%  
> Shuah Khan | 50| 2.4%  
> Andrey Konovalov | 44| 2.1%  
> Aaron Brown | 44| 2.1%  
> Andrew Bowers | 38| 1.9%  
> Dmitry Vyukov | 35| 1.7%  
> Arnaldo Carvalho de Melo | 27| 1.3%  
> Borislav Petkov | 21| 1.0%  
> Jason Self | 17| 0.8%  
> Stan Johnson | 16| 0.8%  
> Joe Lawrence | 15| 0.7%  
> Marc Zyngier | 12| 0.6%  
> Jon Masters | 12| 0.6%  
> Krishneil Singh | 12| 0.6%  
> | Reviewed-by  
> ---  
> Greg Kroah-Hartman | 112| 2.1%  
> David Sterba | 109| 2.1%  
> Christoph Hellwig | 107| 2.0%  
> Alexey Makhalov | 100| 1.9%  
> Matt Helsley | 100| 1.9%  
> Bo Gan | 100| 1.9%  
> Hannes Reinecke | 80| 1.5%  
> Johannes Thumshirn | 72| 1.4%  
> Borislav Petkov | 70| 1.3%  
> Jan Kara | 70| 1.3%  
> Christian König | 59| 1.1%  
> Nick Desaulniers | 57| 1.1%  
> Andrew Morton| 54| 1.0%  
> Andy Shevchenko | 54| 1.0%  
> Ingo Molnar | 52| 1.0%  
> Juergen Gross | 50| 0.9%  
> Eric Dumazet | 48| 0.9%  
  
Only 1,527 patches applied to 4.4 contained Tested-by tags — that is just 8% of the total. That seems a bit surprising, given that each of those patches is meant to be a fix, so testing whether it works should be relatively straightforward. 

The first few lines of the Reviewed-by column are not particularly surprising; both Kroah-Hartman and Christoph Hellwig do a lot of reviews in general, and David Sterba is the Btrfs maintainer. The next few names, Alexey Makhalov, Matt Helsley, and Bo Gan, are a bit more surprising, though; Makhalov and Helsley only have one directly authored patch each in the kernel, while Gan has none. In all three cases, digging through the logs shows a flurry of activity in mid-2018; these three developers, as it turns out, played a big role in the backporting of the Meltdown and Spectre fixes to the 4.4 kernel. 

#### Patch review and selection

There were 4,101 patches applied to 4.4 that contained at least one Reviewed-by tag; that is 22% of the total. Compare that to the 37% of patches going into 5.16 with such tags. One might conclude that patches going into the stable kernels are getting less review than they should, but it is also likely that we are seeing the growth in the use of Reviewed-by tags over time. The fact that only 19% of the patches going into 4.5 had Reviewed-by tags would tend to back up that hypothesis. 

How are the bugs fixed in the stable updates found? One clue can be found in the `Reported-by` tags used to credit bug reporters; 3,529 commits in 4.4.x contained such tags, with the most active reporters being: 

> Top bug reporters in 4.4.x  
> ---  
> Reporter| Reports| Pct  
> Syzbot| 684| 17.1%  
> Dmitry Vyukov | 146| 3.7%  
> Hulk Robot | 145| 3.6%  
> kernel test robot | 88| 2.2%  
> Andrey Konovalov | 85| 2.1%  
> Dan Carpenter | 69| 1.7%  
> Ben Hutchings | 39| 1.0%  
> Jann Horn | 31| 0.8%  
> kbuild test robot | 30| 0.8%  
> Guenter Roeck | 23| 0.6%  
> Al Viro | 20| 0.5%  
> Wen Xu | 18| 0.5%  
> Linus Torvalds| 16| 0.4%  
> Jianlin Shi | 16| 0.4%  
> Geert Uytterhoeven | 14| 0.4%  
> Julien Grall | 14| 0.4%  
> Eric Dumazet | 13| 0.3%  
> Tetsuo Handa | 13| 0.3%  
> Eric Biggers | 13| 0.3%  
> Alexander Potapenko | 13| 0.3%  
  
Dmitry Vyukov is the developer behind Syzbot, so the first two lines could properly be combined, showing that one developer is responsible for nearly 21% of the credited bug reports leading to 4.4.x fixes — an impressive total. Other automated testing systems added another 6.6% of the reports. Some of the other top reporters are almost certainly using tools of their own to seek out bugs. Every one of these bugs is a problem that was fixed before it was encountered by users, which can only be a good thing. 

Also on the topic of tags: maintainers add `stable@vger.kernel.org` to a patch's CC list to mark a patch as being suitable for stable-kernel updates. Over the course of 4.4.x, 3446 patches (a bit over 18% of the total) were marked in this way. Obviously, the CC list is not the mechanism by which most patches get into the stable updates. In some subsystems ~~(networking in particular)~~ , the maintainers manage a separate queue of stable-bound patches and actively discourage developers from marking patches themselves. 

Most of the time, though, the patches are being picked by the stable-kernel maintainers themselves, either by hand or through the use of a machine-learning system. This process is [frequently controversial](<https://lwn.net/Articles/863505/>). The `Fixes:` tag is often thought to be the indicator used by the stable team to pick patches that have not been explicitly marked for stable updates. The 4.4.x series included 7,629 patches with such tags, which is still only 40% of the total, so that tag is not a strong indicator either. The stable team is selecting patches that look like fixes even in the lack of any explicit markings. 

#### Regressions and overall conclusion

One interesting use for `Fixes:` tags, though, is to see whether they refer to other patches that had been included in previous stable releases. In other words, is a patch with a `Fixes:` tag fixing a bug that was introduced in the stable series itself? Each such occurrence would indicate a regression shipped in a stable update, which is exactly what the stable kernels are meant to avoid. 

Over the course of the 4.4 stable series, [1,309 patches](<https://lwn.net/Articles/884852/>) contained `Fixes:` tags referring to commits appearing earlier in the stable series. Of those, though, 310 fixed patches that had been merged for the _same_ stable update; those bugs never made it through to a release, and thus should not be counted as regressions. Some patches required multiple fixes and should not be counted twice; filtering those out leaves 884 patches — 4.7% of the total — that added regressions to the stable updates. Many of these regressions were likely never noticed by users, of course, but a few of them were serious. 

Through all of this, the 4.4 kernels drove a huge number of devices. It was an allowed kernel version for the Android 7 through 9 releases, and is, without doubt, still running on a lot of devices despite the end of ongoing support. Many millions of people have benefited from the work that the stable-kernel maintainers (and all of the people who have helped them) have done. Assigning a dollar value to this work would be difficult, but the resulting number would have to be huge; it is a significant gift to the world as a whole. 

The free-software community has often shown that it is better at cranking out software than supporting that software over long periods of time; the latter task often falls to providers who expect to be paid for access to their work. The 4.4.x kernel has shown, though, that the kernel community is, when it sets its mind to the task, indeed able to provide support for a full six years. The stable team's approach to picking patches is controversial, and probably will always be regardless of how the policy evolves, but the results were clearly good enough to build an extended ecosystem around. It is hard to conclude that this effort was anything but a significant success.
