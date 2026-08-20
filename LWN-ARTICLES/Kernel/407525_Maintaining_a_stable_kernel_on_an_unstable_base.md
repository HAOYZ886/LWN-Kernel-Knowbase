---
title: Maintaining a stable kernel on an unstable base
url: https://lwn.net/Articles/407525/
date: "September 29, 2010"
category: "Development model-Stable tree; Releases-Stable updates"
author: "By Jonathan Corbet September 29, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 29, 2010

Greg Kroah-Hartman launched his LinuxCon Japan 2010 keynote by stating that the most fun thing about working on Linux is that it is not stable; it is, in fact, the fastest-moving software project in the history of the world. This claim was justified with a number of statistics on development speed, all of which will be quite familiar to LWN readers. In summary, over the last year, the kernel has been absorbing 5.5 changes per hour, every hour, without a break. How, he asked, might one try to build a stable kernel on top of such a rapidly-changing base? 

The answer began with a history lesson. Fifteen years ago, the 2.0.0 kernel came out, and things were looking good. We had good performance, SMP support, a shiny new mascot, and more. After four months of stabilization work, the 2.1.0 tree was branched off, and development of the mainline resumed. This was, of course, the days of the traditional even/odd development cycle, which seemed like the right way to do things at the time. 

It took 848 days and 141 development releases to reach the 2.2.0 kernel. There was a strong feeling that things should go faster than that, so when, four months later, the 2.3.0 kernel came out, there was a hope that this development cycle would be a little bit shorter. To an extent, we succeeded: it only took 604 days and 58 releases to get to 2.4.0. But people who were watching at the time will remember that 2.4 took a long time to really stabilize; it was a full ten months before Linus felt ready to create the 2.5 branch and go into development mode again. 

This time around, the developers intended to do a short development cycle for real. There was a lot of new code which they wanted to get into the hands of users as soon as possible. In fact, the pressure to push features to users was so strong that the distributors were putting considerable resources into backporting 2.5 code into the 2.4 kernels they were shipping. The result was "a mess" at all levels: shipped 2.4 kernels were an unstable mixture of patches, and the developers ended up doing their feature work twice: once for 2.5, and once for the backport. It did not work very well. 

As a result, the 2.5 development cycle ran for 1057 days, with 86 releases. It was painful in a number of ways, but the end result - the 2.6 kernel - was significantly better than 2.4. Various things happened over the course of this development cycle; the development community learned a number of lessons about how kernel development should be done. The advent [![\[Greg Kroah-Hartman\]](https://static.lwn.net/images/conf/2010/LC-Tokyo/GregKH-sm.jpg)](<https://lwn.net/Articles/407533/>) of BitKeeper made distributed development work much better than it did in the past and highlighted the importance of breaking changes down into small, reviewable, debuggable pieces. The kernel community which existed at the 2.6.0 release was wiser and more experienced than what had existed before; we had figured out how to do things better. 

This evolution led to the adoption of the "new" development model in the early 2.6 days. The separate development and stable branches were gone, replaced with a single, fast-moving tree with releases about every three months. This system worked well for development; it is still in use several years later. But it made life a bit difficult for distributors and users. Even three months can be a long time to wait for important fixes, and, if those fixes come with a new load of bugs, they may not be entirely welcome. So it became clear that there needed to be a mechanism to distribute fixes (and only fixes) to users more quickly. 

The discussion led to Linus's [classic email](<http://article.gmane.org/gmane.linux.kernel/283396>) saying that it would not be possible to find somebody who could maintain a stable kernel over any period of time. But, still, he expressed some guidelines by which a suitable "sucker" could try to create such a tree. Within a few minutes, Greg had held up his hand as a potential sucker; Chris Wright followed thereafter. Greg has been doing it ever since; Chris created about 50 stable releases before eventually moving back to "real work" and away from stable kernel work. 

The stable tree has been in operation ever since. The model has changed little over that time; once a mainline release happens, it will receive stable updates for at least one development cycle. For most kernels, those updates stop after exactly one cycle. This is an important part of how the stable tree works; it puts an upper bound on the number of trees which must be maintained, and it encourages users to move forward to more current kernels. 

Greg presented the rules which apply to submissions to the stable tree: they must fix real bugs, be small and easily verified, etc. The most important rule, though, is the one stating that any patches must appear in the mainline before they can be applied to the stable tree. That rule ensures that important fixes get into both trees and increases assurance that the fixes have been properly reviewed. 

Some kernels receive longer stable support than others; one example is 2.6.32. A number of distribution kernel maintainers got together around 2.6.30 to see if they could all settle on a single kernel to maintain for a longer period; they settled on 2.6.32. That kernel has since been incorporated into SLES11 SP1, RHEL6, Debian Squeeze, Ubuntu 10.04 LTS, and Oracle's recently-announced enterprise kernel update. It has received over 2000 fixes to date, with contributions from everybody involved; 2.6.32 is a great example of inter-distribution contribution. It is also, as the result of all those fixes, a high-quality kernel at this point. 

Greg pointed out one other interesting thing about 2.6.32: two enterprise distributions (SLES and Oracle's offering) have moved forward to this kernel for an existing distribution. That is a bit of a change in an area where distributors have typically stuck with their original kernel versions over the lifetime of a release. There are significant costs to staying with an ancient kernel, so it would be encouraging if these distributors were to figure out how to move to newer stable kernels without creating problems for their users. 

The stable process is generally working well, with maintainers doing an increasingly good job of sending important fixes over. Some maintainers are quite good, with dedicated repository branches for stable patches. Others are...not quite so good; SCSI maintainer James Bottomley was told in a rather un-Japanese manner that he and his developers could be doing better. 

People who are interested in upcoming stable releases can participate in the review cycle as well. Two or three days before each release, Greg posts all of the candidate patches to the lists for review. Some people complain about the large number of posts, but he ignores them: the Linux community, he says, does its development in public. There are starting to be more people who are interested in helping with pre-release testing, a development which Greg described as "awesome." 

The talk concluded with a demo: Greg packaged up and released [2.6.35.7](<http://lwn.net/Articles/407520/>) (code name "Yokohama") from the stage. It seems that the 2.6.35.6 update - evidently released during Dirk Hohndel's MeeGo talk earlier in the week - contained a typo which made life difficult for Xen users. The fix, possibly the first major kernel release done in front of a crowd, hopefully will not suffer from the same kind of problem.
