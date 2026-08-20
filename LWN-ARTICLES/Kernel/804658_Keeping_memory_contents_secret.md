---
title: Keeping memory contents secret
url: https://lwn.net/Articles/804658/
date: "November 15, 2019"
category: "Memory management-Address-space isolation"
author: "By Jonathan Corbet November 15, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 15, 2019

One of the many responsibilities of the operating system is to help processes keep secrets from each other. Operating systems often fail in this regard, sometimes due to factors — such as hardware bugs and user-space vulnerabilities — that are beyond their direct control. It is thus unsurprising that there is an increasing level of interest in ways to improve the ability to keep data secret, perhaps even from the operating system itself. The [`MAP_EXCLUSIVE`](<https://lwn.net/ml/linux-kernel/1572171452-7958-1-git-send-email-rppt@kernel.org/>) patch set from Mike Rapoport is one example of the work that is being done in this area; it also shows that the development community has not yet really begun to figure out how this type of feature should work. 

`MAP_EXCLUSIVE` is a new flag for the [`mmap()`](<http://man7.org/linux/man-pages/man2/mmap.2.html>) system call; its purpose is to request a region of memory that is mapped _only_ for the calling process and inaccessible to anybody else, including the kernel. It is a part of a larger [address-space isolation](<https://lwn.net/Articles/803823/>) effort underway in the memory-management subsystem, most of which is based on the idea that unmapped memory is much harder for an attacker to access. 

Mapping a memory range with `MAP_EXCLUSIVE` has a number of effects. It automatically implies the `MAP_LOCKED` and `MAP_POPULATE` flags, meaning that the memory in question will be immediately faulted into RAM and locked there — it should never find its way to a swap area, for example. The `MAP_PRIVATE` and `MAP_ANONYMOUS` flags are required, and `MAP_HUGETLB` is not allowed. Pages that are mapped this way will not be copied if the process forks. They are also removed from the kernel's direct mapping — the linear mapping of all of physical memory — making them inaccessible to the kernel in most circumstances. 

The goal behind `MAP_EXCLUSIVE` seems to have support within the community, but the actual implementation has raised a number of questions about how this functionality should work. One area of concern is the removal of the pages from the direct mapping. The kernel uses huge pages for that mapping, since that gives a significant performance improvement through decreased translation lookaside buffer (TLB) pressure. Carving specific pages out of that mapping requires splitting the huge pages into normal pages, slowing things down for every process in the system. The splitting of the direct mapping in another context caused a 2% performance regression at Facebook, [according to Alexei Starovoitov](<https://lwn.net/Articles/801871/>) in October; that is not a cost that everybody is willing to pay. 

Elena Reshetova [indicated](<https://lwn.net/ml/linux-kernel/2236FBA76BA1254E88B949DDB74E612BA4EEC0CE@IRSMSX102.ger.corp.intel.com/>) that she has been working on similar functionality; rather than enhancing `mmap()`, her patch provides a new `[madvise()](<http://man7.org/linux/man-pages/man2/madvise.2.html>)` flag and requires that the secret areas be a multiple of the page size. Her version will eventually wipe any secret areas before returning the memory to general use in case the calling process doesn't do that. 

Reshetova also raised the idea of mapping this memory uncached. The benefit of doing so would be to protect its contents from a whole range of speculative-execution attacks, known and unknown. On the other hand, the effect on application performance would be something between "painful" and "crippling", depending on how often the memory is accessed. Some users would likely welcome the extra protection; many others may well find that the performance penalty rules out this feature's use entirely. Andy Lutomirski [said](<https://lwn.net/ml/linux-kernel/CALCETrWN9kc+10tf7YoBp9ixqkO_KZ=b1E_cFBr_Uogxhu68PQ@mail.gmail.com/>) that uncached memory should only be provided if it is explicitly asked for, but Alan Cox [responded](<https://lwn.net/ml/linux-kernel/57f25261400464ea58b65bf39ca1b4f89eea2ce2.camel@linux.intel.com/>) that users generally do not know whether they want uncached memory or not. 

More to the point, Cox continued, there may be any of a number of things that the system might do to protect the contents of secret memory; those things will vary from one system to the next and users will not be in a position to know what any specific system should use. That makes it all the more important to nail down what the `MAP_EXCLUSIVE` flag really means: 

IMHO the question is what is the actual semantic here. What are you asking for ? Does it mean "at any cost", what does it guarantee (100% or statistically), what level of guarantee is acceptable, what level is -EOPNOTSUPP or similar ? 

James Bottomley [took this argument even further](<https://lwn.net/ml/linux-kernel/1572371012.4812.19.camel@linux.ibm.com/>), describing `MAP_EXCLUSIVE` as ""a usability problem"". Protecting secret data might, on some systems, involve hardware technologies like [TME and SEV](<https://lwn.net/Articles/776688/>), for example, but developers cannot know that in a general way. Somehow, Bottomley suggested, the kernel should make the best choice it can for how to protect secret memory; one such choice could be to make the memory uncached only on systems where the speculative-execution mitigations are not active. Lutomirski [worried](<https://lwn.net/ml/linux-kernel/CALCETrUhGqyEJtyBQWubh=5cUt2i-N-oZJqcBM5jBnGi1saxsQ@mail.gmail.com/>) that this approach would not work, though; there are too many variables and ways in which things could go wrong. 

There is only one truly clear conclusion from this discussion: a desire for memory with higher levels of secrecy exists, but the development community lacks a clear idea of how that secrecy should be implemented and how it should be presented to the user. That suggests that this feature will not be showing up in a mainline kernel anytime soon. Getting memory secrecy wrong risks saddling the community with the maintenance of a misdesigned interface and, possibly, giving application developers a false sense of security. It is better to go slow in the hope of getting things right.
