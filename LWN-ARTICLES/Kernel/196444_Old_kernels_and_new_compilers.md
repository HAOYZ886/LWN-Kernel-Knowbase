---
title: Old kernels and new compilers
url: https://lwn.net/Articles/196444/
date: "August 21, 2006"
category: Development model
author: 
---

Under the long-lasting maintainership of Marcelo Tosatti, the 2.4 kernel went into a deep maintenance mode, with only important fixes being considered for merging. For some people, perhaps, it was a little too deep \- Marcelo clearly had other tasks besides 2.4 maintenance keeping him busy. Even so, few expected major changes when Willy Tarreau took over 2.4 maintenance after the 2.4.33 release. Why mess with 2.4 at this point? 

So Willy's [2.4.34-pre1 announcement](<http://lwn.net/Articles/195898/>) raised a few eyebrows. The prepatch itself contains a relatively small number of patches of the type one would expect. But the announcement itself notes that Willy is considering merging a set of patches to allow 2.4 kernels to be built with current gcc 4.x compilers. This is not a trivial set of changes; gcc 4.x is sufficiently different that a fairly wide-ranging set of fixes is required. The gcc 4.x transition for 2.6 was not an overnight affair. 

A clear question comes immediately to mind: why would somebody who is not interested in running a current kernel be bothering with contemporary compilers? One answer is to be found in the announcement itself: there are administrators who deploy 2.4 kernels on ultra-stable systems, but who build those kernels on their desktops. It is getting increasingly hard to find a current distribution with a compiler old enough to build 2.4 kernels, so these administrators are finding themselves in a bit of a bind. A 2.4 kernel which could be compiled with a current gcc would allow current systems to be used to build kernels for deployment on stable, production systems, many of which may not have their own compilers installed at all. 

Solar Designer has also [noted](<https://lwn.net/Articles/196445/>) that [Openwall GNU/*/Linux](<http://openwall.com/Owl/>) is planning to upgrade to gcc 4.x and would really rather not have to change to the 2.6 kernel at the same time. 

For an interesting read, see [Willy's description](<https://lwn.net/Articles/196447/>) of the user base, as he sees it, for the 2.4 kernel. In his view, the major users are those setting up very high-reliability sites. These people prefer 2.4 kernels for this job: 

Simply because we already know from collective experience that these versions can achieve very long uptimes (while we don't know this yet for a fresh new version which got 5700 patches in the last 3 months), and because with the addition of very few patches, you can make a bet on security: as long as newly discovered vulnerabilities don't affect you or are covered by your additional patches, you win. If you need to update and induce excessive downtime, you lose and pay penalties. 

The idea is to keep these people happy - by enabling the use of current compilers, among other things - until a 2.6 kernel comes along which is able to provide the same sort of stability guarantees. The 2.6 development model makes that sort of guarantee harder, however, because older 2.6.x kernels go out of general maintenance relatively quickly (though distributors can and do maintain them for longer). It is hard to find a 2.6 kernel with a multi-year track record of reliability, security, and ongoing fixes. 

Willy's hope is that the current 2.6.16 kernel, which Adrian Bunk has stepped forward to maintain for the long term, will help in this regard. Once 2.6.16 has received a year or two of fixes (and nothing else), it might reach a point where high-reliability people might trust it in deployed systems. Time will tell if this kernel is able to reach that point. 

As an aside, it's worth mentioning that a small number of developers (well, OK, one developer) have expressed some discontent about the 2.6.16 long-term process. This developer has [said](<https://lwn.net/Articles/196449/>) that it would have been better to elect an extra-stable tree maintainer through some sort of popular vote, and, perhaps, to move on to a 2.7 development series as well. This complaint ignores the fact that volunteers to maintain 2.6 kernels over the long term have been in relatively short supply; in fact, Adrian would appear to be about the only one. It does not appear that Adrian's appointment as the long-term 2.6.16 maintainer has deprived anybody else of their lifetime dreams. So maintainer elections - other than those of the "vote with your feet" variety - seem unlikely to happen in the near future.
