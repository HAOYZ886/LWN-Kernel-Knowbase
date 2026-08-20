---
title: Avoiding the OS abstraction trap
url: https://lwn.net/Articles/454716/
date: "August 12, 2011"
category: Development model
author: "August 12, 2011 This article was contributed by Dan J. Williams"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

August 12, 2011

This article was contributed by Dan J. Williams

It is an innocent idea. After all, "[all problems in computer science can be solved by another level of indirection](<http://www.dmst.aueb.gr/dds/pubs/inbook/beautiful_code/html/Spi07g.html>)." However, when the problem is developing a device driver for acceptance into the current mainline Linux kernel, OS abstraction (using a level of indirection to hide a kernel's internal API) is taking things a level too far. Seasoned Linux kernel developers will have already cringed at the premise of this article. But they are not my intended readers; instead, this text is aimed at those that find themselves in a similar position as the original authors of the isci driver: a team new to the process of getting a large driver accepted into the mainline, and tasked with enabling several environments at once. The isci driver fell into the OS abstraction trap. These are the lessons learned and attitudes your author developed about this trap while leading the effort to rework the driver for upstream acceptance. 

As mentioned above, one would be hard pressed to find an experienced Linux kernel developer willing to accept OS abstraction as a general approach to driver design. So, a simplistic rule of thumb for those wanting to avoid the pain of reworking large amounts of code would be to not go it alone. Arrange for a developer with at least 100 upstream commits to be permanently assigned to the development team, and seek the advice of a developer with at least 500 commits early in the design phase. After the fact, it was one such developer, Arjan van de Ven, who set the expectation for the magnitude of rework effort. When your author was toying with ideas of Coccinelle and other automated ways to unwind the driver's abstractions Arjan presciently noted (paraphrasing): "...it isn't about the specific abstractions, it's about the wider assumptions that lead to the abstractions." 

The fundamental problem with OS abstraction techniques is that they actively defeat the purpose of having an open driver in the first place. As a community we want drivers upstream so that we can refactor common code into generic infrastructure and drive uniformity across drivers of the same class. OS abstraction, in contrast, implies the development of driver-specific translations of common kernel constructs, "lowest common denominator" programming to avoid constructs that do not have a clear analogue in all environments, and overlooking the subtleties of interfaces that appear similar but have important semantic differences. 

So what were the larger problematic assumptions that led to to the rework effort? It comes down to the following list that, given the recurrence of OS-abstracted drivers, may be expected among other developers new to the Linux mainline acceptance process. 

  1. The programming interface of the the Linux kernel is a static contract to third party developers. The documentation is up to date, and the recourse for upper layer bugs is to add workarounds to the driver. 

  2. The "community" is external to the development team. Many conversations about Linux kernel requirements reference the "community" as an anonymous body of developers and norms external to the driver's development process. 

  3. An OS abstraction layer can cleanly handle the differences between operating systems. 

In the case of the isci driver these assumptions resulted in a nearly half-year effort to rework the driver as measured from the first public release until the driver was ultimately deemed acceptable. 

#### Who fixes the platform?

The kernel community runs on trust and reciprocation. To get things done in a timely manner one needs to build up a cache of trust capital. One quick way to build this capital is to fix bugs or otherwise lower the overall maintenance burden of the kernel. Fixing a bug in a core library, or spotting some duplicated patterns that can be unified are golden opportunities to demonstrate proficiency and build trust. 

The attitude of aiming to become a co-implementer of common kernel infrastructure is an inherently foreign idea for a developer with a proprietary environment background. The proprietary OS vendor provides an interface sandbox for the driver to play in that is assumed to be rigid, documented and supported. A similar assumption was carried to the isci driver; for example, it initially contained workarounds for bugs (real and perceived) in libsas and the other upper layers. The assumption behind those workarounds seems to be that the "vendor's" (maintainer's) interface is broken and the vendor is on the hook for a fix. This is, of course, the well-known "[platform problem](<https://lwn.net/Articles/443531/>)." 

In the particular case of libsas, SCSI maintainer James Bottomley noted: "there's no overall maintainer, it's jointly maintained by its users." Internal kernel interfaces evolve to meet the needs of their users, but the users that engender the most trust tend to have an easier time getting their needs met. In this case, root-causing the bugs or allowing time to clarify the documentation for libsas ahead of the driver's introduction might have streamlined the acceptance process; it certainly would have saved the effort of developing local workarounds to global problems. 

#### We the community

Similar to how internal kernel interfaces evolve at a different pace than their documentation, so too do the expectations of the community for mainline-acceptable code versus the documented submission requirements. However, in contrast to the interface question where the current code can be used to clarify interface details, the same cannot be done to determine the current set of requirements for mainline acceptance. The reality is that code exists in the mainline tree that would not be acceptable if it were being re-submitted for inclusion today. 

A driver with an OS-abstracted core can be found in the tree, but over time the maintenance burden incurred by that architecture has precluded future drivers from taking the same approach. As a result, attempting to understand "the community" from an external position is a sure-fire way to underestimate the current set of requirements for acceptable code. The only way to acquire this knowledge is ongoing participation. Read other drivers, read the code reviews from other developers, and try to answer the question "would someone external to the development team have a chance at maintaining the driver without assistance?". 

One clear "no" answer to this question from the isci driver experience came from the simple usage of c99 structure initializers. The common core was targeted for reuse in environments where there was no compiler support for this syntax. However, the state machine implementation in the driver had dozens of tables filled with, in some cases, hundreds of function pointers. Modifying such tables by counting commas and trusting comments is error prone. The conversion to c99-style struct initialization made the code safer to edit (compiler verifiable), more readable and, consequently, allowed many of those tables to be removed. These initializations were a simple example of lowest-common-denominator programming and a nuance that an "external" Linux kernel developer need not care to understand when modifying the driver, especially when the next round of cleanups are enabled by the change. 

#### Can the OS be hidden?

OS abstraction defenders may look at that last example and propose automated ways to convert the code for different environments. The dangerous assumptions of automated abstraction replacement engines are the same as listed above. The Linux kernel internal interface is not static, so the abstraction needs to keep up with the pace of API change, but that effort is better spent participating in the Linux interface evolution. 

More dangerous is the assumption that similar looking interfaces from different environments can be properly captured by a unified abstraction. The prominent example from the isci driver was memory mapping: converting between virtual and physical addresses. As far as your author knows, Linux is one of the few environments that utilizes IOMMUs (I/O memory management units) to protect standard streaming DMA mappings requested by device drivers (via the DMA API). The isci abstraction had a virtual-to-physical abstraction that mapped to `virt_to_phys()` (broken but straightforward to fix), but it also had a physical-to-virtual abstraction mapped to `phys_to_virt()` which was not straightforward to fix. The assumption that physical-to-virtual translation was a viable mechanism lead to an implementation that not only missed the DMA API requirements, but also the need to use `kmap()` when accessing pages that may be located in high memory. The lesson is that convenient interfaces in other environments can lead to the diversionary search for equivalent functionality in Linux and magnify the eventual rework effort. 

#### Conclusion

The initial patch added 60,896 lines to the kernel over 159 files. Once the rework was done, the number of new files was cut down to 34 and overall diffstat for the effort was: 

```
192 files changed, 23575 insertions(+), 60895 deletions(-)
```

There is no question that adherence to current Linux coding principles resulted in a simpler implementation of the isci driver. The community's mainline acceptance criteria are designed to maximize the portability and effectiveness of a kernel developer's skills across drivers. Any locally-developed convenience mechanisms that diminish that global portability will almost always result in requests for changes and prevent mainline acceptance. In the end participation and getting one's hands dirty in the evolution of the native interfaces is the requirement for mainline acceptance, and it is very difficult to achieve that through a level of indirection. 

I want to thank Christoph Hellwig and James Bottomley for their review and recognize Dave Jiang, Jeff Skirvin, Ed Nadolski, Jacek Danecki, Maciej Trela, Maciej Patelczyk and the rest of the isci development team that accomplished this rework.
